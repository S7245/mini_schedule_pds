# Batch 13e 测试场景 —— 签到 / 履约 / 爽约（标到课 / 结束场次 / 确认爽约 / hold 收口）

> 契约：[batch-13e-attendance.md](batch-13e-attendance.md)。混合执行（镜像 13c/d）：happy 走 UI；消费/释放落库、状态机、并发、越权走 API + **psql 实查**。
> **派生态铁律**：签到 consume（hold→consumed/locked--/consumed++ + attendance/consumption/session_record 落行）、爽约 release（locked--/remaining++）、结束场次批量转 pending_no_show 必 psql 实查 DB 真值，不能只看前端。

## 前置数据
- owner `18816820405 / admin123`（fast-path 全权限）。
- 只读 `13900139777` 现为 `course_operator`——**无任何 attendance 权限**（view/mark/no_show_confirm 全 403，按钮不可见）。
- instructor 角色账号：有 `attendance.view + attendance.mark`，**无 `attendance.no_show_confirm`**（确认爽约 403/disabled）——用于 P 系列。
- 门店 讯美广场 loc1；**scheduled 场次** + 测试学员 ×3 + active 权益：① 次卡（count-based，remaining 有值）② 不限次会员卡（remaining=NULL）。
- 已下单 booking（/bookings 代预约 13c）：次卡 booked、会员卡 booked、**none 模式占位** booked（requires_entitlement_fix=true，无 hold）。
- policy 两套：`no_show_consumes_entitlement=false`（默认，测 release）与 `=true`（测 consume）。

## Happy Path（UI 优先）

| # | 步骤 | 预期 |
|---|---|---|
| H1 | /schedule 场次行点「签到」 | AttendanceDrawer 打开；头部课程/时间/门店 + 计数（已约 N）；名单列该场次预约学员（已约状态 + 权益名） |
| H2 | 对次卡 booked 预约点「标到课」 | booking→attended。**psql**：`attendance_records`(booking 唯一)、`entitlement_consumptions`(type=attendance, hold/attendance 关联)、`session_records`(attendance)、`entitlement_holds`→consumed、`learner_entitlements` locked--/consumed++ **remaining 不变**、`entitlement_transactions`(consume, delta=0, consumption_id 非空) |
| H3 | 对不限次会员卡 booked 标到课 | attended；**psql**：hold→consumed、consumed++，**remaining 保持 NULL**，txn(consume, delta=0, balance_after NULL) |
| H4 | 对占位预约（requires_entitlement_fix）标到课 | attended + `attendance_records` + `session_records`；**psql**：**无 `entitlement_consumptions` 行**、无 hold 变更、requires_entitlement_fix 仍 true（前端提示「无权益·已记异常·待补」） |
| H5 | drawer 点「结束场次」 | session→completed；未签到的 booked 批量→pending_no_show；返回 pending_no_show_count；名单未签到行变「待爽约」、「确认爽约」启用 |
| H6 | policy=false 场次，对 pending_no_show 点「确认爽约」填原因 | booking→no_show。**psql**：hold→released、locked--/**remaining++**、txn(release, delta=+1)、`session_records`(no_show, note=原因)；无 consumption |
| H7 | policy=true 场次，对 pending_no_show「确认爽约」 | booking→no_show。**psql**：hold→consumed、consumed++、txn(**no_show_consume**, delta=0)、`entitlement_consumptions`(type=no_show)、`session_records`(no_show) |
| H8 | 结束场次后对某 pending_no_show 行点「标到课」（补签纠错） | attended + consume（同 H2 路径）；该行不再算待爽约 |
| H9 | 学员详情「履约记录」Tab | 列该学员终态 booking（到课/爽约/待爽约）+ 课程/时间/门店 + 消耗权益(hold consumed/released) |

## Edge（API + psql）

| # | 场景 | 预期 |
|---|---|---|
| E1 | 对已 attended 预约再标到课 | `ATTENDANCE_ALREADY_MARKED` 409；psql 无第二条 attendance/consumption |
| E2 | 对已 cancelled 预约标到课 | `BOOKING_NOT_ATTENDABLE` 409 |
| E3 | 对 cancelled 场次的预约签到（其 booking 已 cancelled） | `BOOKING_NOT_ATTENDABLE` 409（§22.5 场次已取消不可签到） |
| E4 | 结束已 completed / cancelled 场次 | `SESSION_NOT_ENDABLE` 409 |
| E5 | 对 booked（非 pending_no_show）确认爽约 | `BOOKING_NOT_CONFIRMABLE` 409 |
| E6 | 对占位预约（无 hold）确认爽约 | 成功 no_show + `session_records`(no_show)；**psql**：无 hold/consumption 变更，不报错 |
| E7 | 签到 note / 爽约 reason 超长（>1000） | 400 校验 |
| E8 | 越权 booking id（别的 brand / 不存在） | 404 `BOOKING_NOT_FOUND` |
| E9 | 并发：两请求同时对同一 booking 标到课 | 串行化（booking 行锁）→ 仅一成功；另一 `ATTENDANCE_ALREADY_MARKED`（attendance unique + consumption unique 兜底）；psql 各仅一行 |
| E10 | 结束场次时无未签到 booked（全已到课） | session→completed，pending_no_show_count=0 |

## 权限 + data_scope

| # | 场景 | 预期 |
|---|---|---|
| P1 | course_operator 标到课 / 结束场次 / 确认爽约 | 全 403；前端按钮不可见（无 attendance.view） |
| P2 | instructor（view+mark，无 no_show_confirm）标到课 | 200；**确认爽约 403**；前端确认爽约按钮 disabled + Hint |
| P3 | 越权（非 assigned_locations 场次）标到课 / 结束场次 / 确认爽约 | 404（BOOKING_NOT_FOUND / SESSION_NOT_FOUND） |
| P4 | 前端：无 attendance.mark 时「标到课」「结束场次」disabled + Hint；无 no_show_confirm 时「确认爽约」disabled | 按钮禁用 |

## TX-3 回归（场次取消级联，13e 后零改动需复验）

| # | 场景 | 预期 |
|---|---|---|
| C1 | scheduled 场次有 attended + booked 预约 → 取消场次 | **psql**：attended 不变（终态，已消费 hold consumed 不回退）；booked 走 13c 级联 cancel + hold released + booked_count=0 |
| C2 | 已 completed 场次（有 pending_no_show）尝试取消 | `SESSION_CANCEL_NOT_ALLOWED` 409（completed 不可取消 → pending_no_show 不被 TX-3 触及，证 TX-3 零改动安全） |

## 13c/13d 回归（settleHoldOnCancel 扩展后必复跑）
- 13c 全部 DB 单测（TestBooking_* / TestSessionCancel_*）+ 13d（TestWaitlist_*）仍绿——consume 收口扩展（加 consumption 写入 / insertBookingTransaction 加 consumption_id 形参 nil）不得改变取消 release/forfeit 行为。

## psql 实查清单
```sql
SELECT id,status FROM bookings WHERE id=?;
SELECT id,status,booked_count FROM class_sessions WHERE id=?;
SELECT id,booking_id,class_session_id,marked_by,attended_at FROM attendance_records WHERE booking_id=?;
SELECT id,entitlement_hold_id,booking_id,attendance_id,consumption_type,credits FROM entitlement_consumptions WHERE booking_id=?;
SELECT id,class_session_id,booking_id,record_type,note FROM session_records WHERE booking_id=?;
SELECT id,status,held_at,released_at,consumed_at FROM entitlement_holds WHERE booking_id=?;
SELECT status,total_credits,remaining_credits,locked_credits,consumed_credits FROM learner_entitlements WHERE id=?;
SELECT action,delta_credits,balance_after,hold_id,consumption_id FROM entitlement_transactions WHERE booking_id=? ORDER BY id;
```

## 执行方式
后端 e2e 前 `go build -o /tmp/api-brand ./cmd/api-brand` 重建重启（:8081，旧二进制掩盖改动）；前端 `rm -rf .next` 重启（:3002）+ curl 预热；filter 用 `@mini-schedule/brand`。Happy 走 chrome-devtools UI，Edge/消费落库/释放/状态机/并发/越权走 API + psql 实查。并发（E9）用并行 curl。DB 单测 `newMigratedTestDB`（v12，**无新 migration**）。除 e2e 外必跑 prod `pnpm --filter @mini-schedule/brand build`。
