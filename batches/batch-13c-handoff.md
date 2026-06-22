# Batch 13c 交接 prompt —— 学员预约下单（Booking + EntitlementHold + 取消 + 场次取消级联）

> 新 session 直接读本文件按此执行。**第一步只做 grill 设计树 + 拆批/范围决策，不写代码。**
> 这是 Batch 13（学员预约闭环）第三子批，也是**全闭环最硬的一批——下单原子性**。前置 13a 学员 ✅ / 13b 权益（lock 模型前半）✅ 均已验收并 merge main。

## 先读（按序）
1. `pds/CLAUDE.md` —— 每轮实现流程。**注意**：CLAUDE.md 写的「邮件停止点」被 classifier 拦，本项目已改**会话内 approve / 验收，不发邮件**（13a/13b 实测惯例）。
2. `pds/PROGRESS.md` §5 的 **Batch 13a + 13b** 段（学员档案、权益产品/发放已闭环，下一步学员侧下单）+ §3 缺口表。
3. `pds/COURSE_BOOKING_BUSINESS_BLUEPRINT.md`（source of truth，优先级最高）：**§9.2 Booking 状态机、§11 预约规则、§12 候补（仅了解边界，13c 不实现）、§22.1–22.3 流程、§24.2 并发一致性、§20.11 预约管理、§5.7 权益自动选择、§5.8 员工代预约、§21.2 Location 级角色（预约=代预约/代取消）**。
4. 三库 `.learnings`（backend / web / pds）的 **Batch 13a + 13b** 段必读 —— 直接相关的可复用模式与踩坑：①新域照搬 location/learner/entitlement 流水线（domain→service(require+checker==nil bypass+data_scope)→persistence(tx+audit)→handler→wire）②DB 约束→业务错误别裸 500（23505 unique / 23P01 EXCLUDE 按约束名分流，读 `pgErr.ConstraintName`）③`SELECT FOR UPDATE`+非负校验做锁额（13b adjust 骨架，13c hold 直接复用）④`entitlement.SettleStatus` 纯函数（13c hold/release 改余额后要 re-settle）⑤**聚合 list DTO 必须填齐 detail 同款内嵌字段**（13b F1：list 行常被复用为编辑初值）⑥前端新 api client 登记 `packages/api/package.json` exports；mutation 改 A 实体带动 B 的派生计数时两者都失效。
5. 核对 `backend/migrations/000003_course_booking_schema.up.sql` 里这些表的**真实列/约束/unique**：`bookings`（source/status/cancel_source/requires_entitlement_fix；**unique(class_session_id, brand_learner_profile_id) 全量**）、`entitlement_holds`（status held/released/consumed；unique(booking)）、`brand_booking_policies`（book_ahead_min/max、cancel_deadline_minutes、release_on_cancel、no_show_consumes_entitlement、daily/weekly/concurrent_limit、allow_waitlist、waitlist_limit；default(location_id NULL) + location 级 两条 partial unique）、`class_session_policy_overrides`（unique(session)）、`class_sessions`（booked_count/capacity/waitlist_limit，CHECK booked_count<=capacity）、`learner_entitlements`（remaining/locked/consumed_credits）、`entitlement_transactions`（action hold/release/consume…）。

## Schema 现实（表已建，bookings/holds/policies 零代码）
全部表在 000003 已建。13a/13b 的 `learner` / `entitlement` 域已就绪可绑。本批新增 booking 域 + 大概率只补 1 个 migration（partial unique + 权限映射），不动其余表结构。逐个核对列/约束/状态机后再设计。

## 核心难点：下单原子性（§24.2，grill 必须透）
单事务内：
1. `SELECT FOR UPDATE` 锁 `class_sessions` 行 → 校 `status='scheduled'` & `booked_count<capacity` → `booked_count++`。
2. **权益选择**（§5.7）+ `SELECT FOR UPDATE` 锁 `learner_entitlements` 行 → 校 active/未过期/`remaining>0`/scope 匹配/频次 → 锁额（`remaining--`、`locked++`，re-settle 状态）→ 建 `entitlement_holds`(unique(booking))。
3. INSERT `bookings`（source、unique 防重）。
4. `entitlement_transactions` action=`hold`（delta=-1，balance_after）。
5. staff_assisted 无权益占位 → `requires_entitlement_fix=true` + 原因 + OperationLog（不建 hold，不绕容量）。

抢最后名额 / 最后课时的 race 靠**行锁 + unique + CHECK 兜底**（同 11/12 的 EXCLUDE、13b 的 SELECT FOR UPDATE 经验，不做应用层「先查后插」）。复用 13b 的 `entitlement.SettleStatus` + 锁额骨架。

## grill 设计树重点（写契约前 grill 透，发现不可行就停下问用户）
- **范围**：13c = brand 端 **staff_assisted 下单 + 代取消 + 场次取消级联** + 预约规则解析。**候补(13d)、签到/consume/no-show(13e) 不做**，但 hold 的 release/consume 时机要和 13e 对齐设计（13c 取消默认 release，no_show 留 13e）。
- **权益自动选择**（§5.7）：多张可用权益选哪张——scope 匹配 location/course → 最早过期 FIFO → 课包先于会员卡 → 体验包仅体验态。staff_assisted 弹窗**是否允许员工手动指定具体权益**（13b grill 时倾向允许，13c 拍板）。
- **预约规则解析**（§11）：effective policy = `brand_booking_policies`(default location_id=NULL) 叠 location 级覆盖 叠 `class_session_policy_overrides`。字段 book_ahead_min/max、cancel_deadline、release_on_cancel、no_show_consumes、limits、allow_waitlist、waitlist_limit。频次上限 = entitlement_products 的 daily/weekly/monthly/concurrent + policy 的 limits **叠加取最严**（concurrent=该学员 active「booked」未完成预约数）。
- **跨批回改（必做）**：B11/12b 的 `ClassSession.Cancel` 现在只置 `class_sessions.status=cancelled`。13c 后必须**级联**：cancel 所有 active booking(cancel_source=`session_cancelled`) + release holds(退 locked→remaining、re-settle) + 流水 + (候补失效留 13d)。这是对**已 merge-main** 的 classsession/recurringschedule 代码的回改点。
- **partial unique migration**（用户已在 Batch 13 grill 拍板）：`bookings` + `waitlist_entries` 的 `unique(session,learner)` 改 **partial（仅 active 状态，如 booked/pending_no_show）**，让取消后重约 INSERT 新行（镜像 13a 对 learner brand_identity 的 partial 修法）。13c migration = **000012**。
- **data_scope**：booking 按 `class_session.location_id` ∈ assigned_locations（镜像 `classsession.Service` 的 scopeFilterIDs/guardLocationInScope）。⚠️ 与 13b entitlement（品牌级无 scope）不同——booking 是 location 级，店长/前台要能代预约本门店（§21.2）。
- **权限**：000003 已 seed `booking.view / booking.create_assisted / booking.cancel`（+ attendance.* 留 13e）。**核对 §21 映射是否齐**（§21.2 店长/前台 预约=查看/代预约/代取消 → 要映射给 location_manager/receptionist；§21.1 owner/admin 全、course_operator/finance 查看）。缺则补映射 + backfill（镜像 13b 000011，复用粗码优先，别无脑拆细码）。若做 brand_booking_policies CRUD 需要其权限码（000003 可能没有 booking_policy.*，需新增或复用）。
- **失败模式**：满员（引导候补——13c 先放占位入口还是完全不碰 waitlist？）、无权益（self-service 拒、staff 可占位）、超预约窗口、频次超限、重复预约（unique）、并发抢名额/扣课时。

## 拆批/范围决策点（grill 后用 AskUserQuestion 拍板）
1. **brand_booking_policies CRUD 是否进 13c**：做品牌配预约规则的 API+UI，还是先用 sensible 默认 + 仅读 + class_session_policy_overrides，policies CRUD 延后？（影响 13c 体量）
2. **权益选择**：自动选择（系统按 §5.7 优先级）+ staff_assisted 弹窗是否给「手动指定权益」选项。
3. **满员/候补占位**：13c 满员时放不放「加入候补」入口（候补逻辑 13d），还是 13c 完全不碰 waitlist。
4. **取消 hold 处理**：13c 代取消默认 release（释放名额+退课时），no_show consume 留 13e；确认这个边界。
5. （若需要）booking_policy 权限码方案。

## 流程铁律（本项目惯例，严格遵守）
grill 设计树 → 写契约 `pds/batches/batch-13c-*.md` → **会话内 approve**（不发邮件）→ 写测试场景 `batch-13c-*-tests.md` → **主线程逐 task TDD commit**（用户不要 spawn 实现 subagent；先红→实现→绿→单 task commit，按文件/域为粒度）→ `go build ./... && go test ./...` + `pnpm --filter @mini-schedule/brand lint+build` → `/code-review`（2 个并行 review subagent：后端 correctness / 前端 correctness，修或转 FR）→ **业务验收停止点**（e2e 由用户另开 session 跑，你给自包含 prompt）→ 更新 PROGRESS + 三库 `.learnings` → 三仓库分别 push → **backend/web 的 dev FF 合并进 main**（`git push origin dev:main`；pds 直接 main）。
- 仓库：backend/web 在 `dev` 分支开发，pds 在 `main`。
- 验收铁律：后端 e2e 前必 `go build -o /tmp/api-brand ./cmd/api-brand` 重建重启（端口 **8081**，旧二进制掩盖改动）；前端 `rm -rf .next` 重启（:3002）+ curl 预热；filter 用包名 `@mini-schedule/brand`（`--filter=brand` 报错）；除 e2e 外必跑 prod `pnpm build`。
- DB 单测用真实 Postgres（`newMigratedTestDB` 自动建库跑 migration，无 PG 时 skip）。**dev DB 现在 v11**（13a=000009/000010，13b=000011）；13c migration 从 **000012** 起。
- 派生态/落库类（如 hold 改 remaining、级联取消）验收**须 psql 实查 DB 真值**，不能只看前端（13b settle 落库经验）。

## 测试账号（brand 21，⚠️ 13b 验收后有变更）
- owner `18816820405 / admin123`（显示名「李四」，user_id=16，fast-path 全权限）。
- ⚠️ **只读账号 `13900139777` 现已重指派为 `course_operator`**（13b 验收期改的，only-view 系角色）——13c 权限门测试按它现在的角色（course_operator：预约=查看，代预约/代取消应 403）。
- 门店「讯美广场」id=1（active）；⚠️ brand21 多数历史门店/课程是**软删**（deleted_at 非空），可选只剩 讯美广场(loc1) + 晨间瑜伽(course4 一带)。
- **13c 前置数据**：需 ① scheduled 状态的可预约场次（用 /schedule 排课，B11/12 已有）② 测试学员（/learners，13a）③ 给学员发的 active 权益（/entitlement-products + 学员「权益」Tab，13b）。grill 时确认这条数据链能否在测试库凑齐。

## 复用模式（13a/13b 已验证，直接照搬）
- 新域照搬流水线：domain（实体+Status+IsValid*+Repository）→ application/service（`require(code)` + `checker==nil` bypass + data_scope `scopeFilterIDs`/`guardLocationInScope`）→ persistence（事务 + `audit.Write` + DB 约束按约束名分流）→ interfaces/brand handler → wire `go generate` 重生（NewHandler 加参数，router_test 补 nil）。
- `SELECT FOR UPDATE`（`clause.Locking{Strength:"UPDATE"}`）+ 非负校验 → 业务错误码（不裸触 CHECK 23514→500）。
- 错误码加 `pkg/errors/error.go`；migration seed 三段式（permissions INSERT?复用优先 + role_template_permissions 映射 + 存量 brand backfill，镜像 000011）。
- 前端：types + api client（登记 exports）+ errors 常量 + PERMISSIONS 常量；页面/弹窗照搬 resources/learners/entitlements；mutation 跨查询失效；编辑弹窗外键选项并入「当前已选但已停用」项；DataTable + 状态筛选 + 权限门 disabled+Hint。

第一步只做 grill：盘 schema 与已有域、画 Booking 状态机 + 下单/取消/场次取消级联的 tx 边界与并发点、给拆批/范围建议 + 上述决策点推荐，然后用 AskUserQuestion 让用户拍板。拍板后再写 13c 契约。
