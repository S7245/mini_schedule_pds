# Batch 13c 契约 —— 学员预约下单（Booking + EntitlementHold + 代取消 + 场次取消级联）

> Batch 13（学员预约闭环）第三子批，全闭环最硬的一批——**下单原子性**。
> 前置：13a 学员档案 ✅、13b 权益产品+发放（lock 模型）✅ 均已 merge main。
> 范围：brand 后台 `staff_assisted` 下单 + 代取消 + 场次取消级联 + 预约规则解析。
> 不做：候补（13d）、签到/consume/no_show（13e）、C 端自助预约（Batch 14）、per-location/场次 override 策略 CRUD。

---

## 0. grill 拍板结论（2026-06-22 会话内）

| # | 决策点 | 结论 |
|---|---|---|
| 1 | brand_booking_policies 配置面 | **仅 brand-default 单行 GET+PUT upsert**；复用 `schedule.view`(读)/`schedule.manage`(写)，零新权限码；per-location + 场次 override 的 CRUD 延后，解析层仍读取已存在的覆盖行 |
| 2 | 权益选择（§5.7） | **自动选择 + 允许员工手动指定**；手动指定仍全量校验（active/scope/频次/余额），非法即报错，**不静默回退自动** |
| 3 | 满员处理 | **返 `SESSION_FULL`，13c 完全不碰 waitlist**；候补入口+转正整体留 13d |
| 4 | 取消 hold 边界 | **默认 release**（退名额+退权益）；`release_on_cancel=false` → forfeit `consumed`；**场次取消恒退**（忽略 release_on_cancel）；no_show 扣课留 13e |
| 5 | data_scope 越权语义 | **404 不泄漏存在性**（镜像 13a/13b/classsession 既有约定，写进错误码列） |

**已锁定实现默认（无需再问）：**
- partial-unique 集合 = `WHERE status <> 'cancelled'`（镜像 13a `deleted_at IS NULL`）。
- 不限次会员卡（`remaining_credits IS NULL`）hold：建 hold（credits=1）、**不扣 remaining**、`locked_credits++`（仅占用计数）、txn `delta=0` `balance_after=NULL`。
- `cancel_deadline_minutes` 对**员工代取消同样生效**（员工绕过截止时间 → 转 FR）。
- 手动指定权益校验失败 → 报错，不回退自动选择。
- 频次「叠加取最严」= `entitlement_product(daily/weekly/monthly/concurrent)` ∪ `policy(daily/weekly/concurrent)`，**月限独家来自权益产品**（`brand_booking_policies` 无 monthly 列）。

---

## 1. Schema 现实（migration 000003 已核实，本批不动表结构，仅改索引）

| 表 | 关键列 / 约束 | 13c 用法 |
|---|---|---|
| `bookings` | `source∈{learner_self_service,staff_assisted,waitlist_promotion}`、`status∈{booked,cancelled,attended,pending_no_show,no_show}`(默认 booked)、`cancel_source∈{learner,staff,session_cancelled,system}`、`requires_entitlement_fix`/`no_entitlement_reason`、CHECK`(fix=F OR reason NOT NULL)`、`assisted_by`/`cancelled_by`/`cancelled_at`、**`unique(class_session_id, brand_learner_profile_id)` 全量** | 本批写 `staff_assisted`+`booked`/`cancelled`；唯一索引改 partial（见 §5） |
| `entitlement_holds` | `status∈{held,released,consumed}`(默认 held)、`credits≥1`(默认1)、`unique(booking_id)`、`learner_entitlement_id` ON DELETE **RESTRICT** | 一 booking 一 hold；本批 `held→released`(+forfeit `→consumed`) |
| `brand_booking_policies` | `book_ahead_min_minutes`(默认0)/`book_ahead_max_minutes`(NULL=不限)、`cancel_deadline_minutes`(默认0)、`release_on_cancel`(默认T)、`no_show_consumes_entitlement`(默认F)、`daily/weekly/concurrent_booking_limit`(NULL=不限)、`allow_waitlist`(默认T)、`waitlist_limit`(默认0)；**两条 partial unique 已建**(default `WHERE location_id IS NULL` + location 级) ｜⚠️**无 monthly 列** | GET/PUT 仅操作 default 行(location_id IS NULL) |
| `class_session_policy_overrides` | 全字段 nullable=继承；**有 `allow_cancel`**（brand 层无此字段）、`cancel_deadline_minutes`/`release_on_cancel`/`no_show_consumes`/`allow_waitlist`/`waitlist_limit`；**无 book_ahead**；`unique(class_session_id)` | 解析层读取（稀疏覆盖）；本批不做其 CRUD |
| `class_sessions` | `capacity>0 AND 0≤booked_count≤capacity`(CHECK 兜底超卖)、`status∈{draft,scheduled,in_progress,completed,cancelled}` | 行锁 + 此 CHECK = 抢名额兜底；`booked_count` 由本批维护 |
| `learner_entitlements` | `total/remaining_credits` 可 NULL(会员卡=不限次)、`locked/consumed_credits NOT NULL≥0`、CHECK 非负(23514 兜底) | hold 锁额/release 退额，复用 13b 锁骨架 |
| `entitlement_transactions` | `action∈{grant,hold,release,consume,no_show_consume,manual_adjust}`、`delta_credits NOT NULL`、`balance_after` 可 NULL、`booking_id`/`hold_id` 外键 | 本批写 `hold`(Δ-1) / `release`(Δ+1)；不限次 Δ=0 |

**权限现实（关键）：** `booking.view / create_assisted / cancel` 三码在 000003 已 seed，且角色映射**完整吻合 §21.1/§21.2**：
- `brand_owner / brand_admin / location_manager(店长) / receptionist(前台)` = view + create_assisted + cancel
- `course_operator / finance_support / instructor / location_assistant` = 仅 view

且这些映射是 000003 **base 自带**（非 13a/13b 后补于 000009/000011），存量 brand 建库即已 copy 进 brand_roles → **13c 无需权限 backfill**。`booking_policy.*` 码不存在，但决策 1 复用 `schedule.*` 故**不新增任何权限码**。

---

## 2. Booking 状态机（13c 仅实现实线，虚线留 13e）

```
              ┌──────── 代取消 TX-2 / 场次取消 TX-3 ────────┐
              ▼                                              │
 (下单)──▶ booked ───实───▶ cancelled  [终态]               │
              │                  ▲                           │
              │                  └ partial-unique 放行重约：cancelled 后可 INSERT 新行
              │  ┄┄13e┄┄▶ attended            [终态]
              │  ┄┄13e┄┄▶ pending_no_show ┄┄13e┄┄▶ no_show  [终态]

 Hold: (下单)─▶ held ─实─▶ released   (默认取消 / 场次取消恒退)
                   └──实─▶ consumed   (release_on_cancel=false 的 forfeit)
                   └┄13e┄▶ consumed   (到课正常消耗 / no_show 扣课)
```

---

## 3. 下单原子性 —— 三条 TX 边界（核心难点）

**统一锁序铁律：永远先 `SELECT FOR UPDATE` `class_sessions` 行，再锁 `learner_entitlements` 行**（三条 TX 一致 → 无死锁）。全靠**行锁 + unique + CHECK 兜底**，不做应用层「先查后插」。

### TX-1 代预约 `POST /brand/bookings`（单事务）
```
0. guardLocationInScope：session.location_id ∈ assigned_locations，否则 SESSION_NOT_FOUND(404)
1. SELECT FOR UPDATE class_sessions
   → 校 status='scheduled'（否则 SESSION_NOT_BOOKABLE 409）
   → 校预约窗口：now ∈ [starts_at - book_ahead_max, starts_at - book_ahead_min]（否则 BOOKING_WINDOW_CLOSED 409）
   → 校 booked_count < capacity（否则 SESSION_FULL 409）
2. 校学员属本 brand 且可预约（frozen/inactive → LEARNER_NOT_BOOKABLE 409；不存在 → LEARNER_NOT_FOUND 404）
3a. [entitlement_mode=auto|manual]
    解析 effective policy → 候选权益（active/未过期/scope(location+course)/频次叠加取最严/remaining>0或不限次）
    → auto: 按 §5.7 排序取第 1（无候选 → ENTITLEMENT_NONE_AVAILABLE 409）
    → manual: 用指定 id，仍全校验（不可用→ENTITLEMENT_NOT_USABLE 422 / scope 不符→ENTITLEMENT_SCOPE_MISMATCH 422 / 频次→BOOKING_FREQUENCY_EXCEEDED 409）
    SELECT FOR UPDATE learner_entitlements → 锁额（count 卡 remaining--；不限次跳过）+ locked++ + re-settle(SettleStatus)
    → booked_count++ → INSERT booking(staff_assisted, booked)
    → INSERT entitlement_holds(held, unique booking) → INSERT txn(hold, Δ-1/不限次Δ0, balance_after)
3b. [entitlement_mode=none]（仅 booking.create_assisted）
    必填 no_entitlement_reason（缺→ASSISTED_REASON_REQUIRED 422）
    → 仍校 booked_count<capacity（§5.8 不绕容量）→ booked_count++
    → INSERT booking(staff_assisted, booked, requires_entitlement_fix=true, no_entitlement_reason)
    → 不建 hold、不锁权益
4. audit.Write（action=booking_created；mode=none 时记「无权益代预约」§24.3 必审）
```
**并发兜底：** `class_sessions` CHECK(`booked_count≤capacity`, 23514) + `bookings` partial-unique(23505 重复→BOOKING_DUPLICATE 409) + `learner_entitlements` 非负 CHECK(23514) + `entitlement_holds` unique(23505)。约束名分流复用 `uniqueConstraint(err)`。

### TX-2 代取消 `POST /brand/bookings/:id/cancel`（单事务）
```
0. 解析 booking + guardLocationInScope（session.location ∈ scope，否则 BOOKING_NOT_FOUND 404）
1. SELECT FOR UPDATE class_sessions（该 booking 的场次，与 TX-1/TX-3 串行化）
2. SELECT FOR UPDATE bookings → 校 status='booked'（否则 BOOKING_NOT_CANCELLABLE 409）
3. 解析 effective policy → 校 allow_cancel(override，默认允许)（否则 BOOKING_CANCEL_NOT_ALLOWED 409）
   → 校 now ≤ starts_at - cancel_deadline_minutes（否则 BOOKING_CANCEL_DEADLINE_PASSED 409）
4. booking → cancelled（cancel_source='staff', cancelled_at, cancelled_by, cancel_reason）
5. booked_count--
6. hold 存在时：release_on_cancel=true → hold='released'+released_at，entitlement locked--/remaining++(count)/re-settle，txn(release, Δ+1)
                 release_on_cancel=false → hold='consumed'+consumed_at，locked--/consumed++，txn(consume, Δ0 占位)  [forfeit]
7. audit.Write(action=booking_cancelled)
```

### TX-3 场次取消级联（**回改已 merge-main 代码**）
扩 `internal/infrastructure/persistence/class_session_repository.go:238 Cancel` 现有事务：
```
当前：First(session)[无锁] → status=cancelled + cancel_reason → writeSessionLog
回改：① line 241 First 加 clause.Locking{Strength:"UPDATE"}（与 TX-1/TX-2 串行化）
      ② status=cancelled 后级联：
         SELECT bookings WHERE class_session_id=? AND status='booked'
         逐条 → cancelled(cancel_source='session_cancelled', cancelled_at, cancelled_by=actor)
         逐条 hold(held) → released + 退权益(locked--/remaining++/re-settle) + txn(release)  【恒退，忽略 release_on_cancel】
         booked_count=0
      ③ session_cancelled OperationLog 的 metadata 记 cascaded_booking_ids/count（不每 booking 单独 log，避免刷屏）
      ④ 候补失效留 13d（届时扩此级联取消 waitlist_entries）
```
> 影响面：`classsession` service/repository + `recurringschedule`（其取消是否级联场次？B12b 已定**非级联**，故循环排课取消不触发本级联，仅单场次 Cancel 触发）。回归 B11/12 的 session-cancel 测试需补级联断言。

---

## 4. API 接口契约

> 前缀 `/api/v1`。所有写操作：操作权限 + data_scope 双校验（§21.3）。越权资源一律 **404 不泄漏存在性**。

| 方法 | 路径 | 权限 | 请求字段 | 响应字段 |
|---|---|---|---|---|
| POST | `/brand/bookings` | `booking.create_assisted` | `class_session_id`, `brand_learner_profile_id`, `entitlement_mode∈{auto,manual,none}`, `learner_entitlement_id`(manual 必填), `no_entitlement_reason`(none 必填) | `booking{id,status,source,session{...},learner{...},hold{...}\|null,requires_entitlement_fix,...}` |
| POST | `/brand/bookings/:id/cancel` | `booking.cancel` | `reason?` | `booking{...更新后}` |
| GET | `/brand/bookings` | `booking.view` | `class_session_id?`, `location_id?`, `brand_learner_profile_id?`, `status?`, `requires_entitlement_fix?`, `page`, `page_size` | `rows[]`(见下「list DTO」), `total` |
| GET | `/brand/bookings/:id` | `booking.view` | — | `booking{...full + hold + entitlement + txn refs}` |
| GET | `/brand/bookings/usable-entitlements` | `booking.view` | `class_session_id`, `brand_learner_profile_id` | `entitlements[]{id,product_name,type,remaining,expires_at,scope_match,frequency_ok,auto_selected,reason}`（§5.7 优先级排序，首项 `auto_selected=true`） |
| GET | `/brand/booking-policy` | `schedule.view` | — | `policy{book_ahead_min_minutes,book_ahead_max_minutes,cancel_deadline_minutes,release_on_cancel,no_show_consumes_entitlement,daily_booking_limit,weekly_booking_limit,concurrent_booking_limit,allow_waitlist,waitlist_limit}`（无行时返 sensible 默认） |
| PUT | `/brand/booking-policy` | `schedule.manage` | 同上 policy 全字段 | `policy{...}`（upsert default 行 location_id IS NULL） |

**复用的既有读端点（代预约弹窗选择器，显式登记避免 ASSUMPTION 漏端点）：**
- 选场次 → 复用 `GET /brand/class-sessions`（B11/12，须返 `capacity`/`booked_count` 供满员显示；按 `status=scheduled` + scope 过滤）
- 选学员 → 复用 `GET /brand/learners`（13a）
- 预览/选权益 → **新增** `GET /brand/bookings/usable-entitlements`（上表）

**list DTO 铁律（13b F1 教训）：** `GET /brand/bookings` 每行必须内嵌 detail 同款字段（session 名/时间/门店、learner 名/手机、bound entitlement 名、hold 状态、requires_entitlement_fix），因为列表行会被复用为详情/取消弹窗初值。

**错误码（新增至 `pkg/errors/error.go`；标注 HTTP）：**

| 码 | HTTP | 触发 |
|---|---|---|
| `SESSION_NOT_BOOKABLE` | 409 | 场次非 scheduled（草稿/进行中/完成/取消） |
| `BOOKING_WINDOW_CLOSED` | 409 | 超出 book_ahead_min/max 窗口 |
| `SESSION_FULL` | 409 | booked_count = capacity |
| `BOOKING_DUPLICATE` | 409 | 该学员对该场次已有非 cancelled 预约（partial unique 23505） |
| `LEARNER_NOT_BOOKABLE` | 409 | 学员 frozen/inactive |
| `ENTITLEMENT_NONE_AVAILABLE` | 409 | auto 模式无可用权益 |
| `ENTITLEMENT_NOT_USABLE` | 422 | manual 指定权益非 active/已过期/已耗尽/frozen/cancelled |
| `ENTITLEMENT_SCOPE_MISMATCH` | 422 | manual 指定权益 location/course scope 不匹配该场次 |
| `BOOKING_FREQUENCY_EXCEEDED` | 409 | daily/weekly/monthly/concurrent 超限（Details 带 which/limit/current） |
| `ASSISTED_REASON_REQUIRED` | 422 | none 模式缺 no_entitlement_reason |
| `BOOKING_NOT_CANCELLABLE` | 409 | 取消时 status≠booked |
| `BOOKING_CANCEL_NOT_ALLOWED` | 409 | override allow_cancel=false |
| `BOOKING_CANCEL_DEADLINE_PASSED` | 409 | 超过 cancel_deadline |

复用已存在：`SESSION_NOT_FOUND`(404)、`LEARNER_NOT_FOUND`(404)、`BOOKING_NOT_FOUND`(404，含越权)、`QUOTA_EXCEEDED` 不涉及（booking 不占 subscription 额度）。

---

## 5. Migration 000012（dev DB 现 v11 → v12）

```sql
-- bookings：全量 unique → partial（仅非 cancelled），放行取消后重约 INSERT 新行
DROP INDEX IF EXISTS idx_bookings_session_learner_unique;
CREATE UNIQUE INDEX IF NOT EXISTS idx_bookings_session_learner_active
    ON bookings(class_session_id, brand_learner_profile_id)
    WHERE status <> 'cancelled';

-- waitlist_entries：同款 partial（为 13d 铺路，本批不写 waitlist，幂等无害）
DROP INDEX IF EXISTS idx_waitlist_entries_session_learner_unique;
CREATE UNIQUE INDEX IF NOT EXISTS idx_waitlist_entries_session_learner_active
    ON waitlist_entries(class_session_id, brand_learner_profile_id)
    WHERE status NOT IN ('cancelled', 'skipped');
```
- **无权限变更**（§1 已核实 booking.* 映射齐全 + 存量已 backfill）。落地时 `SELECT` 复核 brand21 `brand_role_permissions` 确含 booking.*；仅在发现缺口时才追加幂等 backfill（镜像 000011）。
- `down`：恢复两条全量 unique。

---

## 6. 前端页面模块

| 页面/模块 | 类型 | 关键字段/操作 |
|---|---|---|
| `/bookings` | 页面（新，mirror `/learners`） | DataTable + 筛选(场次/门店/学员/状态/待补权益) + 分页；「代预约」按钮 gate `booking.create_assisted`+Hint；行「代取消」gate `booking.cancel`；`requires_entitlement_fix` badge + 筛选（§20.11 待补权益队列） |
| BookingCreateDialog | 弹窗 | 学员选择(复用 learner 选择器) + 场次选择(复用 schedule 选择器，显示 capacity/booked) + 权益模式 radio(自动/手动指定/无权益占位)；auto→显示 usable-entitlements 首项预览；manual→下拉(usable-entitlements)；none→原因 textarea；提交错误 inline(SESSION_FULL/FREQUENCY/WINDOW/DUPLICATE…) |
| BookingCancelDialog | 弹窗 | ConfirmDialog(ReactNode 高亮场次/学员) + 可选原因；展示 DEADLINE_PASSED/NOT_ALLOWED |
| Booking 详情 | 抽屉/行展开 | 绑定权益 + hold 状态 + 来源/代预约人 + 取消入口 |
| 学员详情「预约」Tab | 内嵌 Tab | 替换 13a 占位：复用 `GET /bookings?brand_learner_profile_id=` 列学员预约 |
| `/booking-policy` | 页面（新） | brand-default 策略表单(book_ahead/cancel_deadline/release/no_show/daily/weekly/concurrent/waitlist)；保存 gate `schedule.manage`，读 gate `schedule.view` |

**api client：** 新增 `packages/api/src/bookings.ts` + `booking-policy.ts` → **必须登记 `packages/api/package.json` exports**（web 教训）。
**常量：** `PERMISSIONS.BOOKING_VIEW/CREATE_ASSISTED/CANCEL`（+ 复用 SCHEDULE_*）；新错误码常量。
**导航：** 加 `/bookings`(gate booking.view) + `/booking-policy`(gate schedule.view)，接 `NAV_HREF_PERMISSIONS`。
**跨查询失效（web 教训）：** booking mutation 改了 session.booked_count + entitlement.remaining/locked → 失效 `['brand-bookings']` + `['brand-class-sessions']` + `['brand-learner']`(预约 Tab) + `['learner-entitlements']`；详情页删/取消导航回列表用 `refetchType:'all'`。

### 前端实现约束
- 复用 `/web/apps/brand` 现有组件与设计风格，不引入新 UI 库；mirror `/learners`/`/resources`/`/entitlement-products`。
- 编辑/选择弹窗的外键选项并入「当前已选但已停用/归档」项（13a/13b 同源教训）。
- `data-testid` 钩子按既有命名（`booking-create-button`/`booking-row`/`booking-field-*`/`booking-submit` 等）供 e2e。

### Wireframe
```
/bookings ──────────────────────────────────────────────
[筛选: 场次▾ 门店▾ 学员🔍 状态▾ ☐仅待补权益]      [+ 代预约]
┌──────┬────────┬──────┬────────┬──────┬──────────┬────────┐
│学员  │场次    │门店  │权益    │状态  │待补权益  │操作    │
├──────┼────────┼──────┼────────┼──────┼──────────┼────────┤
│张三  │晨瑜6/25│讯美  │次卡-9  │已预约│   —      │[代取消]│
│李四  │晨瑜6/25│讯美  │  —     │已预约│ ⚠待补   │[代取消]│
└──────┴────────┴──────┴────────┴──────┴──────────┴────────┘

BookingCreateDialog ────────────────
学员:   [李四 🔍▾]
场次:   [晨间瑜伽 06/25 09:00 (讯美) 8/12 ▾]
权益:   ( )自动选择 → 将使用「次卡-剩9次(7/30到期)」
        ( )手动指定 → [下拉: 次卡-9 / 会员卡-不限次]
        (•)无权益占位 → 原因[____________]
                                   [取消] [确认预约]

/booking-policy ─────────────────────
品牌默认预约规则
 最早提前预约(min)[__] 最晚提前(max)[__]
 最晚取消(min前)  [__] 取消退权益 ☑
 爽约扣课 ☐
 每日上限[__] 每周上限[__] 同时未完成上限[__]
 候补 ☑  候补上限[__]
                          [保存 (需 schedule.manage)]
```

---

## 7. 复用映射（13a/13b/B11-12 已验证骨架，直接照搬）

| 需求 | 复用 |
|---|---|
| 新 booking 域流水线 | domain(实体+Status+IsValid*+Repository) → service(`require(code)`+`checker==nil` bypass + data_scope `scopeFilterIDs`/`guardLocationInScope`) → persistence(tx+`audit.Write`) → brand handler → wire `go generate`(NewHandler 加参数, router_test 补 nil)，镜像 `classsession`/`learner` |
| 锁额骨架 | `clause.Locking{Strength:"UPDATE"}` + 非负校验（`entitlement_repository.go:533/575`） |
| 状态结算 | `entitlement.SettleStatus(current, expiresAt, totalCredits, remainingCredits *int, now)`（`entitlement.go:88`）—— hold/release 改余额后 re-settle |
| 约束名分流 | `uniqueConstraint(err)(name,ok)`（`learner_repository.go`）→ 23505 按约束名分 BOOKING_DUPLICATE |
| data_scope | session.location_id ∈ assigned_locations，镜像 `classsession.Service`；越权 404 |
| 场次取消回改点 | `class_session_repository.go:238 Cancel`（加锁 + 级联） |
| 学员删除守卫 | `countLearnerActiveReferences`（13a 已含未结束 bookings）→ 本批落地后自动生效，无需回改 13a |

---

## 8. 验收闭环（数据链可在测试库凑齐）

前置数据：① scheduled 场次（`/schedule` 排课，B11/12）② 测试学员（`/learners`，13a）③ 给学员发的 active 权益（`/entitlement-products` + 学员权益 Tab，13b）。门店「讯美广场」loc1 + 晨间瑜伽 course4。

| 关键验收 | 手段 |
|---|---|
| Happy：代预约(自动选权益)→列表→代取消→重约 | UI |
| 满员 SESSION_FULL / 窗口 / 频次 / 重复 DUPLICATE | API 直连（并发抢名额 race） |
| 无权益占位（requires_entitlement_fix + 待补队列 + 不绕容量） | UI + API |
| 手动指定权益 + 非法报错不回退 | UI + API |
| 取消 release 落库（locked--/remaining++/re-settle/txn） | **psql 实查 DB 真值**（13b settle 教训） |
| 场次取消级联（所有 booking cancelled + holds released + booked_count=0） | **psql 实查** |
| 越权代预约/代取消 → 404 | API（course_operator `13900139777` / 非 scope 门店员工） |
| 权限门：course_operator 代预约/代取消 403、receptionist 可 | API + UI |

> **验收铁律**：后端 e2e 前 `go build -o /tmp/api-brand ./cmd/api-brand` 重建重启(:8081)；前端 `rm -rf .next` 重启(:3002)+curl 预热；filter 用包名 `@mini-schedule/brand`；除 e2e 外跑 prod `pnpm build`。派生态(hold/级联)验收**须 psql 实查**。

---

## 9. 出范围 / 前向依赖

**本批不做：** 候补 waitlist 入口+转正（13d）；签到/consume/no_show 扣课/pending_no_show（13e）；C 端 `learner_self_service` 自助预约+认证（Batch 14，届时仅 `source` 不同的薄包装）；per-location 策略 + 场次 override 的 CRUD（解析层读但不写）；学员同一时间冲突检查（§22.1 失败场景「时间冲突」——留 13d/13e 评估，13c 仅靠 unique 防同场次重复）。

**前向依赖（后批必接）：**
- 13d waitlist：满员转候补 + 取消后转正 + **扩 TX-3 级联取消 waitlist_entries**。
- 13e attendance：`booked→attended/pending_no_show/no_show`、hold `held→consumed`、`no_show_consumes_entitlement` 真扣课、占位预约签到提醒。
- Batch 14 C 端：`source=learner_self_service`、`cancel_source=learner`、学员端不可选权益（§5.7 系统自动）。

**转 FR 候选：** 员工代取消绕过 cancel_deadline；学员同一时段多场次冲突检查；booking 通知（微信订阅消息）；per-location/场次 override 策略 CRUD；预约窗口/截止时间的 per-brand 时区（合并 12b TZ FR）。

---

## 10. 流程铁律（本批严格遵守）

写契约 → **会话内 approve（不发邮件）** → 写测试场景 `batch-13c-booking-tests.md` → **主线程逐 task TDD commit**（先红→实现→绿→单 task commit，按文件/域粒度；用户不 spawn 实现 subagent） → `go build ./... && go test ./...` + `pnpm --filter @mini-schedule/brand lint+build` → `/code-review`（后端 correctness / 前端 correctness 两并行 review，修或转 FR） → **业务验收停止点**（e2e 用户另开 session，给自包含 prompt） → 更新 PROGRESS + 三库 `.learnings` → 三仓库分别 push → backend/web `dev` FF 合并 main（`git push origin dev:main`；pds 直接 main）。
- DB 单测用真实 Postgres（`newMigratedTestDB` 自动建库跑 migration，无 PG 时 skip）；migration 从 **000012** 起。
```
