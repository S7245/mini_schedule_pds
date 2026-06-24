# Batch 13e 契约 —— 签到 / 履约 / 爽约（Attendance / Consumption / NoShow）

> Batch 13（学员预约闭环）**第五也是最后一子批**：把 13c 的 hold「锁额」收口为 consume / release。
> 前置 13a 学员 ✅ / 13b 权益 ✅ / 13c 预约下单 ✅ / 13d 候补 ✅ 均已 merge main（dev=main，DB v12）。
> 来源优先级：`COURSE_BOOKING_BUSINESS_BLUEPRINT.md` §13 / §20.12 / §22.5 / §22.6 / §9.2 / §9.3 / §24.2 / §21。

## 决策记录（grill 后 AskUserQuestion 拍板，2026-06-24）

| # | 决策点 | 选择 | 影响 |
|---|---|---|---|
| 1 | PendingNoShow 触发模型 | **显式「结束场次」action** | session `scheduled/in_progress→completed` + 未签到 booked 批量→`pending_no_show`；中间态落库；**TX-3 零改动**（completed 不可取消） |
| 2 | 签到/爽约代码组织 | **并入 booking 域** | booking 域加 `MarkAttendance`/`EndSession`/`ConfirmNoShow`；复用同包 `settleHoldOnCancel`/`insertBookingTransaction` 自由函数，无需导出内部 |
| 3 | 签到/爽约 UI 入口 | **/schedule 场次行 drawer** | 镜像 13d 候补 drawer；按场次列预约学员 + 标到课/结束场次/确认爽约 |
| 4 | 占位预约（requires_entitlement_fix，无 hold）签到 | **可签到但不消费，保留 fix 标志** | attended + attendance_record + session_record，不写 consumption；fix 标志保留待人工补（§13.2） |

## 数据模型 & Migration

**Migration = 零**（dev DB 保持 v12）。逐项核对 000003：
- `attendance_records`：唯一 `idx_attendance_records_booking_unique(booking_id)` 防重签。
- `entitlement_consumptions`：唯一 `idx_entitlement_consumptions_hold_unique(entitlement_hold_id)` 防重复消费同 hold；`consumption_type ∈ {attendance, no_show, manual}`；`learner_entitlement_id` 为 `ON DELETE RESTRICT`。
- `session_records`：`record_type ∈ {attendance, no_show, manual}`；`metadata JSONB NOT NULL DEFAULT '{}'`；无 unique（一 booking 一记录由业务保证，不靠约束）。
- `bookings.status` CHECK 已含 `attended / pending_no_show / no_show`（13c/d 只用了 booked/cancelled）。
- `entitlement_holds.status` CHECK 已含 `consumed`；`entitlement_transactions.action` CHECK 已含 `consume / no_show_consume`。
- `brand_booking_policies.no_show_consumes_entitlement BOOLEAN DEFAULT FALSE` 已就绪。
- 权限 seed 完整：`attendance.view / attendance.mark / attendance.no_show_confirm` + 角色映射全齐（owner/admin/location_manager/receptionist=全3；instructor=view+mark **无** no_show_confirm；location_assistant=view）。**course_operator 无任何 attendance 权限**（只读账号 13900139777 连 view 都 403）。→ **零权限 migration**。

## 状态机 + Hold 收口（核心，§9.2/§9.3/§24.2）

```
booked / pending_no_show ──签到[attendance.mark]──▶ attended
    ├ 有 hold(held): hold→consumed · locked-- · consumed++ (remaining 不变，下单时已扣)
    │   + INSERT attendance_records(unique booking) + entitlement_consumptions(type=attendance, unique hold)
    │   + session_records(attendance) + entitlement_transactions(consume, Δ0, balance_after)
    └ 占位(无 hold): attended + attendance_records + session_records，**不写 consumption**，fix 标志保留

scheduled/in_progress ──结束场次[attendance.mark]──▶ session=completed
    + 未签到的 booked 批量 → pending_no_show（hold 不动，booked_count 不动）

pending_no_show ──确认爽约[attendance.no_show_confirm]──▶ no_show
    ├ policy.no_show_consumes_entitlement=TRUE  → hold→consumed · consumed++ + consumption(no_show) + session_records(no_show) + txn(no_show_consume, Δ0)
    └ =FALSE → hold→released · locked-- · remaining++ (count-based) + session_records(no_show) + txn(release, Δ+1)
    占位(无 hold): no_show + session_records(no_show)，不结算
```

**关键不变量**：
- `attended` / `no_show` 为终态，不可逆（撤销误签到 v1 不做，§20.12 → 人工权益调整，转 FR）。
- 签到接受 `booked` **或** `pending_no_show`（场次已结束后补签到课的学员 → pending_no_show→attended，天然纠错，无需 undo）。
- `booked_count` 只由「取消」回退；attendance/no_show/end 均不动 booked_count（占用名额事实不变）。
- consume 路径在 count-based 卡 remaining **不变**（13c 下单时已 remaining--/locked++，签到只是 locked--/consumed++）；不限次卡 remaining 恒 NULL，txn Δ0。

## 并发兜底（§24.2，靠约束 + 行锁，不先查后插）
- 重复签到 → `attendance_records` unique(booking) 23505 → `ATTENDANCE_ALREADY_MARKED`。
- 重复消费同 hold → `entitlement_consumptions` unique(hold) 23505 → 同样映射 `ATTENDANCE_ALREADY_MARKED`（理论上被前者先挡，作并发深层兜底）。
- 锁序沿用 13c/d：**booking 行 → session 行 → entitlement 行**（与下单 TX-1 / 取消 TX-2 / 场次取消 TX-3 一致，避免死锁）。

## API 接口

所有路径前缀 `/api/v1`，brand 端，沿用 `booking` 域 handler 注册。

| 方法 | 路径 | 权限 | 请求字段 | 响应字段 |
|---|---|---|---|---|
| POST | `/bookings/:id/attend` | `attendance.mark` | `note?`（≤1000） | `Booking`（status=attended，含 hold.status=consumed） |
| POST | `/class-sessions/:id/end` | `attendance.mark` | — | `{ session_id, status: "completed", pending_no_show_count }` |
| POST | `/bookings/:id/no-show` | `attendance.no_show_confirm` | `reason?`（≤1000） | `Booking`（status=no_show，含 hold.status=consumed/released） |

**复用现有端点（零新增 GET）**：
- 签到 drawer 名单：`GET /bookings?class_session_id=:id&page_size=100`（`booking.view`，返回 status + hold，已 JOIN 课程/门店/学员）。
- 学员「履约记录」Tab：`GET /bookings?brand_learner_profile_id=:id`（`booking.view`），前端筛 status ∈ {attended, no_show, pending_no_show} 展示终态 + 消耗权益。

**data_scope**：三个写端点均按 booking 所属 `class_session.location_id ∈ assigned_locations` 守卫（镜像 13c `guardLocationInScope`，越权 404 `BOOKING_NOT_FOUND` / 场次端点 404 `SESSION_NOT_FOUND`）。

## 新增错误码（`pkg/errors/error.go`）

| 码 | HTTP | 触发 |
|---|---|---|
| `ATTENDANCE_ALREADY_MARKED` | 409 | 重复签到（booking 已 attended 或 attendance/consumption unique 23505） |
| `BOOKING_NOT_ATTENDABLE` | 409 | 签到时 booking status ∉ {booked, pending_no_show}（已取消/已爽约） |
| `SESSION_NOT_ENDABLE` | 409 | 结束场次时 session status ∉ {scheduled, in_progress}（草稿/已完成/已取消） |
| `BOOKING_NOT_CONFIRMABLE` | 409 | 确认爽约时 booking status ≠ pending_no_show |

签到/结束场次时若场次已取消 → 其 booking 已是 cancelled → 自然落 `BOOKING_NOT_ATTENDABLE`（§22.5「场次已取消不可签到」）。

## 事务设计

**TX-A 签到 `Attend(bookingID)`**：
1. `SELECT FOR UPDATE` booking 行（不存在/越权→`BOOKING_NOT_FOUND`）；校 status ∈ {booked, pending_no_show}，否则 `BOOKING_NOT_ATTENDABLE`。
2. `SELECT FOR UPDATE` session 行；校 status ≠ cancelled。
3. booking → `attended`。
4. INSERT `attendance_records`（unique(booking) → 23505 按约束名 → `ATTENDANCE_ALREADY_MARKED`）。
5. **hold 收口**（复用/扩 `settleHoldOnCancel` 的 consume 骨架）：若有 `held` hold → 锁 entitlement → hold→consumed / locked-- / consumed++ / re-settle → INSERT `entitlement_consumptions`(type=attendance, attendance_id, unique(hold)→23505→`ATTENDANCE_ALREADY_MARKED`) → `entitlement_transactions`(action=consume, Δ0, balance_after, consumption_id)。无 hold（占位）→ 跳过，不写 consumption。
6. INSERT `session_records`(record_type=attendance, attendance_id, instructor_profile_id 取场次教练)。
7. `audit.Write`。返回更新后 Booking（GetByID）。

**TX-B 结束场次 `EndSession(sessionID)`**：
1. `SELECT FOR UPDATE` session 行；校 status ∈ {scheduled, in_progress}，否则 `SESSION_NOT_ENDABLE`。
2. session → `completed`。
3. 批量 `UPDATE bookings SET status='pending_no_show' WHERE class_session_id=:id AND status='booked'`（hold/ booked_count 不动）。
4. `audit.Write`。返回 `{session_id, status, pending_no_show_count}`。

**TX-C 确认爽约 `ConfirmNoShow(bookingID)`**：
1. `SELECT FOR UPDATE` booking 行（404 同上）；校 status = pending_no_show，否则 `BOOKING_NOT_CONFIRMABLE`。
2. 解析该场次生效 policy（base location/brand-default + 场次 override）取 `no_show_consumes_entitlement`。
3. booking → `no_show`（记 cancel_reason 复用为处理原因？否——no_show 用 session_records.note 记原因，bookings 不动 cancel_* 字段）。
4. **hold 收口**：有 `held` hold → consume(true：no_show_consume txn) 或 release(false：release txn)，复用 `settleHoldOnCancel`（consume 路径加 consumption(type=no_show)）。占位无 hold → 跳过。
5. INSERT `session_records`(record_type=no_show, note=reason, instructor_profile_id)。
6. `audit.Write`。返回更新后 Booking。

**复用 helper 扩展**：`insertBookingTransaction` 增 `consumptionID *int64` 形参（现有调用点传 nil）；consume 收口抽 `consumeHoldForAttendance(tx, ..., consumptionType, attendanceID)` 或在 `settleHoldOnCancel` 基础上加 consumption 写入分支（实现时定，倾向新 sibling 自由函数共享锁+settle 骨架，避免污染 cancel 语义）。

**TX-3（场次取消级联）回改判定 = 零改动**：Cancel 仅允许 `scheduled/in_progress`（`class_session_repository.go:255`）；EndSession 后场次为 `completed` 不可取消 → `pending_no_show` 只存在于 completed 场次 → TX-3 永不触及；attended/no_show 终态本就不在 TX-3 的 `status='booked'` 查询内。已核对，无需扩。

## 前端页面模块（`web/apps/brand`）

| 页面/模块 | 类型 | 关键字段/操作 |
|---|---|---|
| `/schedule` 场次行 | 入口 | 每行加「签到」按钮（`attendance.view` 门），点开 AttendanceDrawer |
| **AttendanceDrawer** | 抽屉 | 头部：课程/时间/门店 + 计数（已约/已到课/待爽约/已爽约，从名单 live 派生）+「结束场次」按钮（`attendance.mark` 门，session 非 scheduled/in_progress 时 disabled）。Body：名单（`GET /bookings?class_session_id&page_size=100`），每行 学员名/手机 + 状态徽标 + 权益(hold.product_name 或「无权益·占位」) + 操作「标到课」(booked\|pending_no_show，`attendance.mark` 门) /「确认爽约」(pending_no_show，`attendance.no_show_confirm` 门) |
| 学员详情 **「履约记录」Tab** | Tab | 替换 13a 占位；`GET /bookings?brand_learner_profile_id`，列终态 booking：课程/时间/门店 + 结果徽标(到课/爽约/待爽约) + 消耗权益(hold 状态 consumed/released) |
| types / api client | 代码 | `attend(bookingId, note)` / `endSession(sessionId)` / `confirmNoShow(bookingId, reason)`；登记 `packages/api` exports；4 个错误码常量；PERMISSIONS 加 `attendance.view/mark/no_show_confirm` |

**跨查询失效**（mutation 后）：签到/爽约/结束场次改 booking 状态 + entitlement（consume/release 动 remaining/locked/consumed）+ 场次派生计数 → 失效 `brand-class-sessions`、`bookings`（按 session + 按 learner）、该学员 `entitlements` / `entitlement-transactions`。模态计数从 live query 派生不用冻结快照（13d 教训）。

## Wireframe（AttendanceDrawer，镜像 13d WaitlistDrawer）

```
┌─ 签到 · 晨间瑜伽 ───────────────────────────────┐
│ 2026-06-25 09:00–10:00 · 讯美广场              │
│ 已约 2 · 已到课 1 · 待爽约 0 · 已爽约 0         │
│                              [ 结束场次 ]      │   ← attendance.mark 门
├───────────────────────────────────────────────┤
│ 张三 · 138…  [已到课]  月卡           —        │
│ 李四 · 139…  [已约]    次卡(剩3)  [标到课][确认爽约(禁)]│  ← 爽约仅 pending_no_show 可点
│ 王五 · 137…  [已约]    无权益·占位  [标到课]    │   ← 占位：标到课不消费，提示「无权益已记异常」
└───────────────────────────────────────────────┘
结束场次后 → 未签到行变「待爽约」，[确认爽约] 启用
```

## 前端实现约束
- 复用 `/web/apps/brand` 现有 DataTable / Drawer / Dialog / Badge / 权限 Hint 组件，不引新 UI 库。
- AttendanceDrawer 直接镜像 `WaitlistDrawer`（13d）结构与 react-query 失效模式。
- 权限门：无权限按钮 `disabled` + Hint tooltip（镜像 13c/d）。
- 占位预约签到给明确提示（无权益、已记异常、待人工补）。

## 范围 / 转 FR
- **IN**：手动签到（booked/pending_no_show→attended + consume）、结束场次产 pending_no_show、确认爽约（→no_show + release/consume）、学员「履约记录」Tab。
- **OUT（转 FR）**：撤销误签到（§20.12 人工权益调整）；cron 自动结束场次 / 自动爽约（§22.6 须员工确认）；C 端扫码自助签到（Batch 14）；场次行 attended/no_show 计数徽标（drawer 内已派生，列表徽标留 FR，避免动 class_session list 查询）；批量签到（一键全到课）；session_records 富履约数据（评价/训练量）。

## 验收前置数据（brand 21，沿用 13c/d 状态）
- owner `18816820405 / admin123`（fast-path）；只读 `13900139777`=course_operator（attendance 全 403）。
- 需：① scheduled 场次（/schedule）② 学员（/learners）③ active 权益（学员权益 Tab）④ 已下单 booking（/bookings 代预约）。占位签到需 none 模式 booking（requires_entitlement_fix）。爽约需先「结束场次」产 pending_no_show。
- 落库类（consume 改 locked/consumed、attendance/consumption/session_record 落行、no_show release）验收须 psql 实查 DB 真值。
