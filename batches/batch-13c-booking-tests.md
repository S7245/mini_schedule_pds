# Batch 13c 测试场景 —— 预约下单 + 代取消 + 场次取消级联

> 契约：[batch-13c-booking.md](batch-13c-booking.md)。
> 混合执行（镜像 13a/13b/12b）：happy path 走 chrome-devtools UI；满员/并发/频次/越权/级联/release 落库走 **API 直连 + psql 实查**。
> **派生态铁律**：凡改 `booked_count` / `remaining`/`locked`/`consumed` / hold status / txn 的场景，验收**须 psql 实查 DB 真值**，不能只看前端。

## 前置数据（测试库凑齐）

| 项 | 来源 | 备注 |
|---|---|---|
| owner `18816820405 / admin123` | 既有 | user_id=16，fast-path 全权限 |
| course_operator `13900139777` | 13b 验收已重指派 | 仅 view，代预约/代取消应 403 |
| receptionist 测试账号 | `/staff` 建（前台 location 角色，绑 loc1） | 验 §21.2 前台可代预约/代取消 |
| 门店 讯美广场 `loc1` (active) | 既有 | data_scope 锚点 |
| scheduled 场次 ×N | `/schedule` 排课（晨间瑜伽 course4 @ loc1） | 含 capacity=1 的（测满员/并发）+ capacity≥3 的（测级联）+ 不同 starts_at（测窗口/截止） |
| 测试学员 ×N | `/learners`（13a） | ≥3 个，验频次/并发/级联 |
| active 权益 | `/entitlement-products` + 学员权益 Tab（13b） | 次卡（count-based，剩余≥3）+ 会员卡（不限次）+ 一张已 frozen/expired 的（验 NOT_USABLE）+ 一张 course/location scope 不匹配的（验 SCOPE_MISMATCH） |

---

## Happy Path（UI 优先）

| # | 步骤 | 预期结果 |
|---|---|---|
| H1 | owner 访问 `/bookings` | 页面加载，列表/空态正常，「代预约」按钮可见（有 create_assisted） |
| H2 | 代预约弹窗：选学员 A + scheduled 场次 S + 模式「自动选择」 | 预览显示「将使用 次卡-剩N次(到期日)」 |
| H3 | H2 点确认 | 200；列表新增「已预约」行。**psql**：`bookings`(staff_assisted/booked/assisted_by=16)、`class_sessions.booked_count` +1、`learner_entitlements` remaining-1/locked+1、`entitlement_holds`(held, unique booking)、`entitlement_transactions`(hold, delta=-1, balance_after) |
| H4 | 代预约：模式「手动指定」→ 下拉选「会员卡(不限次)」 | 200。**psql**：hold 建、`remaining` 不变(NULL)、`locked`+1、txn(hold, delta=0, balance_after=NULL) |
| H5 | 代预约：模式「无权益占位」→ 填原因「待补卡」 | 200；行显示 ⚠待补权益。**psql**：`requires_entitlement_fix=true` + `no_entitlement_reason`，**无 hold 行**；OperationLog 记「无权益代预约」 |
| H6 | 对 H3 的预约点「代取消」→ 确认 | 200；行→已取消。**psql**：`bookings`(cancelled/cancel_source=staff/cancelled_by/cancelled_at)、`booked_count` -1、hold→released+released_at、entitlement remaining+1/locked-1（re-settle）、txn(release, delta=+1) |
| H7 | 对已取消的 学员A+场次S 再次代预约 | 200（partial-unique 放行）；**psql**：新 `bookings` 行（旧 cancelled 行保留） |
| H8 | 学员详情页「预约」Tab | 替换 13a 占位，列出该学员的预约（含已取消历史） |
| H9 | `/booking-policy` 读默认 → 改 `cancel_deadline=120`、`book_ahead_max=10080`、`daily_limit=2` → 保存 | 200 持久化；**psql**：`brand_booking_policies`(location_id IS NULL) upsert |

---

## Edge Cases —— 下单（API 直连 + psql）

| # | 场景 | 预期结果 |
|---|---|---|
| E1 | capacity=1 场次约满后再约 | `SESSION_FULL` 409 |
| E2 | **并发抢最后名额**：2 并发 POST 同 capacity=1 场次 | 一成一 `SESSION_FULL`；**psql** `booked_count=1`（无超卖） |
| E3 | 并发同学员同场次 2 次 POST | 一成一 `BOOKING_DUPLICATE` 409；**psql** 仅 1 active booking |
| E4 | 早于 book_ahead_max / 晚于 book_ahead_min 窗口 | `BOOKING_WINDOW_CLOSED` 409 |
| E5 | daily/weekly/concurrent 设 1 后第 2 次预约 | `BOOKING_FREQUENCY_EXCEEDED` 409，Details 带 which/limit/current |
| E6 | 月限：权益产品 monthly_limit=1 第 2 次（policy 无月限，验月限独家来自产品） | `BOOKING_FREQUENCY_EXCEEDED` 409 (which=monthly) |
| E7 | 对 draft / cancelled / completed 场次预约 | `SESSION_NOT_BOOKABLE` 409 |
| E8 | auto 模式，学员无任何可用权益 | `ENTITLEMENT_NONE_AVAILABLE` 409 |
| E9 | manual 指定 expired/frozen/depleted 权益 | `ENTITLEMENT_NOT_USABLE` 422 |
| E10 | manual 指定 location/course scope 不匹配场次的权益 | `ENTITLEMENT_SCOPE_MISMATCH` 422 |
| E11 | manual 指定非法 → **不回退自动** | 报错（E9/E10），**psql 无 booking 落库**（未静默改用有效权益） |
| E12 | none 模式缺 no_entitlement_reason | `ASSISTED_REASON_REQUIRED` 422 |
| E13 | none 模式但场次已满 | `SESSION_FULL` 409（§5.8 占位不绕容量）；**psql** booked_count 不变 |
| E14 | frozen/inactive 学员预约 | `LEARNER_NOT_BOOKABLE` 409 |
| E15 | §5.7 自动选择优先级：学员持「指定课程权益 + 通用权益 + 更早过期权益 + 次卡 + 会员卡」 | auto 选中符合「指定课程 > 最早过期 > 次卡>会员卡 > 体验包仅体验态」的那张（`usable-entitlements` 首项 auto_selected=true 一致） |

## Edge Cases —— 取消（API + psql）

| # | 场景 | 预期结果 |
|---|---|---|
| E16 | 对已 cancelled / attended 的 booking 再取消 | `BOOKING_NOT_CANCELLABLE` 409 |
| E17 | 超过 cancel_deadline（policy cancel_deadline 设大，场次临近） | `BOOKING_CANCEL_DEADLINE_PASSED` 409 |
| E18 | 场次 override `allow_cancel=false`（SQL 注入 override 行，本批不做 CRUD） | `BOOKING_CANCEL_NOT_ALLOWED` 409 |
| E19 | `release_on_cancel=false` 取消 → forfeit | **psql**：hold→`consumed`、`remaining` 不回退、`locked`-1/`consumed`+1、txn(consume) |

## 场次取消级联（psql 实查）

| # | 场景 | 预期结果 |
|---|---|---|
| C1 | 场次 S(capacity≥3) 有 3 个 active booking（混次卡+会员卡+占位）→ 取消场次 S | **psql**：3 booking 全 `cancelled`(cancel_source=session_cancelled)；2 个有 hold 的→released + 各 entitlement 退额(re-settle)；占位无 hold 不动权益；`booked_count=0`；session_cancelled OperationLog metadata 含 cascaded count=3 |
| C2 | 级联恒退：policy `release_on_cancel=false` 时取消场次 | **psql** holds 仍 released（场次取消忽略 release_on_cancel） |
| C3 | **并发**：取消场次 S 同时并发对 S 下单 | 串行化（同 session 行锁）；终态一致——要么下单先成功后被级联取消，要么下单撞 cancelled 场次返 `SESSION_NOT_BOOKABLE`；**psql** 不出现 cancelled 场次上 status=booked 的脏 booking |
| C4 | 回归 B11/12 单场次取消（无 booking） | 仍正常 cancelled，级联空集不报错 |

## 权限门 + data_scope

| # | 场景 | 预期结果 |
|---|---|---|
| P1 | course_operator(13900139777) 代预约 / 代取消 | 403（仅 booking.view）；`GET /bookings` 200 |
| P2 | receptionist 代预约 / 代取消本店(loc1)场次 | 200（§21.2 前台） |
| P3 | **越权**：员工对非 assigned_locations 的场次代预约 / 代取消 | `404`（SESSION_NOT_FOUND / BOOKING_NOT_FOUND，不泄漏存在性） |
| P4 | `PUT /booking-policy`：receptionist(仅 schedule.view) | 403；course_operator/location_manager(有 schedule.manage) → 200（**复用 schedule.manage 的已知副作用，非 bug**；记录于 FR） |
| P5 | 前端：无 create_assisted 账号登录 | 「代预约」按钮 disabled + Hint tooltip |

## psql 实查清单（每条派生态场景对照）

```sql
-- booking 落库
SELECT id,status,source,cancel_source,requires_entitlement_fix,assisted_by,cancelled_by FROM bookings WHERE class_session_id=? ORDER BY id;
-- 场次容量
SELECT id,status,capacity,booked_count FROM class_sessions WHERE id=?;
-- 权益账（hold/release/forfeit 后）
SELECT id,status,total_credits,remaining_credits,locked_credits,consumed_credits FROM learner_entitlements WHERE id=?;
-- hold
SELECT id,booking_id,status,credits,released_at,consumed_at FROM entitlement_holds WHERE booking_id=?;
-- 流水
SELECT action,delta_credits,balance_after,booking_id,hold_id FROM entitlement_transactions WHERE learner_entitlement_id=? ORDER BY id;
```

## 执行方式

1. 后端 e2e 前：`go build -o /tmp/api-brand ./cmd/api-brand` 重建重启(:8081，旧二进制掩盖改动)。
2. 前端：`rm -rf apps/brand/.next` 重启(:3002) + `curl /login` 预热；filter 用包名 `@mini-schedule/brand`。
3. Happy(H1–H9) 走 chrome-devtools UI；Edge/并发/级联/越权/权限 走 API 直连（`curl` + 登录拿 token）+ psql 实查。
4. 并发场景（E2/E3/C3）用并行 `curl` 或小脚本触发同时请求，验行锁+unique 兜底。
5. DB 单测用真实 Postgres（`newMigratedTestDB` 自动建库跑 migration 至 000012，无 PG 时 skip）。
6. 回归结论汇入验收报告，含 teardown（清测试 booking/hold/txn/取消的场次）+ 复跑要点。
