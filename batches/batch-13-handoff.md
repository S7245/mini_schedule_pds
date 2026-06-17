# Batch 13 交接 prompt —— 学员预约闭环（Entitlement / Booking / Waitlist / Attendance）

> 新 session 直接读本文件按此执行。第一步只做 grill 设计树 + 拆批建议，不写代码。

开始 Batch 13：学员预约闭环（Entitlement / Booking / Waitlist / Attendance）。这是一个大主题，第一步是 grill 设计树并决定如何拆分，不要直接写代码。

## 先读（按序）
1. `pds/CLAUDE.md` —— 每轮实现流程（§4 的 11 步 + 两个强制停止点）。
2. `pds/PROGRESS.md` §5 —— 重点 Batch 11（单场次排课）/ 12a（资源）/ 12b（循环排课）三段 + §3 缺口表。课程/资源/排课主线已全闭环，本批进入学员侧。
3. `pds/COURSE_BOOKING_BUSINESS_BLUEPRINT.md` —— §5.7 学员权益、§5.8 员工代预约、§11 预约规则、以及 attendance/签到/权益消耗相关章节（blueprint 是 source of truth，优先级最高）。
4. 三库 `.learnings`（backend / web / pds）的 **Batch 11 + 12a + 12b** 段必读 —— 直接相关的可复用模式与踩坑：①新域照搬 location/classsession 流水线（domain→service(require+checker==nil+data_scope scopeFilterIDs/guardLocationInScope)→persistence(tx+audit)→handler→wire）②DB 约束→业务错误别裸 500（23P01 EXCLUDE / 23505 unique 按约束名分流）③事务内「先查后插」有 race，靠 DB 约束 + SELECT FOR UPDATE / SAVEPOINT 兜并发 ④新 api client 必须在 packages/api/package.json exports 登记 ⑤前端聚合 type 的内嵌子对象后端 DTO 必须真填、契约 API 表必须列全前端依赖的读端点（grep ASSUMPTION）。

## Schema 现实（关键：表已建，无 CRUD/handler/UI）
全部学员预约相关表已在 `backend/migrations/000003_course_booking_schema.up.sql` 建好，本批大概率**不动表结构、只补权限 seed**（镜像 12a 000008）。逐个核对列/约束/状态机后再设计：
- `entitlement_products`（product_type: class_pack/trial_pack/membership_card；total_credits/validity_days；daily/weekly/monthly/concurrent_booking_limit；location_scope/course_scope all|specific + `entitlement_product_locations`/`_courses` 关联）。
- `learner_entitlements`（绑 brand_learner_profile + product；status active/expired/depleted/frozen/cancelled；total/remaining/locked/consumed_credits；starts_at/expires_at；granted_by）—— **锁额(lock)→消耗(consume) 模型**。
- `bookings`（class_session_id + brand_learner_profile_id；source learner_self_service/staff_assisted/waitlist_promotion；status booked/cancelled/attended/pending_no_show/no_show；cancel_source；unique(session,learner)）。
- `waitlist_entries`（position 正整数 + unique(session,position) active；status waiting/eligible_to_promote/promoted/cancelled/skipped；promoted_booking_id）。
- `entitlement_holds`（booking↔entitlement，credits，status held/released/consumed，unique(booking)）。
- `attendance_records`（unique(booking)，marked_by）+ `entitlement_consumptions`（type attendance/no_show/manual）+ `entitlement_transactions`（流水）。
- `brand_booking_policies`（book_ahead_min/max、cancel_deadline_minutes、release_on_cancel、no_show_consumes_entitlement、daily/weekly/concurrent_booking_limit；可门店级）。
- `class_sessions.booked_count`/`waitlist_limit`/`capacity` 已存在（B11）；`class_session_policy_overrides` 表存在。
- 学员主数据：`learner_identities` / `brand_learner_profiles` / `learner_tags` / `learner_staff_assignments` 表已建——**先确认这些有没有 CRUD**（brand 端「学员管理」`/users` 页是 legacy app_users，与 brand_learner_profiles 不是一回事，grill 要厘清学员档案管理是否本批前置/已存在）。

## grill 设计树重点（写契约前必须 grill 透，发现不可行就停下问用户）
- **范围拆分**（最重要）：这是 Batch 12 量级×N，几乎必拆多子批。给出推荐拆法，例如 13a=Entitlement 产品+发放(learner_entitlements 锁额模型)、13b=Booking 下单(容量+权益 hold 原子性)、13c=Waitlist 候补+晋升、13d=Attendance 签到+consume/no-show。或按 grill 后的更优切法。依赖方向要单向。
- **先做哪端**：brand 后台「员工代预约 staff_assisted」（复用现有 brand 模式、无需铺 api-app）vs C 端 api-app「学员自助 learner_self_service」。给推荐。
- **并发原子性**（核心难点）：下单要在一个事务里同时 ①锁 class_session 行(SELECT FOR UPDATE)校验 booked_count<capacity 并自增 ②锁/扣 learner_entitlement 余额建 entitlement_hold ③插 booking(unique 防重)。抢最后一个名额 / 最后一个课时的 race 怎么兜（行锁 + unique + CHECK）。
- **权益选择**：学员有多张可用 entitlement 时选哪张（按到期 FIFO？scope 匹配 location/course？）。lock→consume 时机：下单 hold，签到 consume，取消/缺勤按 policy release 或 consume。
- **预约规则**：book_ahead 窗口、cancel_deadline、release_on_cancel→恢复名额+课时+晋升候补、no_show_consumes_entitlement。entitlement_products 的 daily/weekly/monthly/concurrent limit + policy limit 叠加。
- **跨批联动**（务必识别）：B11 的「场次取消」目前只置 class_sessions.status，本批后必须级联——取消场次要 cancel 所有 booking（cancel_source=session_cancelled）+ release holds + 候补处理 + 通知。这是对已上线代码的回改点。
- **额度门**：blueprint §4.1——Learner 数量受订阅额度硬限制；建 brand_learner_profile 走 SubscriptionGuard（参考 location Create 的 CheckAndCount）。entitlement/booking 本身不限量。
- **data_scope**：booking/签到按 class_session.location_id 接 assigned_locations（镜像 classsession.Service）。
- **权限**：新增 entitlement.* / booking.* / attendance.* 等细粒度码 + role_template 映射 + 存量 backfill（镜像 12a migration 000008）。

## 流程铁律（本项目惯例，严格遵守）
- 按 pds/CLAUDE.md：grill 设计树 → 写契约 `pds/batches/batch-13x-*.md` → **会话内 approve**（邮件停止点被 classifier 拦，本项目改会话内确认，不发邮件）→ 写测试场景 → **主线程逐 task TDD commit**（用户不要 spawn 实现 subagent；先红→实现→绿→单 task commit）→ go build/test + `pnpm --filter @mini-schedule/brand lint+build` → `/code-review`（high，修或转 FR）→ **业务验收停止点**（e2e 由用户另开 session 跑，你给自包含 prompt）→ 更新 PROGRESS + 三库 .learnings → 三仓库分别 push → **backend/web 的 dev FF 合并进 main**（`git push origin dev:main`；pds 直接在 main）。
- 仓库：backend/web 在 `dev` 分支开发，pds 在 `main`。
- 验收铁律：后端 e2e 前必 `go build -o /tmp/api-brand ./cmd/api-brand` 重建重启（端口 **8081**，旧二进制掩盖改动）；前端 `rm -rf .next` 重启（:3002）+ curl 预热；filter 用包名 `@mini-schedule/brand`（`--filter=brand` 报错）；除 e2e 外必跑 prod `pnpm build`。
- DB 单测用真实 Postgres（`newMigratedTestDB` 自动建库跑 migration，无 PG 时 skip）。

## 测试账号（brand 21）
- owner `18816820405 / admin123`（显示名「李四」，user_id=16）；只读 `13900139777 / admin123`。
- 门店「讯美广场」id=1（active）；教练「张三」instructor_profile id=1；已有已发布课程模板；资源「1号教室」可临时建。
- ⚠️ 需要学员档案(brand_learner_profiles)——grill 时确认是否已有、没有则本批前置建几个测试学员。

第一步只做 grill：盘点 schema 与已有域、画出预约闭环的状态机与并发点、给出拆批建议 + 先做哪端 + 上述决策点的推荐项，然后用 AskUserQuestion 让用户拍板分叉项。拍板后再写第一个子批契约。
