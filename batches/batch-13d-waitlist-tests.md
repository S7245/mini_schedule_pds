# Batch 13d 测试场景 —— 候补（加入 / 手动转正 / 跳过取消 / 场次取消级联）

> 契约：[batch-13d-waitlist.md](batch-13d-waitlist.md)。混合执行（镜像 13c）：happy 走 UI；满员/转正落库/级联/越权走 API + **psql 实查**。
> **派生态铁律**：转正(建 booking+hold+扣额)、级联取消候补 必 psql 实查 DB 真值。

## 前置数据
- owner `18816820405 / admin123`（fast-path）；course_operator `13900139777`（仅 booking.view → 转正/跳过/取消 403）
- 门店 讯美广场 loc1；**capacity 小的 scheduled 场次**（如 cap=1，便于满员）+ 测试学员 ×3 + active 权益（次卡/会员卡）
- effective policy：allow_waitlist=true、waitlist_limit 设 2（测 WAITLIST_FULL）/0（测不限）

## Happy Path（UI 优先）

| # | 步骤 | 预期 |
|---|---|---|
| H1 | 满员场次代预约 → SESSION_FULL → 弹窗显示「加入候补」→ 点 | 候补成功；场次行「候补(1)」 |
| H2 | 再加 2 个学员到候补 | position 递增 1/2/3；drawer 名单按 position |
| H3 | 取消该场次的一个 booking（腾位）→ drawer 点队首「转正」→ 自动选权益 → 确认 | 转正成功。**psql**：`bookings`(source=waitlist_promotion/booked)、`booked_count` 回填、`entitlement_holds`(held)、`learner_entitlements` 扣额、`waitlist_entries`(promoted + promoted_booking_id=新 booking) |
| H4 | 队首转正改「手动指定」/「无权益占位」 | 占位：requires_entitlement_fix=true 无 hold |
| H5 | drawer 对某候补点「跳过」填原因 | entry → skipped + reason |
| H6 | drawer 对某候补点「取消」 | entry → cancelled |

## Edge（API + psql）

| # | 场景 | 预期 |
|---|---|---|
| E1 | allow_waitlist=false 加入候补 | `WAITLIST_NOT_ALLOWED` 409 |
| E2 | 未满场次加入候补 | `WAITLIST_SESSION_NOT_FULL` 409 |
| E3 | 活跃候补达 waitlist_limit(=2) 再加 | `WAITLIST_FULL` 409；limit=0 时不限可继续加 |
| E4 | 同学员对同场次重复候补 | `WAITLIST_DUPLICATE` 409（partial unique） |
| E5 | 学员已有该场次 active booking 再候补 | `BOOKING_DUPLICATE` 409 |
| E6 | 无空位(booked=capacity)时转正 | `SESSION_FULL` 409；**psql** 候补仍 waiting、无新 booking |
| E7 | 转正已 promoted/cancelled/skipped 的 entry | `WAITLIST_NOT_PROMOTABLE` 409 |
| E8 | 转正队首无权益 + auto 模式 | `ENTITLEMENT_NONE_AVAILABLE` 409（员工改 none 占位或跳过） |
| E9 | 取消后重新候补同场次（partial unique 放行新行） | 成功，新 entry（旧 cancelled 保留） |
| E10 | 并发：2 学员同时加入满员场次（位置分配） | 串行化（session 行锁）→ position 不撞（partial unique session,position active 兜底） |

## 场次取消级联（psql 实查）

| # | 场景 | 预期 |
|---|---|---|
| C1 | 场次有 active 候补 + active booking → 取消场次 | **psql**：活跃候补全 `cancelled`、booking 全 cancelled(session_cancelled,13c)+holds released+booked_count=0；session_cancelled log metadata 含 cascaded_waitlist 计数 |
| C2 | 已 promoted 的候补（其 booking）→ 场次取消时该 booking 走 13c 级联 cancel | promoted entry 不变（终态），其 booking 被 13c 级联取消 |

## 权限 + data_scope

| # | 场景 | 预期 |
|---|---|---|
| P1 | course_operator 转正/跳过/取消候补 | 403；查看名单 200 |
| P2 | 越权（非 assigned_locations 场次）加入/转正候补 | 404（SESSION_NOT_FOUND / WAITLIST_ENTRY_NOT_FOUND） |
| P3 | 前端：无 create_assisted 时「加入候补」「转正」disabled + Hint | 按钮禁用 |

## 13c 回归（placeBooking 重构后必复跑）
- 13c 全部 DB 单测（TestBooking_* / TestSessionCancel_*）仍绿——promote 抽 `placeBooking` 不得改变 13c Create 行为。

## psql 实查清单
```sql
SELECT id,position,status,promoted_booking_id,skipped_reason FROM waitlist_entries WHERE class_session_id=? ORDER BY position;
SELECT id,source,status FROM bookings WHERE class_session_id=? AND source='waitlist_promotion';
SELECT id,status,booked_count FROM class_sessions WHERE id=?;
SELECT status,remaining_credits,locked_credits FROM learner_entitlements WHERE id=?;
```

## 执行方式
后端 e2e 前重建重启(:8081)；前端 rm .next 重启(:3002)+curl 预热；Happy 走 chrome-devtools，Edge/转正落库/级联/越权走 API+psql。并发(E10)用并行 curl。DB 单测 `newMigratedTestDB`（v12 无新 migration）。
