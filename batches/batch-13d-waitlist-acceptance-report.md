# Batch 13d 候补（Waitlist）e2e 验收报告

> 执行时间：2026-06-23 · 后端 dev HEAD=d346d98（重建重启 :8081）· web dev HEAD=87925ee（rm .next 重启 :3002）· dev DB v12（零 migration）
> Happy 走 chrome-devtools UI；Edge/转正落库/级联/越权走 API + psql 实查。
> 结论：**全部必验场景 PASS**。0 阻断缺陷。3 项前端 polish 级小瑕疵（不影响功能与数据正确性，已列末尾）。

## 测试账号 / 数据
- owner `18816820405/admin123`（user 16 李四，all_brand）
- course_operator `13900139777/admin123`（user 23，仅 booking.view）
- 越权用户 `13900130033/admin123`（user 28，assigned_locations=[loc17]，有 create_assisted）
- 场次：course 4 晨间瑜伽 @ loc1 讯美广场，新建 cap=1 满员场次 62–70 + cap=2 未满场次 71
- 学员：10/11/12/13（active 次卡/会员卡）；9（active 但权益 frozen，无可用）；2（soft-deleted，作 LEARNER_NOT_FOUND 反例）
- policy：brand-default allow_waitlist=true / limit=0；session 63 override allow_waitlist=false；session 64 override limit=2

---

## Happy Path（UI + psql）

| # | 场景 | 结果 | psql 真值 |
|---|---|---|---|
| H1 | 满员场次(62)代预约→「场次已满」→「加入候补」 | ✅ PASS | waitlist_entries 新增 entry14 L11 pos1 waiting operated_by=16 |
| H2 | 再加 L12/L13 → position 递增；drawer 按 position | ✅ PASS | pos 1/2/3 = L11/L12/L13 waiting；drawer 列表顺序一致 |
| H3 | 取消 booking20 腾位 → drawer 转正队首(auto) | ✅ PASS | entry14→promoted promoted_booking_id=34；booking34 source=waitlist_promotion/booked；booked_count 0→1；hold19 held credits=1；ent9 rem 9→8 locked 1→2 |
| H4 | 转正改「无权益占位」+原因（session67 队首 L10） | ✅ PASS | entry3→promoted booking35；requires_entitlement_fix=**true**，no_entitlement_reason 落库，**无 hold**，booked_count=1 |
| H5 | drawer「跳过」填原因 | ✅ PASS | entry15→skipped，skipped_reason="H5 跳过测试-无可用权益" |
| H6 | drawer「取消」 | ✅ PASS | entry16→cancelled |

drawer 头部空位/满员提示正确切换：满员=「已满（需有人取消才能转正）」+转正 disabled；有空位=「有空位，可转正队首」+转正 enabled。

---

## Edge（API + psql）

| # | 场景 | 期望 | 实际 |
|---|---|---|---|
| E1 | allow_waitlist=false(63) 加入 | WAITLIST_NOT_ALLOWED 409 | ✅ 409 WAITLIST_NOT_ALLOWED |
| E2 | 未满场次(71) 加入 | WAITLIST_SESSION_NOT_FULL 409 | ✅ 409 WAITLIST_SESSION_NOT_FULL |
| E3 | limit=2(64) 加第3人 | WAITLIST_FULL 409 | ✅ pos1/2 OK，第3人 409 WAITLIST_FULL |
| E3b | limit=0(67) 加3人 | 不限 | ✅ pos 1/2/3 全 OK |
| E4 | 同学员重复候补(65) | WAITLIST_DUPLICATE 409 | ✅ 409 WAITLIST_DUPLICATE |
| E5 | 已有 active booking 再候补(62 L10) | BOOKING_DUPLICATE 409 | ✅ 409 BOOKING_DUPLICATE |
| E6 | 无空位转正(66) | SESSION_FULL 409 | ✅ 409；entry 仍 waiting，无新 booking |
| E7 | 转正 cancelled entry(6) | WAITLIST_NOT_PROMOTABLE 409 | ✅ 409 WAITLIST_NOT_PROMOTABLE |
| E8 | 无可用权益 auto 转正(68 L9) | ENTITLEMENT_NONE_AVAILABLE 409 | ✅ 409；entry 仍 waiting |
| E8b | 同 entry 改 none 占位转正 | 成功 | ✅ promoted booking31 requires_entitlement_fix=true reason 落库 无 hold |
| E9 | cancel 后重候补(65 L11) | 成功（旧 cancelled 保留） | ✅ 旧 entry6 cancelled + 新 entry7 waiting 共存（partial unique 放行） |
| E10 | 并发 2 学员加入满员(69) | position 不撞 | ✅ 并行 curl → pos 1/2 distinct |

---

## 场次取消级联（psql 实查）

| # | 场景 | 结果 |
|---|---|---|
| C1 | 场次(70) active 候补×2 + 真实 booking+hold → 取消 | ✅ session booked=0/cancelled；booking32 cancelled cancel_source=session_cancelled；hold17 released；ent14 rem 6→7 locked 3→2；候补 entry12/13→cancelled；**log metadata cascaded_waitlist=2 cascaded_bookings=1 cascaded_booking_ids=[32]** |
| C2 | 场次(69) promoted 候补(含 hold) + active 候补 → 取消 | ✅ promoted entry10 **不变**（终态）；其 booking33→session_cancelled + hold18 released（走 13c 级联）；active entry11→cancelled；metadata cascaded_waitlist=1（仅活跃，promoted 不计）cascaded_bookings=1[33] |

---

## 权限 + data_scope

| # | 场景 | 结果 |
|---|---|---|
| P1 | course_operator(view-only) | ✅ 查看名单 200（3 entries）；promote/skip 403 missing booking.create_assisted；cancel 403 missing booking.cancel；join 403 |
| P2 | 越权(loc17 用户操作 loc1 场次67) | ✅ join→SESSION_NOT_FOUND 404；promote→WAITLIST_ENTRY_NOT_FOUND 404；list→SESSION_NOT_FOUND 404（无存在性泄漏） |
| P3 | 前端无 create_assisted gating | ✅ drawer 内 转正/跳过/取消 全 disabled；/bookings「代预约」「代取消」disabled（无法进入加入候补流程） |

---

## 13c 回归（placeBooking 重构后）

`go test -count=1` 真实 Postgres，全绿：
- TestBooking_*（13 项：CreateAuto/Manual/NonePlaceholder/SessionFull/Duplicate/Cancel/AutoNoneAvailable/Scope/Frequency/Window/DataScope/Policy…）✅
- TestSessionCancel_CascadesBookings / AlwaysReleasesIgnoringPolicy / NoBookings ✅
- TestSessionCancel_**CascadesWaitlist** ✅（13d 新增级联）
- TestWaitlist_JoinAndList / JoinEdges / PromoteFreesSlot / SkipCancelRejoin ✅

→ promote 抽取共享 `placeBooking` 未改变 13c Create 行为。

---

## 小瑕疵（polish 级，非阻断）—— 已处理 ✅

1. **[已修复] drawer 头部容量/转正门控在抽屉内转正后不刷新**
   - 根因：`/schedule` 把场次 `booked_count/capacity` 作为打开抽屉时的**冻结快照**传给 WaitlistDrawer，`isFull` 由该静态快照算；抽屉内转正后 `brand-class-sessions` 已失效但快照不更新。
   - 修复：schedule 页改为只存 `waitlistRef`，用 `useMemo` 从最新 `items` **实时派生** session 容量（列表无此场次时回退快照）。`apps/brand/app/(protected)/schedule/page.tsx`。
   - 复验：抽屉内转正 B 后，头部即时由「容量 0/1 · 有空位」→「容量 1/1 · 已满（需有人取消才能转正）」，剩余 C 的「转正」自动 disabled。✅

2. **[已修复] /schedule 场次行「候补」无人数徽标 (N)**
   - 后端：`class_sessions` 列表 baseQuery 增相关子查询统计活跃候补，新增 `waitlist_count`（domain Session + sessionRow + toSessionDomain；GET 详情同带）。`classsession.go` / `class_session_repository.go`。
   - 前端：行内「候补」改为「候补 (N)」（N>0 时显示）；`ClassSessionListItem` 加 `waitlist_count`；join/skip/cancel 失效集追加 `brand-class-sessions`/`brand-class-session` 使徽标随候补增减实时刷新。`types/src/index.ts` / `api/src/waitlist.ts` / `schedule/page.tsx`。
   - 复验：API 实查 session 64/67 `waitlist_count=2`、62/71=0；UI 行显示「候补 (2)」/「候补」。✅

3. **[确认已实现，无需改] 禁用按钮 Hint**
   - 复查代码：drawer 内 转正/跳过/取消 已用 `<Hint content={DENIED}>` 包裹（Radix tooltip，hover 显示「权限不足，请联系管理员」；转正满员时显示「场次已满，需先有人取消」）。先前误判系 e2e 仅检查 `title` 属性、未触发 hover tooltip 所致。`waitlist-drawer.tsx`。

### 修复后静态验证
- `go build ./...` ✅；`go test ./internal/infrastructure/persistence/...`（含 waitlist_count 子查询）全绿 ✅
- `pnpm --filter @mini-schedule/brand lint` 0 errors ✅；`build` 成功（/schedule 13.1 kB）✅

## 备注
- 本轮在 dev DB 留下测试数据：class_sessions 62–71、对应 bookings/waitlist_entries/holds、session 63/64 的 policy override。dev 环境可接受，如需清理可按 session_id 删。
- 控制台仅 1 条 Radix 嵌套弹窗 aria-hidden a11y warning（PromoteDialog 叠在 drawer 上），无 JS error。
