# Batch 13d 契约 —— 候补机制（Waitlist：加入 / 手动转正 / 跳过取消 / 场次取消级联）

> Batch 13（学员预约闭环）第四子批。前置 13a 学员 ✅ / 13b 权益 ✅ / 13c 预约下单 ✅ 均已 merge main。
> 范围：brand 后台 **满员加入候补 + 员工手动转正 + 跳过/取消 + 查看名单 + 场次取消级联候补**。
> 不做：自动通知 / 限时确认 / 超时顺延（§12 留后续池）；C 端自助候补（Batch 14）；签到/履约（13e）。

---

## 0. grill 拍板结论（2026-06-22 会话内）

| # | 决策点 | 结论 |
|---|---|---|
| 1 | 转正模型 | **手动转正 + 容量门**：promote 锁 session 校 `booked_count<capacity`（锁后复验）；**不做 booking 取消时自动标 eligible_to_promote**（零 cancel↔waitlist 耦合）；状态机实际只用 `waiting`→{promoted,skipped,cancelled} |
| 2 | 加入候补入口 | **独立 endpoint** `POST /bookings/waitlist`；13c 代预约弹窗满员(`SESSION_FULL`)时显示「加入候补」按钮调它 |
| 3 | 候补配置来源 | **effective policy**（allow_waitlist + waitlist_limit，解析 brand-default/location/场次 override，同 13c）；`waitlist_limit=0` = **不限**（由 allow_waitlist 控开关）；`class_sessions.waitlist_limit` 不作 live cap |
| 4 | 候补名单 UI | **场次维度 drawer**：从场次行（`/schedule` / `/bookings`）打开该场次候补名单 drawer（按 position + 转正/跳过/取消） |

**已锁定实现默认（不再问）：**
- 转正无权益 → §22.4「auto/manual/none 占位 + 独立跳过」；候补时**不锁权益**，转正才锁。
- 全程 brand `staff_assisted`（转正 booking `source=waitlist_promotion`）；C 端自助候补留 Batch 14。
- data_scope 同 13c（按 `class_session.location_id` ∈ assigned_locations，越权 404 不泄漏存在性）。
- W1 加入候补锁 session 行串行化 position 分配（position = 活跃 max+1）。

---

## 1. Schema 现实（waitlist_entries 已核实，**13d 零 migration**）

| 项 | 事实 |
|---|---|
| 列 | `position`(CHECK >0)、`status`(默认 `waiting`)、`promoted_booking_id`(FK bookings ON DELETE SET NULL)、`skipped_reason`、`operated_by`(FK brand_users SET NULL)；**无 hold 列**（候补不锁权益，转正才建 booking+hold） |
| status CHECK | `waiting / eligible_to_promote / promoted / cancelled / skipped` |
| unique | ① `idx_waitlist_entries_session_learner_active`(session,learner) partial `WHERE status NOT IN(cancelled,skipped)`（**13c 000012 已建**）② `idx_waitlist_entries_session_position_active`(session,position) partial `WHERE status IN(waiting,eligible_to_promote)`（活跃队列内位置唯一）+ `idx_waitlist_entries_session_queue`(session,status,position) 队列 index |
| 权限 | `booking.view`（描述「查看预约**与候补**」）/`booking.create_assisted`/`booking.cancel` 000003 已 seed 齐 |

→ **零 migration、零权限迁移、零新权限码**（表 + 两条 partial unique + 权限全就绪；dev DB 维持 v12）。

---

## 2. 候补状态机（13d）

```
(满员+allow_waitlist 加入)──▶ waiting ──手动转正(校容量+锁权益)──▶ promoted [终态 +booking(source=waitlist_promotion)+hold]
                                  │──── 员工跳过 ───▶ skipped [终态 +skipped_reason]
                                  │──── 取消/移除 ──▶ cancelled [终态]
   eligible_to_promote：本批不主动写入（决策1 手动转正+容量门）；保留枚举供后续完整候补
```

---

## 3. 流程规格

**统一锁序铁律延续 13c：先锁 `class_sessions` 行，再锁 `learner_entitlements`。**

### W1 加入候补 `POST /bookings/waitlist`（单事务）
```
0. guardLocationInScope（session.location ∈ scope，否则 SESSION_NOT_FOUND 404）
1. SELECT FOR UPDATE class_sessions（串行化 position 分配 + 满员判定）
   → 校 status='scheduled'（否则 SESSION_NOT_BOOKABLE 409）
   → 解析 effective policy；allow_waitlist=false → WAITLIST_NOT_ALLOWED 409
   → 校 booked_count >= capacity（未满不该候补，应直接约）→ 否则 WAITLIST_SESSION_NOT_FULL 409
2. 学员可预约（frozen/inactive → LEARNER_NOT_BOOKABLE 409；不存在 404）
   → 学员对该场次已有 active booking → BOOKING_DUPLICATE 409（已约无需候补）
3. 校 waitlist_limit：COUNT 活跃候补(waiting/eligible) >= limit(>0 时) → WAITLIST_FULL 409（0=不限）
4. position = COALESCE(MAX(position) FILTER active, 0)+1 → INSERT waiting（不锁权益）
   重复候补撞 partial unique(session,learner active) 23505 → WAITLIST_DUPLICATE 409
5. audit(waitlist_joined)
```

### W2 转正（手动）`POST /bookings/waitlist/:id/promote`（单事务）
```
0. 解析 entry + guardLocationInScope（越权 WAITLIST_ENTRY_NOT_FOUND 404）
1. SELECT FOR UPDATE class_sessions（该 entry 的场次）
2. SELECT FOR UPDATE waitlist_entries → 校 status='waiting'（否则 WAITLIST_NOT_PROMOTABLE 409）
3. 校 booked_count < capacity（无空位 → SESSION_FULL 409）
4. placeBooking(复用 13c 下单核心)：entitlement_mode auto(§5.7)/manual(锁行全校验)/none 占位
   → booked_count++ → INSERT booking(source='waitlist_promotion') → 锁权益+hold+流水(action=hold)
   （none 占位须 reason；与 13c 同分流 SESSION_FULL/FREQUENCY/SCOPE/NOT_USABLE/NONE_AVAILABLE…）
5. entry → promoted + promoted_booking_id=新 booking.id
6. audit(waitlist_promoted)
```

### W3 跳过 / 取消
- `POST /bookings/waitlist/:id/skip`（`booking.create_assisted`）：校 status='waiting' → skipped + skipped_reason（顺位让下一个；§22.4 队首无权益时员工跳过）。audit(waitlist_skipped)。
- `POST /bookings/waitlist/:id/cancel`（`booking.cancel`）：校 status∈{waiting,eligible} → cancelled。audit(waitlist_cancelled)。

### W4 查看候补名单 `GET /bookings/waitlist?class_session_id=`（`booking.view`）
按 position 升序返回该场次候补（活跃优先，可含历史 promoted/cancelled/skipped 末尾）；内嵌学员快照 + 状态 + position + promoted_booking_id。data_scope 守卫场次门店。

### 回改 TX-3 场次取消级联（扩展 13c）
`class_session_repository.Cancel` 级联**新增**：cancel 所有活跃候补（`status IN(waiting,eligible_to_promote)` → `cancelled`，operated_by=actor）。13c 已在该处留前向依赖注释。audit metadata 加 `cascaded_waitlist` 计数。

---

## 4. API 接口契约

| 方法 | 路径 | 权限 | 请求字段 | 响应字段 |
|---|---|---|---|---|
| POST | `/brand/bookings/waitlist` | `booking.create_assisted` | `class_session_id`, `brand_learner_profile_id` | `waitlist_entry{id,position,status,...}` |
| GET | `/brand/bookings/waitlist` | `booking.view` | `class_session_id` | `entries[]`（position 序，内嵌 learner/session 快照） |
| POST | `/brand/bookings/waitlist/:id/promote` | `booking.create_assisted` | `entitlement_mode∈{auto,manual,none}`, `learner_entitlement_id?`, `no_entitlement_reason?` | `booking{...新建}` + `waitlist_entry{...promoted}` |
| POST | `/brand/bookings/waitlist/:id/skip` | `booking.create_assisted` | `reason` | `waitlist_entry{...skipped}` |
| POST | `/brand/bookings/waitlist/:id/cancel` | `booking.cancel` | — | `waitlist_entry{...cancelled}` |

**list DTO 铁律**：waitlist 列表每行内嵌 learner 名/手机 + session 时间/课程 + position + status（列表行复用为 drawer 行操作初值，13b F1 教训）。

**错误码（新增至 `pkg/errors/error.go`）：**

| 码 | HTTP | 触发 |
|---|---|---|
| `WAITLIST_NOT_ALLOWED` | 409 | effective policy allow_waitlist=false |
| `WAITLIST_SESSION_NOT_FULL` | 409 | 场次未满，应直接预约而非候补 |
| `WAITLIST_FULL` | 409 | 活跃候补数达 waitlist_limit（>0） |
| `WAITLIST_DUPLICATE` | 409 | 该学员对该场次已在候补（partial unique 23505） |
| `WAITLIST_ENTRY_NOT_FOUND` | 404 | 候补不存在或越权 |
| `WAITLIST_NOT_PROMOTABLE` | 409 | 转正/跳过时 status≠waiting |

复用 13c：`SESSION_NOT_FOUND`(404)、`SESSION_NOT_BOOKABLE`、`SESSION_FULL`、`BOOKING_DUPLICATE`、`LEARNER_NOT_BOOKABLE`、`ENTITLEMENT_*`、`ASSISTED_REASON_REQUIRED`。

---

## 5. Migration

**无。** waitlist_entries 表（000003）+ 两条 partial unique（000012）+ booking.* 权限（000003 seed 齐）全就绪。dev DB 维持 **v12**。

---

## 6. 前端页面模块

| 页面/模块 | 类型 | 关键操作 |
|---|---|---|
| WaitlistDrawer(sessionId) | 抽屉（新） | 候补名单（position/学员/状态）+ 转正(PromoteDialog) + 跳过(reason) + 取消；从场次行打开 |
| 场次行「候补 (N)」入口 | 触发器 | `/schedule`（或 `/bookings`）场次行显示候补人数 + 打开 drawer（gate booking.view） |
| PromoteDialog | 弹窗 | 复用 13c BookingCreateDialog 的权益模式部分（auto 预览/manual 下拉/none 占位）；提交=promote |
| 代预约弹窗「加入候补」 | 13c 弹窗扩展 | 选中满员场次时，`SESSION_FULL` 路径显示「加入候补」按钮 → 调 join endpoint |

**api client：** `packages/api/src/waitlist.ts`（join/list/promote/skip/cancel + hooks）→ **登记 package.json exports**。
**常量：** 6 个 waitlist 错误码常量（复用 BOOKING_* 权限码，无新权限常量）。
**跨查询失效：** waitlist mutation（join/promote/skip/cancel）失效 `['brand-waitlist', sessionId]`；promote 还改 booking/容量/权益 → 额外失效 `['brand-bookings']`+`['brand-class-sessions']`+`['learner-entitlements']`（同 13c 三连）。

### 前端实现约束
复用 `/web/apps/brand` 现有组件；drawer 用现有 Dialog/Sheet 风格；PromoteDialog 抽 13c create-dialog 的权益模式子块复用，不重写。`data-testid`：`waitlist-open`/`waitlist-row`/`waitlist-promote`/`waitlist-skip`/`waitlist-cancel`/`waitlist-join`。

### Wireframe
```
代预约弹窗（场次满员时）──────
场次: [晨瑜 06/25 (讯美) 12/12 满]
⚠ 场次已满  [加入候补]   ← SESSION_FULL 路径

WaitlistDrawer(晨间瑜伽 06/25)──────
容量 12/12 · 候补 3
┌──┬────────┬──────┬──────────────────┐
│#1│张三13800│waiting│[转正][跳过][取消]│
│#2│李四13900│waiting│[转正][跳过][取消]│
│#3│王五…    │waiting│[转正][跳过][取消]│
└──┴────────┴──────┴──────────────────┘
（无空位时[转正]disabled+Hint「场次已满，需先有人取消」）

PromoteDialog（转正 #1 张三）────
权益: (•)自动 → 将使用「次卡-剩9」
      ( )手动指定  ( )无权益占位[原因__]
                          [取消][确认转正]
```

---

## 7. 复用映射

| 需求 | 复用 |
|---|---|
| 转正下单核心 | **抽取 13c `bookingRepository.Create` 的步骤 3-6**（解析权益 + booked_count++ + INSERT booking + 锁权益/hold/流水）为共享 `placeBooking(tx, sess, eff, learnerID, actorID, mode, manualID, reason, source, now)`；Create(staff_assisted) 与 Promote(waitlist_promotion) 共用，零逻辑漂移 |
| 效力策略 / 频次 / scope | 复用 13c `resolveEffectivePolicy`/`frequencyCounts`/`loadCandidates`/`entitlementScopeMatches` |
| 约束名分流 | `uniqueConstraint` → 23505 按约束名分 `WAITLIST_DUPLICATE` |
| data_scope | 镜像 13c service `scopeFilterIDs`/`guardLocationInScope` |
| 场次取消级联点 | `class_session_repository.Cancel`（13c 已留候补前向依赖注释） |
| 前端权益模式子块 | 抽 13c `booking-create-dialog` 的 mode radio + usable-entitlements 为可复用子组件，PromoteDialog 引用 |

---

## 8. 验收闭环

前置：13c 已建的 scheduled 场次（造一个 capacity 小的便于满员）+ 测试学员 + active 权益。

| 关键验收 | 手段 |
|---|---|
| 满员 → 加入候补（位置递增）→ 名单 drawer | UI |
| 取消一个 booking 腾位 → 手动转正队首（auto 选权益）→ entry promoted + booking 建 + booked_count++ | UI + **psql 实查**（booking source=waitlist_promotion、hold held、entitlement 扣额、entry.promoted_booking_id） |
| 转正无权益 → none 占位 / 跳过(skipped+reason) | UI + API |
| 无空位转正 → SESSION_FULL | API |
| allow_waitlist=false 加入 → WAITLIST_NOT_ALLOWED；超 waitlist_limit → WAITLIST_FULL；重复候补 → WAITLIST_DUPLICATE；未满候补 → WAITLIST_SESSION_NOT_FULL | API |
| **场次取消级联候补**（活跃候补全 cancelled）+ 已 promoted 的 booking 也级联 cancel(13c) | **psql 实查** |
| 越权（非 scope 场次）候补/转正 → 404；course_operator 转正 403、查看 200 | API |

> 验收铁律：后端 e2e 前 `go build -o /tmp/api-brand ./cmd/api-brand` 重建重启(:8081)；前端 `rm -rf .next` 重启(:3002)+curl 预热；filter `@mini-schedule/brand`；除 e2e 外跑 prod build；派生态(转正落库/级联)**须 psql 实查**。

---

## 9. 出范围 / 前向依赖

**本批不做：** 自动通知/微信订阅消息、限时确认、超时顺延、候补时锁权益（§12 后续池）；C 端自助候补（Batch 14，`source=learner_self_service` 的薄包装 + 学员自助加入/退出）；签到/consume/no_show（13e）；eligible_to_promote 自动标记（决策1，保留枚举）。

**前向依赖：** Batch 14 C 端候补复用本批 join/cancel；13e 签到不影响候补（promoted 后即普通 booking）。

**转 FR：** 候补转正微信订阅通知；自动转正（cancel 即顺位转正，需先解决无权益处理）；候补位置手动调整/插队；候补到期随场次过期清理。

---

## 10. 流程铁律
写契约 → **会话内 approve** → 测试场景 `batch-13d-waitlist-tests.md` → 主线程逐 task TDD commit（先红→绿→单 task commit）→ `go build/test` + `pnpm --filter @mini-schedule/brand lint+build` → `/code-review`（后端/前端 correctness 两并行，修或转 FR）→ 业务验收停止点（e2e 用户另开 session，给自包含 prompt）→ 更新 PROGRESS + 三库 `.learnings` → 三仓库 push → backend/web `dev` FF 合并 main（pds 直接 main）。
- DB 单测用真实 Postgres（`newMigratedTestDB`，v12 无新 migration）。
- 派生态/落库（转正扣额、级联取消）验收**须 psql 实查 DB 真值**。
