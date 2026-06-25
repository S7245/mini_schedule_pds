# Batch 14 交接 prompt — C 端学员自助预约（WeChat Mini-Program Self-Service Booking）

> 新 session 直接读本文件按此执行。**第一步只做 grill 设计树 + 拆批/范围决策，不写代码。**
> 这是 **Batch 13（学员预约闭环 brand 侧 13a-13e）之后的第一个 C 端批**——把已建好的 booking/entitlement/waitlist 域暴露给学员自助（api-app :8082 + app 前端 :3003）。
> ⚠️ **不是「薄包装」**（早期 PROGRESS 的乐观假设）：核心难点是 **auth 桥接**——C 端现跑 legacy `app_users`，而 booking 域操作 `brand_learner_profiles`，两系统未打通。

## 先读（按序）
1. `pds/CLAUDE.md` — 每轮流程。**注意**：CLAUDE.md 写「邮件停止点」被 classifier 拦，本项目已改**会话内 approve / 验收，不发邮件**（13a-13e 实测惯例）。
2. `pds/PROGRESS.md` §5 的 **Batch 13a–13e** 段（brand 侧学员/权益/预约/候补/签到全闭环就绪）。
3. `pds/COURSE_BOOKING_BUSINESS_BLUEPRINT.md`（source of truth，优先级最高）：**§7 学员小程序需求（§7.1 入口+登录流程/§7.2 菜单/§7.3 预约/§7.4 取消/§7.5 微信订阅消息）、§22.1 学员自助预约流程、§22.3 取消预约、§22.4 简单候补、§14.4 学员小程序菜单、§23.7 学员小程序验收、§5.7 权益优先级、§9.x 状态机、§11 预约规则**。
4. 三库 `.learnings`（backend/web/pds）的 **13c/13d/13e 段必读** — booking 域 `placeBooking` 共享核心、`settleHoldForOutcome`/`settleHoldOnCancel` 收口、零 migration grill 纪律、派生态 stale、跨批复用复跑全测试证零回归。
5. 核对 schema：`app_users`（000001，**legacy**，有 `open_id`/`phone`/`brand_id`）、`learner_identities`（000003，`wechat_open_id` unique/`phone` partial unique）、`brand_learner_profiles`（000003，`learner_identity_id`+`brand_id`）、`bookings`（source CHECK **已含 `learner_self_service`** ✅）、`entitlement_holds`/`waitlist_entries`/`learner_entitlements`。

## C 端现状（已勘察核实，2026-06-25）
- **api-app（端口 :8082，路由前缀 `/api/v1/app`，config-app.yaml）全在 legacy `app_users` 系统**：
  - `internal/interfaces/app/handler.go`：`POST /auth/wechat-login`（公开）+ `JWTAuth("app")` 下 `/profile`(GET/PUT)、`/courses`、`/courses/:id`、`/trainings`(POST/GET)。
  - `wechatLogin`：**openid 是 dev 占位 `"dev_" + req.Code`**（无真实 code2session，注释明写「开发环境占位」，同 WeChat Pay mock）；find-or-create **app_user**（`appUserSvc.GetUserByOpenID`/`CreateUser`）；签发 JWT payload = `{UserID: app_user.ID, BrandID, UserType:"app"}`。
  - `internal/interfaces/middleware/auth.go` `JWTAuth`：注入 `user_id`(=app_user.ID)、`brand_id`、`user_type`。**token/context 不带 brand_learner_profile_id**。cookie 名 `app_access_token`。
- **app 前端（端口 :3003，包名 `@mini-schedule/app`）**：`app/(auth)/login/page.tsx`（输 `brand_id`[默认 `NEXT_PUBLIC_DEFAULT_BRAND_ID`]+登录码 `code` → POST wechat-login）+ `app/(protected)/{dashboard,courses,courses/[id],trainings,profile}`，全 legacy。`next.config.ts` rewrite `/api/*` → `http://localhost:8082/api/v1/app/*`。**booking/entitlement/brand_learner 零使用（greenfield）**。
- **booking 域全在 brand 侧、可复用**：`application/booking.Service`（brand RBAC `PermissionChecker`）、`persistence/booking_repository`（`Create`/`Cancel`/`Attend`/`EndSession`/`ConfirmNoShow` + 共享自由函数 `placeBooking`/`settleHoldForOutcome`/`settleHoldOnCancel`）、`waitlist`、`entitlement`、`learner`。`Create` 当前 hardcode `source=staff_assisted`，但 **`placeBooking(..., source string, ...)` 已参数化 source**（可传 `learner_self_service`）。

## 核心难点（grill 必须透，发现不可行就停下问用户）

### A. Auth 桥接：app_user → brand_learner_profile（最大难点）
booking 域所有操作要 `brand_learner_profile_id`，但 C 端 JWT 只有 `app_user.ID`。两系统**都 key on open_id**（`app_users.open_id` / `learner_identities.wechat_open_id`），但**无 FK 关联**，是两套并行身份。blueprint §7.1 明确登录流程应「微信登录 → 手机号绑定/授权 → **创建或关联该品牌下 BrandLearnerProfile** → 进入学员首页」。
- 倾向方案：`wechat-login` 时 find-or-create `learner_identity`（by openid，复用 13a 逻辑）+ `brand_learner_profile`（by brand+identity），把 `brand_learner_profile_id` 带入 token payload / 或每请求解析。app_user 与 learner 双轨并存（legacy app_users 退役留 FR）。
- 真实 WeChat `code2session` 需 per-brand 小程序 AppID/secret，v1 大概率沿用 dev 占位 openid + 手机号绑定，真实接入留 FR（与 WeChat Pay mock 同节奏）。

### B. 学员自助预约的服务路径：无 brand RBAC
13c `booking.Service` 用 brand RBAC `PermissionChecker.Require(code)`。学员不是 brand_user，无品牌权限码。C 端自助 = **新 app-side 应用服务**（或 booking.Service 的 learner 模式），鉴权 = 「该 profile 属于本次登录的 (brand, identity)」**所有权校验**，而非 RBAC。复用 booking **Repository**（`placeBooking`/`Cancel`/`UsableEntitlements`），不复用 brand Service。
- 学员模式约束：source=`learner_self_service`；entitlement mode **仅 auto**（从学员自己可用权益按 §5.7 自动选；**无 none 占位**[staff 专属]、**无 manual 指定**他人权益）；预约窗口/取消截止/频次/同时上限**对学员强制生效**。
- 候补：学员可**加入**候补（§22.4，复用 13d W1 Join）；**转正仍 staff-manual**（已建，C 端不做）。
- **签到/爽约纯 staff 侧（13e），C 端完全不做**。

### C. §22.1「同一时间已有预约」时间冲突校验
§22.1 列失败场景「同一时间已有预约 → 时间冲突」。13c 有 `BOOKING_DUPLICATE`(同场次 partial unique) + `concurrent_booking_limit`(频次)，但**未必有跨场次时间重叠校验**（学员同一时段约了两节不同课）。核对 13c 现状，决定是否新增。

## grill 设计树重点
- **范围**：C 端 = 课程表(场次列表/详情) → 自助预约(Create `learner_self_service`/auto) → 我的预约(list + 自助取消) → 我的权益(list) → 上课记录(履约 list，复用 13e 终态) → 加入候补(满员)。订阅消息(§7.5)、真实 WeChat、app_users 退役 → 留 FR。
- **跨域复用**：learner/booking/entitlement/waitlist 域全就绪（brand 侧），C 端增量 = 新 app application 服务 + 新 `/api/v1/app` endpoints + app 前端页面 + app-side api client；**复用 repo 层**。
- **C 端 endpoints 草案**（`/api/v1/app` 下，JWTAuth("app") + profile 解析）：`GET /class-sessions`(只读，brand+published+scheduled，窗口内)、`GET /class-sessions/:id`、`POST /bookings`(自助 auto)、`GET /bookings`(我的，含履约终态)、`POST /bookings/:id/cancel`(自助，ownership 校验)、`GET /entitlements`(我的)、`GET /bookings/usable-entitlements`(预约前预览)、`POST /waitlist`(加入候补)、`GET /waitlist`(我的候补)。
- **失败展示**（§7.3/§22.1，全是 13c 既有错误码）：无权益(`ENTITLEMENT_NONE_AVAILABLE`)/满员(`SESSION_FULL`→引导候补)/超窗口(`BOOKING_WINDOW_CLOSED`)/超截止(`BOOKING_CANCEL_DEADLINE_PASSED`)/取消后不可重约(partial unique)/频次(`BOOKING_FREQUENCY_EXCEEDED`)/同时上限/学员状态不可用(frozen)。C 端前端做明确中文文案。
- **跨批回改检查**：C 端场次列表只读复用 classsession（按 brand+status+published filter，可能需 app-side 只读 query）；booking learner 路径走 `placeBooking`（**复跑 13c-13e 全 DB 单测证零回归**，同 13d/13e 纪律）。

## 拆批 / 范围决策点（grill 后 AskUserQuestion 拍板，≤4 问；甜区 4-6）
1. **Auth 桥接模型**：(a) 登录 find-or-create identity+profile，`brand_learner_profile_id` 入 token payload（**推荐**，合 §7.1）/ (b) 保留 app_user token，每请求按 openid 解析 profile（中间件）/ (c) C 端全迁 learner_identities 弃 app_users（最大改动）。**+ 真实 WeChat code2session 是否本批**（推荐沿用 dev 占位 openid + 手机号绑定，真实留 FR）。
2. **自助预约服务形态**：(a) 新 app-side `LearnerBookingService` 复用 booking Repository（**推荐**，与 brand RBAC 解耦干净）/ (b) booking.Service 加 learner 模式分支。
3. **C 端范围切片**：最小闭环(课程表→自助预约→我的预约+取消) vs 全菜单(+我的权益+上课记录+加入候补)。**推荐全菜单纵切片**（已建域复用，增量主要在 endpoint+UI）；订阅消息单列 FR。
4. **时间冲突校验 + migration**：是否新增跨场次时间重叠校验(§22.1) / 仅靠 `concurrent_booking_limit`。**+ 本批是否需 migration**（倾向**零 migration 纯 token 桥接**——identity+profile find-or-create 复用 13a 无新表；若选 app_users↔profile link 列则 000013 起）。

## 复用模式（13a-13e 已验证，直接照搬）
- booking Repository：learner Create 走 `placeBooking`（传 source=`learner_self_service`，actor=系统/学员）；`Cancel`(加 ownership 校验：booking.profile == 登录 profile)；`UsableEntitlements`；waitlist `Join`。
- learner find-or-create：13a `learner_repository` 的 identity(by openid/phone, 合成 openid 经验) + profile(by brand+identity) 逻辑，桥接登录直接复用。
- 错误码全在 `pkg/errors`（booking/entitlement/waitlist 13c-13d 已建，C 端零新增大概率）。
- 前端：app 前端复用现有 `(protected)` 布局 + `components/ui`；新页 课程表/我的预约/我的权益/上课记录；packages/api 新 app-side client module（`@mini-schedule/api/app` 现有或新增 booking/entitlement app 模块）；mutation 跨查询失效（自助预约改 app-bookings + app-entitlements + class-sessions）；失败态明确文案。

## 流程铁律（本项目惯例，严格遵守）
grill 设计树 → 写契约 `pds/batches/batch-14-*.md` → **会话内 approve**（不发邮件）→ 写测试场景 `batch-14-*-tests.md` → **主线程逐 task TDD commit**（用户不要 spawn 实现 subagent；先红→绿→单 task commit）→ `go build ./... && go test ./...` + `pnpm --filter @mini-schedule/app lint+build` → `/code-review`（2 并行 review subagent：后端/前端 correctness）→ **业务验收停止点**（e2e 由用户另开 session 跑，给自包含 prompt）→ 更新 PROGRESS + 三库 `.learnings` → 三仓库分别 push → backend/web 的 `dev` FF 合并 `main`（`git push origin dev:main`）。
- ⚠️ **本批是 C 端，端口/filter 全变**：后端 **api-app :8082**（`go build -o /tmp/api-app ./cmd/api-app` 重建重启，旧二进制掩盖改动）；前端 **app :3003**（`rm -rf apps/app/.next && pnpm --filter @mini-schedule/app dev` + curl 预热）；filter 用 **`@mini-schedule/app`**（不是 brand）。除 e2e 外必跑 prod `pnpm --filter @mini-schedule/app build`。
- 仓库：backend/web 在 `dev`，pds 在 `main`。
- DB 单测用真实 Postgres（`newMigratedTestDB`）。**dev DB 现 v12**；Batch 14 若需 migration 从 **000013** 起（grill 定，倾向零 migration 纯 token 桥接）。
- 桥接/落库类（identity+profile find-or-create、hold、booking source=learner_self_service）验收须 **psql 实查 DB 真值**。

## 测试账号 / 数据（C 端，沿用 brand21）
- **C 端登录**：app 前端 :3003 `(auth)/login` 输 `brand_id`（默认 `NEXT_PUBLIC_DEFAULT_BRAND_ID`，brand21）+ 登录码 `code` → 后端 dev openid `"dev_"+code`（同 code 复登=同一身份，openid 稳定）。桥接后该 openid 应稳定映射到**一个** brand_learner_profile。
- **brand 侧准备**（owner `18816820405 / admin123`）：建 scheduled 场次（/schedule）+ 给「桥接出的学员 profile」发 active 权益（/learners 找到该 profile 或在学员权益 Tab 发放）。门店 讯美广场 loc1。
- ⚠️ **关键端到端验证**：C 端学员（openid→profile）自助预约后，**brand 侧 /bookings 应看到该 booking（source=`learner_self_service`、assisted_by 空）**，证明 C 端与 brand 操作的是**同一个 brand_learner_profile**（桥接正确）；brand 侧给该学员发的权益，C 端「我的权益」也应看到（同一 profile）。这是本批桥接正确性的核心验收。

---

**第一步只做 grill**：盘 C 端 legacy app_user auth 与 booking 域 brand_learner_profile 的桥接关系、画自助预约/取消/候补的服务路径（无 RBAC + ownership 校验 + 仅 auto entitlement）+ 复用 `placeBooking` 的事务边界与所有权校验点、给拆批/范围建议 + 上述 4 决策点推荐，然后用 AskUserQuestion 让用户拍板。拍板后再写 14 契约。
