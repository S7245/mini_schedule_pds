# Batch 14a 契约 —— C 端学员自助预约核心环（WeChat 自助预约 · 桥接 + 课程表 + 下单 + 我的预约 + 取消）

> **Batch 14（C 端微信自助预约）第一子批**：把已建好的 booking/entitlement 域暴露给学员自助（api-app :8082 + app 前端 :3003）。
> 14a = **核心环 + auth 桥接**（证「C 端 openid→profile 与 brand 侧操作同一 brand_learner_profile」闭环）；14b = 我的权益 + 上课记录 + 加入候补（后批）。
> 前置 13a 学员 ✅ / 13b 权益 ✅ / 13c 预约下单 ✅ / 13d 候补 ✅ / 13e 签到 ✅ 均已 merge main（dev=main，DB v12）。
> 来源优先级：`COURSE_BOOKING_BUSINESS_BLUEPRINT.md` §7.1 / §7.3 / §7.4 / §22.1 / §22.3 / §5.7 / §9.2 / §9.3 / §11 / §23.7。
> ⚠️ **不是薄包装**：核心难点是 auth 桥接（C 端跑 legacy `app_users`，booking 域操作 `brand_learner_profiles`，两套并行身份无 FK）。

## 决策记录（grill 后 AskUserQuestion 拍板，2026-06-25）

| # | 决策点 | 选择 | 影响 |
|---|---|---|---|
| 1 | Auth 桥接模型 | **登录 find-or-create identity(by openid)+profile，`brand_learner_profile_id` 入 JWT** | 合 §7.1；零 migration（JWT 无状态）；dev 占位 openid + 真实 code2session/手机号绑定 → FR；app_user 双轨并存 |
| 2 | 自助服务形态 | **新 app 侧 `LearnerBookingService` 复用 RBAC-free repo** | ownership 替 RBAC；mode=auto/source=learner_self_service/scope=nil；镜像 13d `{db,bk}` 事务复用 |
| 3 | 范围/拆批 | **拆 14a 核心环 + 14b 增量** | 14a 隔离高风险桥接单独验收（仿 13c→13d/13e） |
| 4 | §22.1 时间冲突 | **学员路径加跨场次 [starts,ends) 重叠校验** | 只加在 learner 路径（不动共享 placeBooking，staff 代预约可故意双约）；零 migration（纯查询） |

## 数据模型 & Migration

**Migration = 零**（dev DB 保持 v12）。逐项核对：
- `app_users`（000001，**legacy**，brand+open_id）：**双轨保留**，C 端 profile/trainings 旧页不动；退役 → FR。
- `learner_identities`（000003）：`wechat_open_id VARCHAR(100) NOT NULL` + **唯一** `idx_learner_identities_wechat_open_id`（桥接 by-openid find-or-create 的 key）；`phone` partial-unique 可空（**v1 profile 无手机号合法**，phone 留 NULL）。
- `brand_learner_profiles`（000003 + 000010）：唯一 `idx_brand_learner_profiles_brand_identity(brand_id, learner_identity_id) WHERE deleted_at IS NULL`（**find-or-create profile 幂等**，每次登录可重入）。
- `bookings`（000003）：`source` CHECK 已含 **`learner_self_service`** ✅；`cancel_source` CHECK 已含 **`learner`** ✅；`assisted_by BIGINT REFERENCES brand_users(id)` **可空**——**自助预约必须 NULL**（FK 指向 brand_users，塞 learner id 会 23503 外键违反；学员身份靠 `brand_learner_profile_id` + 审计承载）。
- `operation_logs`（000003:1155）：`actor_type` CHECK 已含 **`'learner'`** ✅ → audit 学员 actor **仅补 Go 枚举**（`audit.ActorLearner`），零 migration。
- **零新权限码**（学员非 brand_user，无 RBAC；ownership 校验替代）。
- **JWT**：`cache.TokenPayload` 加 `ProfileID int64`（`profile_id` claim），无状态零 migration。

## Auth 桥接设计（难点 A · §7.1）

```
打开小程序 → 识别 brand_id → 微信登录(dev openid "dev_"+code)
  → find-or-create learner_identity (by wechat_open_id, phone=NULL)
  → find-or-create brand_learner_profile (by brand_id + identity, deleted_at IS NULL)
  → brand_learner_profile_id 写入 JWT payload
  → 进入学员首页
```

**新 repo 方法 `FindOrCreateProfileByOpenID(ctx, brandID, openID, nickname) (*Profile, error)`**（单 tx，**结构镜像 13a `Create`，非函数复用**——13a 是 by-phone+要求 phone+焊死 brand_user 审计，桥接是 by-openid+phone-less+每登幂等+learner 审计）：
1. find identity `WHERE wechat_open_id = openID`；缺则 INSERT（`wechat_open_id=openID`，`phone=NULL`，`nickname`，`status=active`）；并发撞唯一约束回查复用（镜像 `findOrCreateIdentity`）。
2. find profile `WHERE brand_id + learner_identity_id AND deleted_at IS NULL`；命中即返回（幂等）。
3. 缺则建：**`SubscriptionGuard.CheckAndCount(ResourceLearner)` 配额门** → INSERT profile（`primary_location_id=NULL`，`status=active`） → `audit.Write(Actor{Type: ActorLearner, ID: profile.ID}, "learner_self_registered")`。

**边界（验收须覆盖）**：
- brand 满 `max_learners` 时新学员**首登建 profile → `QUOTA_EXCEEDED`(409) 登录失败**（honor 硬限，返干净错误）。
- **同人双 profile**：brand 侧 13a 按手机号建档（openid `manual:<phone>`）与 C 端登录（openid `dev_xxx`）是**两个不同 identity/profile，v1 不自动合并**（by-phone↔by-openid 合并 → FR）。→ 验收链路必须**学员先 C 端登录建 profile P，brand 再给 P 发权益**。

**改造点**：
- `cache/jwt.go`：`TokenPayload.ProfileID` + `GenerateToken`/`GenerateRefreshToken`/`ParseToken` 三处加 `profile_id` claim。
- `interfaces/app/handler.go wechatLogin`：现有 app_user find-or-create **保留**（双轨）+ 串桥接 `FindOrCreateProfileByOpenID` + payload 带 `ProfileID`。
- `interfaces/middleware/auth.go JWTAuth`：注入 `profile_id`；新 `GetProfileID(c)`。`UserType="app"` 的 token 若缺 `profile_id`（桥接前旧 token）→ 视为需重新登录（业务端点取 profile_id 为 0 时返 401/重登，dev 重登成本低）。

## 学员自助服务路径（难点 B · 无 RBAC + ownership）

**新包 `internal/application/learnerbooking`**，`Service{ repo booking.Repository; sessionRepo classsession.Repository }`，**无 `PermissionChecker`**。每个方法用 token 的 `profileID` 收口所有权（不跨 profile）：

| 方法 | 复用 repo | 鉴权 |
|---|---|---|
| `ListSessions(brandID, from, page)` | classsession `ListBookable`（新 RBAC-free 只读） | brand 范围只读 |
| `GetSession(brandID, id)` | classsession 只读 | brand 范围只读 |
| `Book(brandID, profileID, sessionID)` | booking `CreateByLearner`（auto/self_service/overlap） | profile 自己下单 |
| `ListMyBookings(brandID, profileID, status?)` | booking `List(filter{brand, profile})` | 仅本 profile |
| `CancelMyBooking(brandID, profileID, id, reason)` | booking `CancelByLearner`（ownership tx 内） | 仅本 profile |
| `UsableEntitlements(brandID, profileID, sessionID)` | booking `UsableEntitlements(scope=nil)` | 仅本 profile（**预览，§5.7 学员不选权益**） |

**学员模式约束（锁死）**：source=`learner_self_service`；entitlement **仅 auto**（无 none 占位[staff 专属]/无 manual 指他人权益）；窗口/截止/频次/并发上限**强制**（placeBooking/Cancel 已有，直接复用）；`ScopeLocationIDs=nil`（无 data_scope，学员可约本 brand 任意 scheduled 场次）。

## Repo 改动（复用 placeBooking 的 3 摩擦点 + 重叠校验）

**改 `placeBooking`（参数化 assisted_by）**：现签名用 `actorID int64` 同时喂 `AssistedBy=&actor` 与 txn `operatedBy=&actor`。→ 改为 `assistedBy *int64`：
- staff `Create` / 候补 `Promote` 传 `&actorID`（不变）；
- learner `CreateByLearner` 传 **`nil`**（assisted_by NULL + txn operated_by NULL）。
- **改完立即复跑 13c `TestBooking_*`/`TestSessionCancel_*` + 13d `TestWaitlist_*` + 13e `TestAttend_*`/`TestNoShow_*` 全绿才提交**（13d/13e 抽取纪律）。

**补 `audit.ActorLearner`**：`audit.go` 加常量 `ActorLearner ActorType = "learner"` + `IsValidActorType` 分支（DB CHECK 已允许）。`writeBookingLog` 加 `actor audit.Actor` 形参（现调用点传 `{ActorBrandUser, actorID}`；learner 路径传 `{ActorLearner, profileID}`）。

**新 `CreateByLearner(ctx, in LearnerCreateInput) (*Booking, error)`**（镜像 `Create`，单 tx）：
1. `SELECT FOR UPDATE` session（`scope=nil`，`status='scheduled'`，否则 `SESSION_NOT_BOOKABLE`）。
2. 解析 effective policy + `WithinBookingWindow`（否则 `BOOKING_WINDOW_CLOSED`）。
3. **跨场次时间重叠校验**（难点 C，仅 learner 路径）：`hasOverlappingBooking(tx, brand, profile, sess)` —— 查该 profile 是否已有 `status NOT IN ('cancelled')` 的 booking、其 session `id<>本场次` 且 `starts_at < 本.ends_at AND ends_at > 本.starts_at` → 命中返 **`BOOKING_TIME_CONFLICT`(409)**。
4. `placeBooking(tx, sess, eff, brand, assistedBy=nil, profileID, ModeAuto, nil, "", "learner_self_service", now)`。
5. `audit.Write({ActorLearner, profileID}, "booking_created")`。

**新 `CancelByLearner(ctx, brandID, profileID, id, reason) (*Booking, error)`**（镜像 `Cancel` + ownership）：
1. 预读 booking `WHERE id AND brand_id`（不存在→`BOOKING_NOT_FOUND`）。
2. **ownership：`booking.brand_learner_profile_id == profileID`，否则 `BOOKING_NOT_FOUND`(404，不泄漏存在性)**。
3. 锁序先场次后预约：`SELECT FOR UPDATE` session → booking `WHERE id AND brand_id AND brand_learner_profile_id=profileID`。
4. 校 `status='booked'`（否则 `BOOKING_NOT_CANCELLABLE`）→ policy `AllowCancel`（否则 `BOOKING_CANCEL_NOT_ALLOWED`）→ `CancelDeadlinePassed`（否则 `BOOKING_CANCEL_DEADLINE_PASSED`）。
5. `applyCancel(..., cancelSource=`**`booking.CancelSourceLearner`**`, release=eff.ReleaseOnCancel)`（hold release/forfeit 复用既有）。
6. `audit.Write({ActorLearner, profileID}, "booking_cancelled")`。

**新 classsession `ListBookable(ctx, brandID, from, offset, limit)`**（RBAC-free 只读，不经 brand classsession.Service 的 `require schedule.view`）：`WHERE brand_id AND status='scheduled' AND starts_at >= from` order `starts_at ASC`，返回反范式（course_title/location_name/capacity/booked_count）。

**接口登记**：`booking.Repository` 加 `CreateByLearner`/`CancelByLearner`；`classsession.Repository` 加 `ListBookable`（或等价只读 query）。新增的 domain struct 经 REST 返回**建时即加 snake_case json tag**（13c P0 教训）。

## API 接口

路径前缀 `/api/v1/app`，`JWTAuth("app")` + profile 解析（除登录公开）。

| 方法 | 路径 | 鉴权 | 请求字段 | 响应字段 |
|---|---|---|---|---|
| POST | `/auth/wechat-login` | 公开 | `brand_id`, `code`, `nickname?` | `access_token`(含 profile_id), `refresh_token`, `user`, `is_new_user` |
| GET | `/class-sessions` | profile | `page`, `page_size`, `from?`(默认 now) | list：`id, course_title, location_name, starts_at, ends_at, capacity, booked_count, remaining, status` |
| GET | `/class-sessions/:id` | profile | — | 场次详情 + 预约规则提示（窗口/取消截止派生文案） |
| POST | `/bookings` | profile | `class_session_id` | `Booking`（`source=learner_self_service`, `assisted_by=null`, 含 hold） |
| GET | `/bookings` | profile | `status?`, `page`, `page_size` | 我的预约（**仅本 profile**，含 hold.product_name；终态 attended/no_show 也返，14b 上课记录复用） |
| POST | `/bookings/:id/cancel` | profile | `reason?` | `Booking`（`status=cancelled`, `cancel_source=learner`） |
| GET | `/bookings/usable-entitlements` | profile | `class_session_id` | 可用权益预览（§5.7 序，`auto_selected=true` 为将用项；**仅展示，学员不选**） |

**ownership 铁律**：`GET /bookings` 后端强制 `brand_learner_profile_id = token.profile_id`（不接受前端传 learner 参数）；`POST /bookings/:id/cancel` 在 tx 内校所有权（越权 404）。`GET /class-sessions` 是 brand 范围只读（任意 scheduled 场次）。

## 新增错误码（`pkg/errors/error.go`）

| 码 | HTTP | 触发 |
|---|---|---|
| `BOOKING_TIME_CONFLICT` | 409 | 学员同一时段已有未取消预约（跨场次 [starts,ends) 重叠，§22.1） |

**复用 13c 既有码**（C 端零新增，仅前端补中文文案）：`ENTITLEMENT_NONE_AVAILABLE`(无权益→引导联系机构) / `SESSION_FULL`(满员→14b 引导候补，14a 暂提示「已满」) / `BOOKING_WINDOW_CLOSED`(超窗口) / `BOOKING_CANCEL_DEADLINE_PASSED`(超截止) / `BOOKING_CANCEL_NOT_ALLOWED`(场次不允许取消) / `BOOKING_DUPLICATE`(同场次重复/取消后 partial unique) / `BOOKING_FREQUENCY_EXCEEDED`(频次) / `LEARNER_NOT_BOOKABLE`(学员冻结) / `SESSION_NOT_BOOKABLE`(场次非 scheduled) / `BOOKING_NOT_FOUND`(越权/不存在) / `BOOKING_NOT_CANCELLABLE` / `QUOTA_EXCEEDED`(首登建 profile 超额)。

## 事务设计

**桥接 FoC（登录内，单 tx）**：identity by openid（缺则建，并发回查）→ profile by brand+identity（缺则配额门 + 建 + audit）。幂等：每次登录重入，命中即返。

**TX-L1 自助下单 `CreateByLearner`**：锁 session(scheduled/窗口) → **重叠校验** → `placeBooking`(assisted=nil/auto/self_service) → audit(learner)。锁序 session→entitlement（与 13c 一致）。并发兜底靠 `bookings` partial unique(session_learner) + booked_count 行锁 + entitlement 行锁（复用 13c 全套）。

**TX-L2 自助取消 `CancelByLearner`**：ownership 预校 → 锁 session→booking(带 profile 条件) → policy/deadline → `applyCancel`(cancel_source=learner) → audit(learner)。hold release/forfeit 复用 `settleHoldOnCancel`。

**TX-3（场次取消级联）零改动**：brand 侧 `ClassSession.Cancel` 已级联 cancel 所有 active booking（含 learner_self_service，按 status 不按 source）→ C 端预约被 brand 取消场次时一并 cancelled + hold released，已覆盖。

## 前端页面模块（`web/apps/app`）

| 页面/模块 | 类型 | 关键字段/操作 |
|---|---|---|
| `(auth)/login` | 改造 | 现有 `brand_id`(默认 `NEXT_PUBLIC_DEFAULT_BRAND_ID`)+`code` 登录；`onSuccess` **`queryClient.clear()`**（会话边界铁律）；token 带 profile |
| `(protected)/class-sessions`（课程表） | 新页 | 场次卡片列表（课程/时间/门店/`剩余 N`）+「预约」按钮；下拉/分页；空态；失败态中文文案 |
| `(protected)/class-sessions/[id]`（场次详情，可选） | 新页 | 场次详情 + 预约确认（经 `usable-entitlements` 显示「将使用 X 权益」/「无可用权益」）+「确认预约」 |
| `(protected)/bookings`（我的预约） | 新页 | 我的预约列表（状态筛选：即将上课/已取消/已结束）+「取消」按钮（截止内可点）+ ConfirmDialog |
| types / api client | 代码 | `packages/api` 新 app 模块（`listSessions`/`getSession`/`book`/`listMyBookings`/`cancelBooking`/`usableEntitlements`）；登记 exports；类型逐字段对齐后端 snake_case（含 nullable）；错误码常量 + 中文映射 |

**跨查询失效**（mutation 后）：自助预约 → 失效 `app-class-sessions`(remaining 变) + `app-bookings`（+ 14b `app-entitlements`）；自助取消 → `app-bookings` + `app-class-sessions`。模态/行派生计数从 live query 派生，不用冻结快照（13d 教训）。

## Wireframe（mobile，课程表 + 我的预约）

```
课程表                              我的预约
┌──────────────────────────┐      ┌──────────────────────────┐
│ 晨间瑜伽                  │      │ [即将上课] [已取消] [已结束]│
│ 06-26 09:00–10:00         │      ├──────────────────────────┤
│ 讯美广场 · 剩余 3         │      │ 晨间瑜伽  06-26 09:00     │
│              [ 预约 ]     │      │ 讯美广场 · 月卡           │
├──────────────────────────┤      │              [ 取消 ]     │
│ 搏击操                    │      ├──────────────────────────┤
│ 06-26 09:30–10:30 (重叠!) │      │ 搏击操    06-25 (已取消)  │
│ 讯美广场 · 剩余 5         │      │ 自助取消                  │
│  [ 预约 ]→时间冲突提示    │      └──────────────────────────┘
└──────────────────────────┘
失败态文案：无可用权益/已满/超窗口/超截止/时间冲突/频次上限/状态不可用
```

## 前端实现约束
- 复用 `/web/apps/app` 现有 `(protected)` 布局 + `components/ui`，不引新 UI 库。
- 失败态明确中文文案（map 13c 既有码 + `BOOKING_TIME_CONFLICT`）；API 级 inline 用 `{silent:true}`+setState（沿用既有）。
- **会话边界铁律**：login `onSuccess` / logout 必 `queryClient.clear()`（web 跨用户缓存泄漏教训）。
- hydration 竞态：app `(protected)` layout 已知 zustand persist 未水合瞬时弹走（既有 FR，本批不引新、不修，沿用）。

## 范围 / 转 FR
- **IN（14a）**：auth 桥接（identity/profile by-openid FoC + profileID 入 JWT）、课程表只读、自助预约（auto/self_service）、我的预约（本 profile）、自助取消（ownership/cancel_source=learner）、权益预览、跨场次时间冲突校验。
- **OUT（14b）**：我的权益 list（`GET /entitlements`）、上课记录（13e 终态 list）、加入候补（`POST /waitlist` 满员）。
- **OUT（转 FR）**：微信订阅消息（§7.5）；真实 WeChat `code2session` + per-brand 小程序 AppID/secret；手机号绑定/授权（§7.1）；同人 by-phone↔by-openid identity 合并；app_users 退役（profile/trainings 旧页）；候补转正（staff-manual）；满员引导候补 UI（14b）。

## 验收前置数据 & 桥接锚点（brand 21）
- **C 端登录**：app :3003 `(auth)/login` 输 `brand_id`(默认 brand21)+`code` → dev openid `"dev_"+code`（同 code 复登=同一身份，openid 稳定 → 稳定映射**一个** profile）。
- **brand 侧准备**（owner `18816820405 / admin123`）：① 建 scheduled 场次（/schedule，门店 讯美广场 loc1）② 给「桥接出的 profile」发 active 权益（/learners 找到该 C 端学员 profile → 权益 Tab 发放）。
- **核心桥接锚点（本批验收重心）**：C 端学员（openid→profile）自助预约后——
  1. **brand 侧 `/bookings` 应见该 booking：`source=learner_self_service`、`assisted_by` 空、同一 `brand_learner_profile`**（证 C 端与 brand 操作同一 profile）。
  2. brand 给该 profile 发的权益，C 端 `usable-entitlements` 预览可见（同一 profile）。
- **psql 实查**（桥接/落库类）：`learner_identities`(by `dev_<code>` openid)、`brand_learner_profiles`(by brand+identity)、`bookings`(source/assisted_by NULL)、`entitlement_holds`、`operation_logs`(actor_type=learner)。
- **重叠校验**：约两节时段重叠的不同课 → 第二节返 `BOOKING_TIME_CONFLICT`。
- ⚠️ **环境纪律**：后端 `go build -o /tmp/api-app ./cmd/api-app` 重建重启（旧二进制掩盖改动）；前端 `rm -rf apps/app/.next && pnpm --filter @mini-schedule/app dev` + curl 预热；filter 用 `@mini-schedule/app`。除 e2e 外必跑 prod `pnpm --filter @mini-schedule/app build`。
