# Batch 13e 交接 prompt —— 签到 + 履约 + 爽约（Attendance / Consumption / NoShow）

> 新 session 直接读本文件按此执行。**第一步只做 grill 设计树 + 拆批/范围决策，不写代码。**
> 这是 Batch 13（学员预约闭环）**第五也是最后一子批**——把 hold「锁额」收口为 consume/release。
> 前置 13a 学员 ✅ / 13b 权益 ✅ / 13c 预约下单 ✅ / 13d 候补 ✅ 均已验收并 merge main（dev=main，DB v12）。

## 先读（按序）
1. `pds/CLAUDE.md` —— 每轮实现流程。**注意**：CLAUDE.md 写的「邮件停止点」被 classifier 拦，本项目已改**会话内 approve / 验收，不发邮件**（13a-13d 实测惯例）。
2. `pds/PROGRESS.md` §5 的 **Batch 13a–13d** 段（学员/权益/预约下单/候补已闭环）+ §3 缺口表。
3. `pds/COURSE_BOOKING_BUSINESS_BLUEPRINT.md`（source of truth，优先级最高）：**§13 签到和爽约（§13.1 签到方式/权限、§13.2 签到后动作、§13.3 爽约处理）、§22.5 签到流程、§22.6 爽约流程、§20.12 签到/到课/履约记录、§9.2 Booking 状态机、§9.3 Entitlement 状态（hold→consumption / released）、§24.2 并发一致性（签到不能重复消费同一 hold）、§21.1/21.2 attendance 权限映射**。
4. 三库 `.learnings`（backend / web / pds）的 **Batch 13c + 13d** 段必读 —— 直接相关的可复用模式与踩坑：①新域照搬 booking/waitlist 流水线（domain→service(require+checker==nil bypass+data_scope)→persistence(tx+audit)→handler→wire）②**hold consume/release 记账复用 13c `settleHoldOnCancel`**（release：locked--/remaining++/txn release Δ+1；consume/forfeit：hold→consumed/locked--/consumed++/txn delta0）——13e 签到=consume 路径 + 加 attendance/consumption/session_record；no_show=按 policy release 或 consume③DB 约束→业务错误别裸 500（23505 unique 按约束名分流，读 `pgErr.ConstraintName`，复用 `uniqueConstraint`）④`entitlement.SettleStatus` 纯函数 + `SELECT FOR UPDATE` 锁额骨架⑤**domain struct 经 REST 返回必加 snake_case json tag**（13c P0：漏 tag→HTTP PascalCase，struct 单测测不到；13d 已靠此经验零 P0）⑥前端新 api client 登记 `packages/api/package.json` exports；mutation 改 A 实体带动 B 的派生计数/快照时两者都失效；模态/行展示的父实体派生态从 live query 派生不用冻结快照（13d drawer 教训）。
5. 核对 `backend/migrations/000003_course_booking_schema.up.sql` 这些表的**真实列/约束/unique**：`attendance_records`（booking_id/class_session_id/profile/marked_by/attended_at/note；**unique(booking_id) 防重签**）、`entitlement_consumptions`（entitlement_hold_id/learner_entitlement_id(RESTRICT)/booking_id/attendance_id/credits/`consumption_type∈{attendance,no_show,manual}`/operated_by；**unique(entitlement_hold_id) WHERE NOT NULL 防重复消费同 hold**）、`session_records`（class_session_id/booking_id/attendance_id/instructor_profile_id/`record_type∈{attendance,no_show,manual}`/note/`metadata JSONB NOT NULL DEFAULT '{}'`）、`bookings`（status 机 booked→attended/pending_no_show→no_show；13c 只做了 booked/cancelled）、`entitlement_holds`（status held/released/**consumed**；13c/d 用了 held/released，13e 落 consumed）、`entitlement_transactions`（action **consume/no_show_consume** 待用）、`brand_booking_policies`（`no_show_consumes_entitlement` 决定爽约 consume vs release）、`class_sessions`（status scheduled/in_progress/completed/cancelled；13c/d 只用 scheduled）。
6. 核对 000003 的 **attendance 权限 seed**：`attendance.view / attendance.mark / attendance.no_show_confirm` 三码已 seed，**核对 §21.1/21.2 角色映射是否齐**（§13.1：前台/店长/instructor/owner/admin 可签到；爽约确认 owner/admin/店长/前台，instructor 似只 view+mark 无 no_show_confirm——核对）。缺则补 migration（镜像 13b 000011 三段式，复用粗码优先，别无脑拆细码）；齐则零权限迁移。**grep `grep -nE "attendance\." migrations/000003*` 先看映射全不全**（13b 发现 entitlement 映射缺要补，13c/13d 发现 booking 映射齐不用补——别默认要 backfill）。

## Schema 现实（表已建，attendance/consumption/session_records 零代码）
全部表在 000003 已建。13a-13d 的 learner/entitlement/booking/waitlist 域已就绪可绑。本批新增 attendance 域（或并入 booking 域，grill 拍板），**大概率零或仅 1 个权限 migration**（attendance.* 映射若 §21 缺）。逐表核对 unique/约束/状态机后再设计。13e migration 若需要从 **000013** 起。

## 核心难点：签到/爽约的 hold 收口原子性（§24.2，grill 必须透）

**签到（attendance）单事务**：
1. `SELECT FOR UPDATE` 锁 `bookings` 行 → 校 status='booked'（非 booked→已处理）；锁场次校 status≠cancelled。
2. booking → `attended`。
3. INSERT `attendance_records`（unique(booking) 兜重签 → 23505 业务错误）。
4. **hold 收口**（复用 13c settleHoldOnCancel 的 consume 路径）：`held` hold → `consumed`，`SELECT FOR UPDATE` entitlement 锁 → `locked--`/`consumed++`（remaining 不变，hold 时已扣）→ re-settle → INSERT `entitlement_consumptions`(type=attendance, unique(hold) 兜重复消费) → `entitlement_transactions`(action=consume, delta=0, balance_after)。
5. INSERT `session_records`(record_type=attendance)。
6. **无权益占位**（requires_entitlement_fix，无 hold）：仍可 attended + attendance + session_record，**不消费**（§13.2 提示无权益 + 记异常，留人工补）。
7. audit。

**爽约（no_show）单事务**：booking pending_no_show → `no_show` + hold 按 policy `no_show_consumes_entitlement`：true→consume（同上 consumption type=no_show, txn no_show_consume）/ false→release（locked--/remaining++/txn release）+ session_record(no_show) + 记处理人原因 + audit（§24.3 爽约确认必审）。

**pending_no_show 触发**（grill 决定点，§22.6「场次结束→未签到→PendingNoShow」无 cron）：手动「结束场次」action 批量转 un-attended booked→pending_no_show（+场次→completed？）／惰性按 now>ends_at 派生待爽约队列。

并发兜底靠 `attendance_records` unique(booking) + `entitlement_consumptions` unique(hold) + 行锁，不做应用层先查后插（同 13c/d）。

## grill 设计树重点（写契约前 grill 透，发现不可行就停下问用户）
- **范围**：13e = brand 端 **手动签到（booked→attended + consume）+ 爽约确认（pending_no_show→no_show + release/consume）+ 履约记录展示**。撤销误签到（§20.12 留人工权益调整）、学员扫码自助签到（C 端 Batch 14）**不做**。
- **跨域依赖**：0 个未实现域（booking/entitlement/classsession/learner 全就绪）→ 可独立闭环。
- **hold 记账复用**：consume/release 正是 13c `settleHoldOnCancel` 的 forfeit/release 两路，抽共享 helper 或扩它，加 consumption/attendance/session_record 行 + 不同 txn action。**不限次会员卡**：consume=locked--/consumed++（remaining 保持 NULL）、txn delta0，同 13c 不限次路径。
- **attendance 域 vs 并入 booking 域**：新建独立 `attendance` 域，还是 attendance/no_show 作为 booking 域的新方法（它们改 booking 状态 + 复用 booking 的 hold 收口）？倾向**并入 booking 域 / 或薄 attendance 域复用 booking repo**（同 13d waitlist 持 bookingRepository 的模式），grill 拍板。
- **占位预约签到**：requires_entitlement_fix booking 签到 → attended + attendance（无 consumption），requires_entitlement_fix 标志保留待补；确认这个边界。
- **场次签到入口**：按场次列预约学员 + 标到课（§20.12）。从 `/schedule` 场次行 drawer（镜像 13d 候补 drawer），还是 /bookings 筛选？grill 拍板 UI 放置。
- **学员「履约记录」Tab**：13a 在学员详情留了占位 Tab（13b 填权益、13c 填预约、**13e 填履约**），列 session_records / attendance / no_show。
- **跨批回改检查**：场次取消（13c TX-3）已级联 cancel booking + release hold + cancel waitlist；13e 后场次取消是否要考虑 attended/pending_no_show booking？——attended 终态不动（已消费），pending_no_show 若场次取消应 cancel（释放）。核对 TX-3 是否需再扩。

## 拆批/范围决策点（grill 后用 AskUserQuestion 拍板，≤4 问；甜区 4-6，超 4 拆必答/备选）
1. **pending_no_show 触发模型**：手动「结束场次」action（session→completed + un-attended booked→pending_no_show 批量）／惰性按 now>ends_at 派生待爽约队列（无显式中间态落库）／员工逐个标。影响场次状态机是否动 + no-show 队列形态。
2. **attendance 落点**：独立 attendance 域（持 bookingRepository 复用 hold 收口，镜像 13d waitlist）／并入 booking 域加方法。影响代码组织。
3. **签到 UI**：/schedule 场次行 drawer（镜像 13d 候补）／/bookings 扩展。
4. **爽约 hold 边界确认**：no_show 按 policy `no_show_consumes_entitlement`（true 扣课/false 退）；占位预约签到不消费、保留 fix 标志待补；撤销误签到不做（人工调整）。确认这组边界。
5. （若需要）attendance.* 权限映射是否齐 / 补 migration 方案。

## 流程铁律（本项目惯例，严格遵守）
grill 设计树 → 写契约 `pds/batches/batch-13e-*.md` → **会话内 approve**（不发邮件）→ 写测试场景 `batch-13e-*-tests.md` → **主线程逐 task TDD commit**（用户不要 spawn 实现 subagent；先红→实现→绿→单 task commit，按文件/域为粒度）→ `go build ./... && go test ./...` + `pnpm --filter @mini-schedule/brand lint+build` → `/code-review`（2 个并行 review subagent：后端 correctness / 前端 correctness，修或转 FR）→ **业务验收停止点**（e2e 由用户另开 session 跑，你给自包含 prompt）→ 更新 PROGRESS + 三库 `.learnings` → 三仓库分别 push → **backend/web 的 dev FF 合并进 main**（`git push origin dev:main`；pds 直接 main）。
- 仓库：backend/web 在 `dev` 分支开发，pds 在 `main`。
- 验收铁律：后端 e2e 前必 `go build -o /tmp/api-brand ./cmd/api-brand` 重建重启（端口 **8081**，旧二进制掩盖改动）；前端 `rm -rf .next` 重启（:3002）+ curl 预热；filter 用包名 `@mini-schedule/brand`（`--filter=brand` 报错）；除 e2e 外必跑 prod `pnpm build`。
- DB 单测用真实 Postgres（`newMigratedTestDB` 自动建库跑 migration，无 PG 时 skip）。**dev DB 现 v12**（13a=9/10，13b=11，13c=12，13d=零 migration）；13e migration 若需要从 **000013** 起。
- **派生态/落库类（consume 改 locked/consumed/remaining、attendance/consumption/session_record 落库、no_show release）验收须 psql 实查 DB 真值**，不能只看前端（13b/c/d settle 落库经验）。

## 测试账号（brand 21，⚠️ 沿用 13c/d 状态）
- owner `18816820405 / admin123`（user_id=16，fast-path 全权限）。
- ⚠️ 只读账号 `13900139777` 现为 `course_operator`（13b 验收期改）——13e 权限门测试按它现在的角色（course_operator 有无 attendance.view？核对 seed；mark/no_show_confirm 应 403）。
- 门店「讯美广场」id=1（active）；brand21 多数历史门店/课程软删，可选只剩 讯美广场(loc1)+晨间瑜伽(course4 一带)。
- **13e 前置数据**：需 ① scheduled 场次（/schedule）② 学员（/learners）③ active 权益（/entitlement-products + 学员权益 Tab）④ **已下单的 booking**（/bookings 代预约，13c）——签到对象是 booked 预约。验收占位签到需 requires_entitlement_fix booking（13c none 模式）。爽约需「场次结束/过期」的 booked 预约（排一个 starts_at 已过的场次或用结束场次 action）。

## 复用模式（13a-13d 已验证，直接照搬）
- 新域照搬流水线：domain（实体+Status+IsValid*+Repository）→ application/service（`require(code)` + `checker==nil` bypass + data_scope `scopeFilterIDs`/`guardLocationInScope`，越权 404）→ persistence（事务 + `audit.Write` + DB 约束按约束名分流）→ interfaces/brand handler → wire `go generate` 重生（NewHandler 加参数，router_test 补 nil）。
- **hold 收口**：复用/扩 13c `settleHoldOnCancel`（自由函数，同包）——release（locked--/remaining++/re-settle/txn release Δ+1）与 consume/forfeit（hold→consumed/locked--/consumed++/txn delta0）两路；13e consume 加 `entitlement_consumptions`(unique hold) + `attendance_records`(unique booking) + `session_records`。
- 若 attendance 独立域：持 `&bookingRepository{db}` 复用其方法（同 13d waitlist；方法只用 tx 不触 receiver.db，事务边界完整）。
- 错误码加 `pkg/errors/error.go`；新域权限先 grep 000003 seed 看映射全不全（粗码优先），缺才补 migration 三段式（镜像 000011）。
- 前端：types（**domain struct 后端记得加 snake_case json tag**）+ api client（登记 exports）+ 错误码常量 + PERMISSIONS（attendance.*）；签到 drawer 镜像 13d 候补 drawer（场次行入口 + 列预约学员 + 标到课/确认爽约）；学员详情「履约记录」Tab 替换 13a 占位；mutation 跨查询失效（attendance 改 booking 状态 + 权益 + 派生计数）；模态派生态从 live query 派生（13d 教训）；DataTable + 权限门 disabled+Hint。

第一步只做 grill：盘 attendance/consumption/session_records 三表与已有域、画 Booking 状态机收口（booked→attended/pending_no_show→no_show）+ hold→consumed/released 的 tx 边界与并发点（unique 防重签/重复消费）、给拆批/范围建议 + 上述决策点推荐，然后用 AskUserQuestion 让用户拍板。拍板后再写 13e 契约。
