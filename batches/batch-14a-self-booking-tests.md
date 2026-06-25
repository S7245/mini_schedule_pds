# Batch 14a 测试场景 —— C 端学员自助预约核心环（桥接 + 课程表 + 下单 + 我的预约 + 取消）

> 契约：[batch-14a-self-booking.md](batch-14a-self-booking.md)。**C 端**：后端 api-app :8082、前端 app :3003、filter `@mini-schedule/app`。
> 混合执行（镜像 13c–13e）：happy 走 app UI；桥接落库 / source+assisted_by NULL / ownership / 重叠 / 配额 走 API + **psql 实查**。
> **桥接铁律**：C 端 openid→profile 必须稳定映射**一个** `brand_learner_profile`，且与 brand 侧操作的是同一行——所有桥接/下单落库必 psql 实查 DB 真值，不能只看前端。

## 前置数据（brand 21）
- **C 端登录**：app :3003 `(auth)/login` 输 `brand_id`（默认 `NEXT_PUBLIC_DEFAULT_BRAND_ID`=brand21）+ 登录码 `code` → 后端 dev openid `"dev_"+code`（同 code 复登 = 同一身份）。用 `code=alice` / `code=bob` 造两个独立学员身份。
- **brand 侧准备**（owner `18816820405 / admin123`）：① /schedule 建 **scheduled 场次**（门店 讯美广场 loc1）：S1 09:00–10:00、S2 09:30–10:30（与 S1 时段重叠）、S3 次日、S4 容量=1（测满员）。② C 端学员**先登录建出 profile** 后，brand 侧 /learners 找到该 profile → 权益 Tab 发放：次卡（count-based remaining=10）+ 不限次会员卡（remaining=NULL）。
- policy：默认（窗口不限、取消截止 0、release_on_cancel=true）；另备一套 `cancel_deadline_minutes` 较大的场次覆盖测 E 截止。

## Happy Path（app UI 优先）

| # | 步骤 | 预期 |
|---|---|---|
| H1 | app :3003 `code=alice` 登录 | 进入学员首页；token 带 `profile_id`。**psql**：`learner_identities`(wechat_open_id=`dev_alice`, phone NULL) 1 行 + `brand_learner_profiles`(brand21+该 identity) 1 行 |
| H2 | 同 `code=alice` 退出再登录 | 仍同一 profile（幂等）。**psql**：identity/profile 各仍 1 行（无重复） |
| H3 | 课程表页 | 列 brand21 scheduled + 未来场次（S1/S2/S3/S4），每行 课程/时间/门店/`剩余 N`，含「预约」按钮 |
| H4 | 对 S1 点「预约」（alice 有次卡+会员卡） | 预约成功，我的预约出现 S1。**psql**：`bookings`(source=`learner_self_service`, **assisted_by NULL**, profile=alice, status=booked) + `entitlement_holds`(held, 自动选权益按 §5.7) + `learner_entitlements` locked++/remaining--（次卡）+ `entitlement_transactions`(hold, operated_by NULL) + `operation_logs`(actor_type=`learner`, actor_id=alice profile) |
| H5 | 我的预约页 | 列 alice 本人预约（S1 booked）+ 状态/课程/时间/权益名；**只本人**（见 E ownership） |
| H6 | 对 S1 预约点「取消」填原因确认 | booking→cancelled。**psql**：`cancel_source=`**`learner`**、hold→released、次卡 remaining++/locked--、txn(release)、booked_count-- |
| H7 | 取消后对同 S1 再「预约」 | 成功（partial unique 允许，新 booking 行）；旧 cancelled 行保留 |
| H8 | 预约前（详情/确认）`usable-entitlements` 预览 | 显示「将使用 X 权益」（§5.7 auto，[0].auto_selected=true）；无可用时显示「无可用权益」 |

## 桥接锚点（**本批验收重心** · API + psql + brand 侧交叉验证）

| # | 场景 | 预期 |
|---|---|---|
| B1 | C 端 alice 自助预约 S1 后，**brand 侧** owner 登录 /bookings 查 | 见该 booking：`source=learner_self_service`、`assisted_by` 空、`brand_learner_profile` = C 端 alice 的同一 profile（证同一 profile） |
| B2 | brand 侧给 alice profile 发的权益，C 端 `usable-entitlements`/预约能选中 | C 端可见并可用（同一 profile） |
| B3 | **psql 桥接链路**：`SELECT li.wechat_open_id, p.id FROM learner_identities li JOIN brand_learner_profiles p ON p.learner_identity_id=li.id WHERE li.wechat_open_id='dev_alice'` | 恰 1 行；该 `p.id` 即 B1 booking 的 profile、H4 hold 的 profile（全链一致） |
| B4 | brand 侧先按手机号建档学员 X（openid `manual:138…`）→ C 端另一 code 登录 | 是**不同 profile**（v1 不合并）——验证「先 C 端登录建 P，brand 再发权益」是正确链路（反向不通） |

## Edge（API + psql）

| # | 场景 | 预期 |
|---|---|---|
| E1 | 满员场次 S4（capacity=1，已 1 人约）再自助预约 | `SESSION_FULL` 409（14a 提示「已满」，引导候补留 14b） |
| E2 | **跨场次时间重叠**：alice 已约 S1（09:00–10:00），再约 S2（09:30–10:30） | `BOOKING_TIME_CONFLICT` 409（§22.1）；psql 无 S2 booking |
| E3 | 无可用权益学员自助预约 | `ENTITLEMENT_NONE_AVAILABLE` 409（引导联系机构）；前端明确文案 |
| E4 | 超预约窗口（场次 policy 限 book_ahead_max，当前不在窗口） | `BOOKING_WINDOW_CLOSED` 409 |
| E5 | 超取消截止（场次 cancel_deadline 已过）自助取消 | `BOOKING_CANCEL_DEADLINE_PASSED` 409 |
| E6 | 同场次重复预约（已有 active booking） | `BOOKING_DUPLICATE` 409 |
| E7 | 频次超限（policy/产品 日/周/月/并发） | `BOOKING_FREQUENCY_EXCEEDED` 409 + which 维度 |
| E8 | 冻结学员（status≠active）自助预约 | `LEARNER_NOT_BOOKABLE` 409 |
| E9 | 非 scheduled 场次（draft/completed/cancelled）自助预约 | `SESSION_NOT_BOOKABLE` 409 |
| E10 | **ownership**：alice 取消 bob 的 booking id | `BOOKING_NOT_FOUND` 404（不泄漏存在性，非 403） |
| E11 | **ownership**：`GET /bookings` 不接受前端传 learner 参数 | 恒返 token profile 自己的；alice 看不到 bob 的预约 |
| E12 | **配额**：brand 满 `max_learners` 时新 code 首登建 profile | `QUOTA_EXCEEDED` 409 登录失败（honor 硬限，返干净错误，非 500） |
| E13 | 旧 token（无 profile_id，桥接前签发）调业务端点 | 401 需重新登录 |
| E14 | 并发：alice 两请求同时约 S4（capacity=1，0 已约） | 串行化（session 行锁 + booked_count + partial unique）→ 仅一成功，另一 `SESSION_FULL` 或 `BOOKING_DUPLICATE`；psql booked_count=1，booking 1 行 |

## 回归（placeBooking 参数化 assisted_by 后**必复跑**）
- 后端：13c `TestBooking_*`/`TestSessionCancel_*` + 13d `TestWaitlist_*` + 13e `TestAttend_*`/`TestEndSession_*`/`TestNoShow_*` **全绿**——`assistedBy *int64` 改动不得改变 staff 路径（staff 仍 assisted_by=actor）。
- `go build ./... && go test ./...` 全包绿；prod `pnpm --filter @mini-schedule/app build` exit 0。
- TX-3 场次取消级联：含 learner_self_service booking 的 scheduled 场次被 brand 取消 → 该 C 端 booking 一并 cancelled + hold released（按 status 不按 source，已覆盖，复验）。

## psql 实查清单
```sql
-- 桥接
SELECT id, wechat_open_id, phone, nickname FROM learner_identities WHERE wechat_open_id LIKE 'dev_%';
SELECT id, brand_id, learner_identity_id, primary_location_id, status FROM brand_learner_profiles WHERE brand_id=? ;
-- 下单/取消
SELECT id, source, status, assisted_by, brand_learner_profile_id, cancel_source FROM bookings WHERE brand_learner_profile_id=? ORDER BY id;
SELECT id, status, booked_count FROM class_sessions WHERE id=?;
SELECT id, status, held_at, released_at FROM entitlement_holds WHERE booking_id=?;
SELECT status, total_credits, remaining_credits, locked_credits FROM learner_entitlements WHERE id=?;
SELECT action, delta_credits, operated_by, hold_id FROM entitlement_transactions WHERE booking_id=? ORDER BY id;
-- 审计 actor=learner
SELECT actor_type, actor_id, action FROM operation_logs WHERE action IN ('learner_self_registered','booking_created','booking_cancelled') ORDER BY id DESC LIMIT 10;
```

## DB 单测（newMigratedTestDB，v12 无新 migration）
- `TestBridge_FindOrCreateProfileByOpenID`：新建 identity+profile / 复登幂等(同 profile) / 软删后重建 / 并发回查 / 满配额 QUOTA_EXCEEDED。
- `TestLearnerBooking_Create`：source=learner_self_service + assisted_by NULL + auto 选权益 + hold/txn(operated_by NULL) + audit(learner)。
- `TestLearnerBooking_TimeConflict`：S1 已约，约重叠 S2 → BOOKING_TIME_CONFLICT；不重叠 S3 → 成功。
- `TestLearnerBooking_CancelOwnership`：本人取消成功(cancel_source=learner) / 取消他人 → BOOKING_NOT_FOUND。
- `TestLearnerBooking_FailureCodes`：满员/无权益/窗口/频次/冻结/非 scheduled 复用 13c 校验。
- `TestClassSession_ListBookable`：仅 brand+scheduled+未来；draft/completed/过期不返。
- `TestPlaceBooking_AssistedByParam`：staff 路径 assisted_by=actor（回归）/ learner 路径 NULL。
- `TestActorLearner_Valid`：audit.IsValidActorType(ActorLearner)=true。
- `TestTokenPayload_ProfileIDRoundTrip`：generate→parse profile_id 往返。

## 执行方式
后端 e2e 前 `go build -o /tmp/api-app ./cmd/api-app` 重建重启（:8082，旧二进制掩盖改动）；前端 `rm -rf apps/app/.next && pnpm --filter @mini-schedule/app dev`（:3003）+ curl 预热；filter 用 `@mini-schedule/app`。Happy 走 app UI（chrome-devtools），桥接锚点 B/Edge/落库/ownership/重叠/配额走 API + **psql 实查**。并发（E14）用并行 curl。DB 单测 `newMigratedTestDB`（v12，**无新 migration**）。除 e2e 外必跑 prod `pnpm --filter @mini-schedule/app build`。
