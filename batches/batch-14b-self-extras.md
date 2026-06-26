# Batch 14b 契约 —— C 端学员自助增量（我的权益 + 上课记录 + 加入候补）

> **Batch 14（C 端微信自助预约）第二子批**，14a 核心环（桥接 + 课程表 + 下单 + 我的预约 + 取消）之上的增量。
> 14a 已建：桥接(profile_id 入 JWT)、`learnerbooking.Service`(无 RBAC/ownership 收口)、app api 模块、课程表/我的预约页。14b 复用全部模式。
> 前置 14a ✅ 已 merge main（dev=main，DB v12）。来源优先级：blueprint §7.2 菜单 / §5.7 权益 / §22.4 候补 / §14.4 菜单 / §23.7 验收。

## 决策记录（grill 后 AskUserQuestion 拍板，2026-06-26）

| # | 决策点 | 选择 | 影响 |
|---|---|---|---|
| 1 | 加入候补 learner 服务形态 | **参数化 Join + writeWaitlistLogAs** | JoinInput 加 self-service 语义（operated_by NULL + audit actor=learner + scope nil）；writeWaitlistLog 抽 writeWaitlistLogAs(actor 参数)；staff Join 行为不变，复跑 13d 回归 |
| 2 | 上课记录实现 | **后端多状态 filter** | booking `ListFilter` 加 `Statuses []string`（IN 过滤）；learner 端点 `status` 支持逗号分隔（attended,no_show）；复跑 13c–13e 回归 |
| 3 | 页面/导航布局 | **「我的」页 hub + 候补折叠进我的预约** | 我的权益/上课记录 = 「我的」(profile)页入口链接 → 独立子页；底部导航维持 4 项；我的候补 折叠进「我的预约」页（加「候补中」筛选）；课程表满员→加入候补 |

## 数据模型 & Migration

**Migration = 零**（dev DB 保持 v12）。逐项核对：
- 我的权益：复用 `learner_entitlements`/`entitlement_products`；`ListEntitlementsByLearner` **已 RBAC-free + 读触发 settle**（C 端读自动落库正确 expired/depleted）。
- 上课记录：复用 `bookings`（attended/no_show 终态，13e 已产）；ListFilter 加 `Statuses` 是纯查询，无 schema。
- 加入候补：`waitlist_entries`（000003）；`operated_by` **brand_users FK 可空**（学员自助传 NULL）；`operation_logs.actor_type` CHECK 已含 `'learner'`。§22.4：候补**不锁权益**（无 entitlement txn FK 问题）。
- **零新权限码**（学员无 RBAC）；**零新错误码**（候补复用 13d `WAITLIST_*` + `BOOKING_DUPLICATE`/`LEARNER_NOT_BOOKABLE`/`SESSION_NOT_FOUND`）。

## A. 我的权益

**后端**：`learnerbooking.Service` 注入 `entitlement.Repository`，加 `ListMyEntitlements(ctx, brandID, profileID)` → `entRepo.ListEntitlementsByLearner(brandID, profileID)`（RBAC-free，settle-on-read）。`requireProfile` 守卫。
**端点**：`GET /api/v1/app/entitlements` → `[]Entitlement`（product_name/product_type/total/remaining/locked/status/expires_at，snake_case 已齐）。
**前端**：`(protected)/entitlements` 页（从「我的」hub 进）：权益卡片（产品名 + 剩余/总·不限次 + 到期 + 状态 badge）。hook `useAppEntitlements` 用 queryKey **`['app-entitlements']`**（14a 预约/取消已预埋失效 → 下单后余额即时刷新）。

## B. 上课记录

**后端**：`domainbooking.ListFilter` 加 `Statuses []string`；repo `List` 当 `len(Statuses)>0` 用 `status IN ?`（否则沿用单 `Status`，brand 侧不变）。`learnerbooking.ListMyBookings` 形参 `status string` → 解析逗号为 `[]string` 传 `Statuses`（单值=单元素，14a 我的预约 status=booked 不受影响）。
**端点**：复用 `GET /api/v1/app/bookings`，`status` 支持逗号分隔（上课记录传 `status=attended,no_show`）。**强制本 profile**（同 14a）。
**前端**：`(protected)/records` 页（从「我的」hub 进）：复用 14a 预约卡片，按终态展示（已到课/已爽约 + 课程/时间/门店 + 消耗权益 hold）。hook `useAppBookings('attended,no_show', ...)`。

## C. 加入候补

**后端（参数化 Join，镜像 14a）**：
- `domainwaitlist.JoinInput` 加 `SelfService bool`（或等价：service 传 `OperatedBy *int64`）。repo `Join`：`SelfService` 时 `OperatedBy=nil` + audit `Actor{ActorLearner, BrandLearnerProfileID}`；否则 `&ActorID` + `ActorBrandUser`（不变）。`writeWaitlistLog` 抽 `writeWaitlistLogAs(tx, actor, ...)`（镜像 14a `writeBookingLogAs`）。学员路径 `ScopeLocationIDs=nil`。
- 新 `waitlist.Repository.ListByLearner(ctx, brandID, profileID)`（活跃 waiting/eligible，按 session 时间序，反范式 course_title/location_name/position）——13d 仅有 ListBySession。
- `learnerbooking.Service` 注入 `waitlist.Repository`：`JoinWaitlist(brandID, profileID, sessionID)`（self-service）+ `ListMyWaitlist(brandID, profileID)`。requireProfile 守卫；转正仍 staff-manual（C 端不做）。
- 改完复跑 13d `TestWaitlist_*` 全 DB 单测证零回归。

**端点**：`POST /api/v1/app/waitlist`（`class_session_id`）→ `Entry`；`GET /api/v1/app/waitlist` → `[]Entry`（我的候补）。
**前端**：
- 课程表「已满」session：14a 的 disabled「已满」改「加入候补」按钮 → `useAppJoinWaitlist`；成功 toast +「已候补」；失败复用文案（WAITLIST_FULL/NOT_ALLOWED/DUPLICATE/已约）。
- 我的预约页加「候补中」筛选（chip）：切到 `useAppWaitlist`（GET /waitlist），渲染候补卡片（课程/时间/门店 + 位置 position）；候补不可自助转正（仅展示），可留「取消候补」(复用 13d W3 cancel 的 learner 路径——**本批是否含取消候补见范围**)。

## API 接口

| 方法 | 路径 | 鉴权 | 请求 | 响应 |
|---|---|---|---|---|
| GET | `/api/v1/app/entitlements` | profile | — | `[]Entitlement`（本人，settle-on-read） |
| GET | `/api/v1/app/bookings` | profile | `status`(支持逗号分隔), `page` | `PageResponse<Booking>`（上课记录传 attended,no_show） |
| POST | `/api/v1/app/waitlist` | profile | `class_session_id` | `Entry`（operated_by NULL, audit learner） |
| GET | `/api/v1/app/waitlist` | profile | — | `[]Entry`（我的活跃候补） |

## 前端页面模块（`web/apps/app`）

| 页面/模块 | 类型 | 关键 |
|---|---|---|
| `(protected)/profile`（我的） | 改造 | 加 hub 入口链接：我的权益 `/entitlements`、上课记录 `/records`（保留退出登录） |
| `(protected)/entitlements`（我的权益） | 新页 | 权益卡片列表；空态「暂无权益，请联系机构」 |
| `(protected)/records`（上课记录） | 新页 | 终态预约（已到课/已爽约）+ 消耗权益 |
| `(protected)/class-sessions`（课程表） | 改造 | 满员 disabled「已满」→「加入候补」按钮 + join mutation |
| `(protected)/bookings`（我的预约） | 改造 | 加「候补中」筛选 → GET /waitlist 渲染候补卡片（位置 + 可选取消候补） |
| packages/api `app.ts` | 代码 | `useAppEntitlements`(key app-entitlements) / `useAppJoinWaitlist` / `useAppWaitlist`（+候补 cancel?）+ 类型 AppEntitlement/AppWaitlistEntry；appBookingErrorText 补 WAITLIST_* 文案 |

**跨查询失效**：加入候补 → 失效 `app-class-sessions`（满员行状态）+ `app-waitlist`；（若做）取消候补 → `app-waitlist` + `app-class-sessions`。我的权益用 `app-entitlements`（14a 已预埋）。

## 范围 / 转 FR
- **IN**：我的权益 list、上课记录（终态）、加入候补（满员 self-join）+ 我的候补 list、**取消候补**（已拍板 IN：13d W3 cancel 的 learner 路径，operated_by NULL + tx 内 ownership，端点 `POST /api/v1/app/waitlist/:id/cancel`，对称自助取消预约）。
- **OUT（转 FR）**：候补转正（staff-manual，C 端不做）；候补转正/上课提醒微信订阅消息（§7.5）；真实 WeChat code2session；app_users 退役。

## 前端实现约束
- 复用 `/web/apps/app` `(protected)` 布局 + `components/ui`，不引新库。
- id 用 number（对齐后端 int64，14a 教训）；新 domain struct 经 REST 返回建时即加 snake_case json tag。
- 失败态明确中文（appBookingErrorText 补候补码）；mutation `{silent:true}` + inline。

## 验收前置 & 锚点（brand 21，沿用 14a alice/bob）
- 我的权益：brand 给 alice profile 发的权益，C 端「我的权益」可见（剩余/到期/状态正确，过期权益 settle 后显已过期）。
- 上课记录：brand 侧给 alice 的某 booking 标到课（13e），C 端「上课记录」显「已到课」+ 消耗权益；爽约显「已爽约」。
- 加入候补：满员场次（capacity 占满）C 端「加入候补」→ **psql** `waitlist_entries`(status=waiting、**operated_by NULL**、position 递增)、`operation_logs`(actor_type=learner)；brand 侧候补 drawer 见该 entry（同一 profile）。我的候补 list 显示。
- 回归：13d staff 候补 join/转正/跳过/取消仍正常；13c–13e 多状态 filter 改后全绿。
- ⚠️ 环境：api-app `CONFIG_PATH=configs/config-app.yaml` 重建重启 :8082；app :3003 `rm -rf .next`；filter `@mini-schedule/app`。
