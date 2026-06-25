# 课程预约实现进度

更新时间：2026-06-16

状态：平台商业化第一阶段进行中

## 1. 当前阶段目标

先完成 **平台端商业化闭环**，再进入品牌端业务配置和学员预约。

当前阶段的闭环定义：

```text
平台配置 SaaS Plan
-> 品牌负责人公开注册
-> 创建 Pending Brand / Pending Staff
-> 选择套餐并微信支付
-> 回调验签、金额校验、幂等处理
-> 开通 BrandSubscription 快照
-> 进入品牌初始化向导
-> 平台可查看、续期、冻结、补偿和审计
```

## 2. 已完成

| 模块 | 状态 | 说明 |
|---|---:|---|
| 需求蓝图 | Done | `COURSE_BOOKING_BUSINESS_BLUEPRINT.md` 已整理完整业务边界 |
| Go 语言设计 | Done | `GO_BACKEND_LANGUAGE_DESIGN.md` 已补充后端语言和分层约束 |
| 数据库基础表 | Done | `000003_course_booking_schema` 已包含平台、品牌、课程、预约、权益等核心表 |
| 平台套餐后端 | Done | SaaS Plan 创建、列表、启停 |
| 平台订阅后端 | Done | BrandSubscription 列表、续期、额度调整、状态调整 |
| 平台审计后端 | Done | 手动商业关键操作写 OperationLog |
| 平台支付观测后端 | Done | 订单、支付流水、回调日志查询接口 |
| 平台 summary 后端 | Done | 商业化健康指标接口 |
| 公开注册首购订单后端 | Done | Public API 创建 pending Brand、Brand Owner 占位和 pending_payment SaaSPlanOrder |
| Admin 前端平台页 | Done | dashboard、套餐、订阅、支付、操作日志页面 |
| 微信支付 Native 下单 | Done | 后端四层实现（domain/app/repo/handler）+ Brand 前端支付页（二维码/倒计时/轮询） |
| self-improving 本地学习 | Done | `/pds/.learnings` 已关联 `self-improving-agent` |

## 3. 当前缺口

| 优先级 | 缺口 | 说明 |
|---|---|---|
| P0 | 微信支付 RSA 签名接入 | 下单 + 回调验签均为 mock，待商户号下来后接真实 RSA-SHA256 / AES-256-GCM（service.go:CreateWeChatNativePay、wechat_adapter.go:VerifyAndDecrypt） |
| P0 | 支付异常补偿 | 需要从异常订单人工补偿开通订阅 |
| P0 | Learner 额度硬限制 | Batch 4 Location / Batch 5 Staff 已接 SubscriptionGuard；Learner POST 接口落地时一并接（参数化 ResourceKind） |
| P1 | Brand 初始化向导后续步骤 | Batch 4-5 完成第 1-3 步；第 4-8 步（课程分类/课程模板/权益/场次/小程序二维码）随后续 batch 逐步落地 |
| ✅ | RBAC enforcement + 数据权限 | Batch 6 完成：service 层 require(code) + data_scope（role_default/assigned_locations/own_*）实际生效 |
| ✅ | 品牌自定义角色 | Batch 7 完成：自建角色 / 调整权限 CRUD（B1 增量提权校验 + C1 缓存主动失效），2026-06-12 验收通过 |
| ✅ | 课程和场次 | Batch 11（分类+模板+单场次排课）+ Batch 12a（Location Resource 资源管理+场次绑资源+资源冲突）+ Batch 12b（RecurringSchedule 循环排课）全部完成。下一主线：Booking/学员预约 |
| P1 | 学员权益 | Entitlement Product、Learner Entitlement、Hold、Consume、Adjust |
| P1 | 学员预约 | 微信小程序品牌空间、课程表、预约、取消 |
| P2 | 候补机制 | 第一版可员工手动转正，自动候补后期补 |
| P2 | 通知 | 员工站内通知，学员微信订阅消息 |

## 4. Now / Next / Later

### Now

1. RBAC middleware + data_scope 落地（Batch 6 候选）。
2. 课程分类 / 课程模板 / 课程场次（向导第 4-7 步）。

### Next

1. 学员权益和权益模板。
2. 学员预约（小程序端）。
3. 小程序二维码生成（向导第 8 步）。
3. 学员权益模板和发放。
4. 学员预约（小程序端）。

### Later

1. 学员小程序预约。
2. 权益锁定和消耗。
3. 签到和爽约处理。
4. 微信订阅消息。
5. 候补机制增强。
6. 报表和数据分析。

## 5. 推荐下一批任务

### Batch 1：公开注册和首购订单

状态：Done（2026-05-28）

目标：

- 支持品牌负责人从公开入口提交手机号、短信验证码、密码、品牌资料。
- 创建 `pending` Brand。
- 创建 pending/active BrandUser 或 Staff 占位，按当前数据模型取舍。
- 选择 SaaS Plan 和 billing cycle。
- 生成 `SaaSPlanOrder`，金额必须取 SaaS Plan 价格。

后端范围：

- `internal/domain/commercial`
- `internal/application/commercial`
- `internal/infrastructure/persistence`
- `internal/interfaces/admin` 或新增 public commercial handler

前端范围：

- admin 暂时只需可查看订单。
- public purchase 页面可后置到支付批次。

验收：

- Done：`POST /api/v1/public/signup/orders` 可创建 pending Brand + BrandUser owner 占位 + pending_payment SaaSPlanOrder。
- Done：订单金额在后端事务中从 active SaaSPlan 的 monthly/yearly price 读取，不接收前端金额。
- Done：重复手机号按 Brand/User 唯一约束返回明确错误。

### Batch 2：微信支付 Native 下单 ✅

详细契约 + Wireframe：[pds/batches/batch-02-wechat-pay.md](batches/batch-02-wechat-pay.md)

契约状态：**已完成**（2026-05-29 人工验收通过）

完成内容：
- 后端：`POST /api/v1/public/payment/native` + `GET /api/v1/public/payment/orders/:order_id`（四层全通，mock 模式可本地运行）
- 前端：`/signup/payment/[order_id]` 页面，含品牌框架、步骤进度条、二维码、倒计时、状态轮询
- api-brand 路由补充 `/api/v1/public` 组，挂载 RegisterPublicRoutes
- 待完善：真实微信支付 RSA-SHA256 签名（生产环境需接入）

验收：

- 调用下单接口返回 `code_url`。
- 订单保持 `pending_payment`。
- 支付配置缺失时返回明确错误。
- 前端支付页二维码可展示，倒计时和状态轮询正常运行。

### Batch 3：微信支付回调开通订阅 ✅

详细契约：[pds/batches/batch-03-wechat-callback.md](batches/batch-03-wechat-callback.md)
测试场景：[pds/batches/batch-03-wechat-callback-tests.md](batches/batch-03-wechat-callback-tests.md)

契约状态：**已完成**（2026-06-06 Happy Path 人工验收通过，mock 模式）

完成内容：
- 后端：`POST /api/v1/public/payment/callback` 四层完整（domain + application + infrastructure + interfaces）
- 新增 `internal/infrastructure/payment/wechat_adapter.go`：mock 验签 + timestamp 防重放（≤ 5 min），真实 RSA-SHA256 / AES-256-GCM 留 TODO
- repository 单事务推进：`SELECT FOR UPDATE order` → 校验 trade_state / 金额 → 写 CallbackLog → 写 Transaction → 更新 Order → 创建 Subscription → 激活 Brand → 更新 CallbackLog 为 processed → 写 OperationLog
- 服务层兜底：验签失败 / 事务回滚场景在事务外补写 failed CallbackLog
- 配置：`payment.wechat.allow_mock` 默认 true，生产环境置 false 后会强制要求真实证书路径

验收（Happy Path 已通过日志验证）：

- 回调返回 `{"code":"OK","message":"success"}`
- saas_plan_orders.status = paid，paid_at 非空，third_party_trade_no 已填
- brand_subscriptions 新增一条 active（按 billing_cycle 推算 expires_at，额度从 plan 复制）
- brands.status = pending → active
- payment_transactions / payment_callback_logs / operation_logs 完整留痕

待完善：
- 真实微信 RSA-SHA256 验签和 AES-256-GCM 解密未实现（需商户证书 / 平台证书，等微信支付商户号下来后接）
- Edge cases（E1–E19）未逐条跑过；当前以 Happy Path 通过认定本批闭环
- `payment_callback_logs.headers` / `payload` 暂存空 `{}`，留档不完整，待审计追溯需求出现再扩

本批踩坑：
- `payment_callback_logs.headers` / `payload` 是 `JSONB NOT NULL DEFAULT '{}'`，模型用 `[]byte` 默认 nil 会写成 NULL → 违反约束。Fix：模型加 `BeforeCreate` hook 兜底为 `[]byte("{}")`

### Batch 4：品牌初始化向导骨架 + Location 闭环 ✅

详细契约：[pds/batches/batch-04-onboarding-location.md](batches/batch-04-onboarding-location.md)
测试场景：[pds/batches/batch-04-onboarding-location-tests.md](batches/batch-04-onboarding-location-tests.md)

契约状态：**已完成**（2026-06-06 Happy Path 人工验收通过）

完成内容：
- 后端 9 commits（`cab353d..3b7d710` on `dev`）：
  - migration 000004：`brands.description VARCHAR(2000)`
  - 新 domain `onboarding` + `location`；8 个 step_key（brand_profile / location / staff / course_category / course_template / entitlement_template / class_session / mini_program_qrcode）
  - onboarding 应用层：`GET /api/v1/brand/onboarding/status`（7 表实时 COUNT 聚合）、`PATCH /onboarding/steps/:key/skip`、`POST /onboarding/complete`（单事务原子化）
  - brand profile：`GET / PATCH /api/v1/brand/profile`，白名单字段
  - Location 完整 CRUD（GET 列表 / GET 详情 / POST / PATCH / PATCH /status / DELETE 软删），含：
    - SubscriptionGuard：单事务 SELECT FOR UPDATE subscription（status='active' AND grace_ends_at/expires_at > now）+ COUNT + INSERT，避免并发 race + 过期套餐绕过
    - OperationLog：location_created / location_status_changed / location_deleted 三个生命周期事件
  - 错误码新增 10 个：BRAND_NOT_ACTIVE / STEP_NOT_SKIPPABLE / INVALID_STEP_KEY / ONBOARDING_NOT_READY / LOCATION_NAME_DUPLICATED / LOCATION_NOT_FOUND / QUOTA_EXCEEDED / SUBSCRIPTION_RESTRICTED / BRAND_CODE_DUPLICATED / INVALID_PARAM
  - AppError 扩 `Details map[string]any`，response.Error 统一序列化进 Response.Data
  - JSONB BeforeCreate 清债：补 PaymentTransactionModel / SaaSPlanOrderModel / OperationLogModel / BrandOnboardingStepModel 兜底 hook
  - Wire 重生：之前手改 `wire_gen.go` 全部 `go generate` 干净
- 前端 8 commits（`5bf6c6d..3d46fbb` on `dev`）：
  - 路由分组 `(protected)/onboarding/`：layout 守卫 + 8 个子路由（brand-profile / locations / 动态 [stepKey] / complete）
  - WizardShell 进度组件 + StepPlaceholder 占位（第 3-8 步 + 跳过按钮）
  - LocationFormDialog（RHF + Zod）+ LocationStatusToggle（按 Q3 决定用既有"按钮 + ConfirmDialog"模式而不是 shadcn switch）
  - 工作台首页进度条卡片（onboarding 未完成时显示，已完成自动隐藏）
  - 支付页 paid → /login?next=/onboarding

Post-impl code-review：7 finder 并行（行扫描 / 删除行为 / 跨文件追踪 / 复用 / 简化 / 效率 / 高度），dedupe 26→10 finding：
- 7 项 P0/P1/P2 correctness 本批修完（commits `4b0dcd7` / `ad11205` / `6dee9c9`）：
  - A1 Complete 非原子 → CompleteOnboarding 单事务
  - B2 quota 忽略 expires_at → 加 grace_ends_at/expires_at 校验
  - C1 completed step 不持久化 → EnsureStepCompleted upsert
  - C2 7 张 COUNT status 过滤不一致 → 全部加 active / status 过滤
  - A5 UpsertSkippedStep 不清 completed_at → ON CONFLICT 加 nil 重置
  - A2 QUOTA_EXCEEDED gin.H envelope bypass → 统一走 response.Error + AppError.Details
  - A3 skipStep 静默吞 bind 错 → 加 ContentLength check + 返 400
- 9 项 architectural / cleanup 转 `backend/.learnings/FEATURE_REQUESTS.md` "Batch 4 code-review 转移项"，下批起飞前清。

验收（业务流通过）：
- 登录 brand → 自动跳 onboarding → 填资料 → 创 Location → 跳过第 3-8 步 → complete → 跳工作台
- PATCH /brand/profile 真实落库（含 description）
- step 状态实时反映；progress 8/8 后 brand.onboarding_status = completed

本批踩坑：
- migration 000004 需在本地 DB 手动跑（`Makefile` 假设用户是 `postgres`，但开发机实际是 `liushan` → `make migrate-up` 静默失败）。Fix：直接 `psql -d mini_schedule -c "ALTER TABLE brands ADD COLUMN IF NOT EXISTS description VARCHAR(2000);" + UPDATE schema_migrations`。建议 `.learnings` 记录"自动 migration on server boot" 选项。

待完善：
- Edge cases（E1-E22）未自动化 Playwright，仅 Happy Path 通过
- E24/E25 quota 并发竞争测试需本地 pg 真测，sandbox 无法 testcontainer
- mini_program_qrcode 步骤无实际后端，只能跳过；后续小程序集成 batch 落地后再做

### Batch 4.5：Migration auto-apply on boot ✅

详细契约：[pds/batches/batch-04.5-migration-autoboot.md](batches/batch-04.5-migration-autoboot.md)

契约状态：**已完成**（2026-06-06 人工验收通过）

完成内容（backend dev 4 个 task commit `13bdcd2..696d02d`）：
- `migrations/embed.go`：`//go:embed *.sql` 把 migration 文件打进二进制
- `internal/infrastructure/database/migrate.go`：基于 `golang-migrate/v4` + `source/iofs` 的 `RunMigrationsUp`，dirty 状态拒启动
- `config.Database.AutoMigrateOnBoot` 配置项 + env 绑定 + 4 个 dev yaml 默认 true
- 三个 `cmd/api-*/main.go` 在 Wire 之前调 `RunMigrationsUp`，失败 `log.Fatal`
- Makefile `DATABASE_URL` 兜底用 `${PG_USER:-${USER}}`，不再硬编码 `postgres:postgres`

验收（本地手测全过）：
- v3 → v4：日志 `migrations: applied from_version=3 to_version=4`
- 已最新：日志 `migrations: schema up to date version=4`
- dirty=true：boot 报错并拒启动
- 默认 OS user 的 `make migrate-up` 直接可跑

本批踩坑：
- `//go:embed *.sql` 在 `migrations` 包内时，文件挂在 embed.FS 的**根**，`iofs.New(fs, "migrations")` 报 `open migrations: file does not exist`。正确写法 `iofs.New(fs, ".")`。

### Batch 5：Staff + InstructorProfile + 角色 seed + SubscriptionGuard 重构 ✅

详细契约：[pds/batches/batch-05-staff-instructor-roles.md](batches/batch-05-staff-instructor-roles.md)
测试场景：[pds/batches/batch-05-staff-instructor-roles-tests.md](batches/batch-05-staff-instructor-roles-tests.md)
Plan 文件：[pds/batches/batch-05-staff-instructor-roles-plan.md](batches/batch-05-staff-instructor-roles-plan.md)

契约状态：**已完成**（2026-06-06 业务验收通过）

完成内容：
- 后端 15 commits（`6921def..97a33bb` on `dev`）：
  - migration 000005：13 个 permission code（brand/location/staff/instructor 4 域）+ 8 个预置 role_templates + role_template_permissions 映射 + brand_users.is_owner 列
  - `internal/audit/` pkg：统一 OperationLog 写入入口，actor_type 枚举（brand_user/platform_admin/system），用 `tx.Table().Create(map)` 避免循环依赖（清债 Batch 4 FR）
  - `internal/application/commercial/subscription_guard.go`：抽出参数化 ResourceKind（Location/Staff/Learner）的 quota guard；Location 改造复用（清债 Batch 4 FR）
  - Staff 四层：domain/staff + role + instructor / application/staff（含 service.go + role_allocator.go）/ infrastructure/persistence/{staff,role,instructor}_repository.go + models / interfaces/brand/staff_handler.go（11 endpoints）
  - role_allocator：`EnsureBrandRolesSeeded` 从 role_templates 复制 8 个 brand_roles + `AssignDefaultOwnerRoles` ON CONFLICT DO NOTHING 幂等
  - 注册流程改造：`commercial.Service.CreatePublicSignupOrder` 创建 owner 时同步分配品牌负责人角色
  - admin backfill 接口 `POST /api/v1/admin/system/backfill-owner-roles`（一次性给现存 brand owner 补角色）
  - 错误码扩 7 个：STAFF_PHONE_DUPLICATED / STAFF_NOT_FOUND / OWNER_PROTECTED / ROLE_NOT_FOUND / LOCATION_ASSIGNMENT_INVALID / INSTRUCTOR_PROFILE_NOT_FOUND / INSTRUCTOR_PROFILE_CONFLICT
  - Wire 三个 wire_gen.go 干净重生
- 前端 9 commits（`a05b76a..a3af71a` on `dev`）：
  - F01 types + 4 个 api client + 错误码常量
  - `/staff` 列表（DataTable + 搜索 + 状态过滤）
  - `/staff/[id]` 详情页（基础信息 + 角色任职 + Location 任职 + Instructor 折叠卡）
  - StaffCreateDialog（RHF + Zod，role 多选 chip + location 多行 + QUOTA_EXCEEDED toast+inline 双展示）
  - StaffRoleAssignmentEditor + StaffLocationAssignmentEditor（PUT 全量替换；scope-aware；is_primary 单选；owner 隐藏编辑入口）
  - InstructorProfileSection 折叠卡（csv 文本框；启用/编辑/注销）
  - onboarding 第 3 步真实化 + 工作台菜单加"员工管理"（不加角色管理）
  - 验收期 bug 修：详情页 `staff.{role,location}_assignments ?? []` 兜底

Post-impl code-review：3 finder 并行（backend correctness / frontend correctness / 架构回归），20 个候选 dedupe → 6 项当批合（commits `9cd2807` + `36bd170`）：
- B2 staffRepository.Update 加 audit.Write
- B3 RoleAssignmentEditor 对 owner 隐藏编辑入口 + OWNER_PROTECTED 错误处理
- B4 service.ReplaceRoleAssignments 对 owner 拒绝（阻断 brand_admin 静默清空 owner 的 brand_owner 角色这一权限提升漏洞）
- B5 ReplaceRoleAssignments / ReplaceLocationAssignments 加 SELECT FOR UPDATE on brand_user 行
- B8 instructor unique violation 改用 INSTRUCTOR_PROFILE_CONFLICT
- B11 StaffCreateDialog 在非 quota 错误分支清掉 quota counter

11 项 architectural / cleanup 转 `backend/.learnings/FEATURE_REQUESTS.md` "Batch 5 code-review 转移项"。

验收（业务流通过）：
- 登录 owner → 员工管理列表显示 owner 角色"品牌负责人" → 详情页角色任职**无编辑按钮**（review B3）
- 新增员工：phone/name/password + 角色"店长" + Location1 manager primary → 列表新增 + 详情页"晋升为教练" → 填资料保存 → 列表显示教练标识
- 进 /onboarding → 第 3 步 staff 显示 completed
- backfill 接口幂等：第一次 `processed:1`，第二次 `skipped:1`

本批踩坑：
- `Staff.RoleAssignments` / `LocationAssignments` 用 `json:",omitempty"`，owner 等无任职场景下整个字段从 JSON 丢失，前端 `.map()` 炸 → 前端类型说 `string[]` 但运行时 undefined。修：后端去 omitempty + toStaffDomain 把 nil 规整为空切片；前端两个 editor 加 `?? []` 兜底。规则：**任何前端会迭代的数组字段都不要 omitempty**
- handler binding `Specialties / Certificates string` vs 前端发 `string[]` → Gin ShouldBindJSON 类型不匹配返通用 INVALID_REQUEST，看不出根因。修：handler 接 `[]string` + joinCSV→DB string；GET 用 embedding struct splitCSV→[]string

待完善：
- 11 项 review 转 FR（含 service.Create 原子性、OwnerRoleAllocator ctx 参数、providePublicHandler 真重构、ResourceStatusToggle 泛型抽出等），下批起飞前清
- E2E Playwright 未自动跑（前端 agent 加了 data-testid 钩子）
- E10 并发 staff quota race 需本地 pg 真测

### Batch 6：RBAC enforcement + data_scope 落地 ✅

详细契约：[pds/batches/batch-06-rbac-enforcement-data-scope.md](batches/batch-06-rbac-enforcement-data-scope.md)
测试场景：[pds/batches/batch-06-rbac-enforcement-data-scope-tests.md](batches/batch-06-rbac-enforcement-data-scope-tests.md)
Plan 文件：[pds/batches/batch-06-rbac-enforcement-data-scope-plan.md](batches/batch-06-rbac-enforcement-data-scope-plan.md)

契约状态：**已完成**（2026-06-11 业务验收通过）

完成内容：
- 后端 10 commits（`2bdc02f..113317f` on `dev`）：
  - domain/rbac：`PermissionSet`（Has/HasAll/Codes/Expand，edit/create→view、delete→view+edit 隐含，内存推导不落库）+ `DataScope`（Kind + LocationIDs，MergeScopes union，all_brand 胜）
  - infrastructure/persistence/rbac_repository：`LoadEffectiveRaw` 单条 SQL JOIN 三表（brand_role_permissions / brand_user_role_assignments / brand_roles）+ data_scope 推导（role_default → all_brand / assigned_locations）
  - application/rbac/checker：`Resolve`（ctx-cache → Redis L1 60s TTL → DB）+ `Require` + owner fast-path；`ScopeResolver.ApplyToQuery`（DataScope → GORM where）
  - service 层校验（非 middleware）：staff/location/onboarding/brandprofile 每个 service 方法前置 `require(code)`；checker==nil 时 bypass（兼容 bootstrap）
  - T07 data_scope 收紧：staff/location 的 list+detail+write 路径按 assigned_locations 过滤；列表 EXISTS/IN 子查询，详情/写路径 out-of-scope 返 404（不泄漏存在性）
  - GET /me/permissions 独立 endpoint：序列化 `permissions[]` + `data_scope {kind, location_ids?}`
  - 错误码扩 1 个：PERMISSION_DENIED（HTTP 403）
- 前端 4 + 2 commits（`b05b4f1..8c4efef` + 验收期 2 修 on `dev`）：
  - F01 `usePermissions` hook + Context Provider + PERMISSIONS 常量；fail-closed（load 失败/loading → has() 全 false → 按钮默认 disabled）
  - F02 菜单按权限隐藏（`/staff` 需 staff.view；未列入的旧菜单默认可见）
  - F03 操作按钮按权限 disabled + tooltip
  - F04 全局 PERMISSION_DENIED toast handler
  - 验收期修 1（`e71781d`）：disabled 元素不派发指针事件 + shadcn `disabled:pointer-events-none` → 原生 title tooltip 永不弹。新增 `Hint`（Radix tooltip 包裹器，触发器挂非 disabled span + `[&_:disabled]:pointer-events-none`），替换 5 处失效 title=
  - 验收期修 2（`69f513c`）：登出再登录后权限菜单仍停留在上一用户（见下"本批踩坑"）

本批踩坑：
- **跨用户缓存泄漏**：`/me/permissions` 等 query key 静态（不带 user id），logout 只清 auth state 不清 React Query 缓存，登录 onSuccess 只 invalidate `['auth']` → 新用户命中旧缓存、60s staleTime 内不重新请求，必须强刷才更新。修：三端登录 onSuccess 改 `queryClient.clear()` + 登出处 `queryClient.clear()`。规则：**会话边界（登入/登出）必须清空整个 query cache**，否则所有非 user-keyed 缓存（权限/staff/门店）跨用户泄漏
- **disabled 控件 native title tooltip 永不弹**：disabled 元素不派发鼠标事件 + shadcn Button 自带 `disabled:pointer-events-none`。修：tooltip 触发器挂到非 disabled 的 span 包裹器上，强制内部 disabled 子元素 pointer-events:none

待完善（转 FR）：
- T08 延后：`GET /roles/:code` 单条角色详情 + `GET /permissions` 全量权限列表（本批只做 /me/permissions，自定义角色 CRUD 留 Batch 7）→ ✅ Batch 7 已补
- T10 完整回归（35 个测试场景 H1-H6/E1-E35 的端到端 Playwright）未自动跑
- 品牌自定义角色 / 调整权限 CRUD 仍留 Batch 7 → ✅ Batch 7 已完成

### Batch 7：品牌自定义角色 / 调整权限 CRUD ✅

详细契约：[pds/batches/batch-07-custom-roles.md](batches/batch-07-custom-roles.md)
测试场景：[pds/batches/batch-07-custom-roles-tests.md](batches/batch-07-custom-roles-tests.md)
启动预备：[pds/batches/batch-07-custom-roles-prep.md](batches/batch-07-custom-roles-prep.md)

契约状态：**已完成**（2026-06-12 端到端验收通过：E1–E8 + E4-编辑变体 + Happy #7/#8/#10 全绿）

grill 设计树定论：A1 系统角色完全只读（可复制为自定义）｜A2 code 系统生成（前端只填 name）｜A3 scope_type 创建后锁定｜A4 有任职引用禁止删（`ROLE_IN_USE`）｜**B1 增量提权校验**（update 只对新增权限做 ⊆ actor 校验，保留/移除既有权限放行，owner 例外）｜C1 改角色后主动批量失效持有者缓存｜gate 新增 `role.manage`。

完成内容：
- 后端（`dev`，commits `ce5624e..e17cdd0`）：
  - migration `000006`：seed `role.manage`（domain=role）+ 映射 brand_owner/brand_admin 模板 + **backfill 存量 brand**（owner 走 fast-path 自动有，brand_admin 必须 backfill）
  - 错误码扩 4 个：`ROLE_IS_SYSTEM` / `ROLE_IN_USE`（均 409）/ `ROLE_PERMISSION_EXCEEDS_ACTOR`（403，data.exceeded）/ `ROLE_CODE_DUPLICATED`（409）
  - repo：CreateBrandRole（gen `custom_<hex>`，is_system=FALSE，原始 code 不展开）/ UpdateBrandRole（事务全量替换权限）/ status / delete / CountAssignmentsByRole / ListBrandUserIDsByRole / ListRolePermissionCodes（B1 增量 diff）
  - service：CRUD 全 gate `role.manage`；is_system 拦截、OWNER_PROTECTED 优先、A4 引用检查、B1 增量校验、A3 scope 锁定、C1 post-commit `checker.Invalidate` 批量失效
  - handler：`GET /permissions`（按 domain 分组）、`GET /roles/:code`、`POST/PUT/PATCH status/DELETE /roles`
  - 验收期修：`isUniqueViolation` 改 pgconn SQLSTATE 23505（修复手机号重复返 500 的线上 bug，~11 处调用点受益）
- 前端（`dev`，commits `26a6d4f..eabac0f`）：
  - `/roles` 角色管理页（列表 + 系统/自定义 badge + 权限 gate 按钮）
  - 角色编辑器 Dialog（create/edit/copy，权限勾选树按 domain 分组，scope 编辑锁定，超 actor 权限项 disable+Hint）+ 系统角色只读视图
  - `packages/api/roles.ts` CRUD hooks（mutation invalidate `['brand-roles']`）；`packages/types` 加 CreateRoleInput/UpdateRoleInput/PermissionGroup；`PERMISSIONS.ROLE_MANAGE`；工作台「角色管理」入口
  - 验收期修：staff 列表失效用 `refetchType:'all'`（删员工后列表 stale 直到硬刷）；角色状态切换补 `ROLE_NOT_FOUND` 处理

C1 验证证据（Redis 层）：员工预热 `rbac:perms:23` EXISTS=1 → owner PUT 去权限 200 → 瞬间 EXISTS=0（role→users 反查 DEL）→ 员工 60s 内重查权限即少一项，不依赖 TTL。

本批踩坑：
- **`isUniqueViolation` 字符串前缀匹配在 pgx 驱动下漏判**：pgx 错误串 `...(SQLSTATE 23505)` 无 `ERROR:` 前缀也不以 `duplicates` 结尾 → 唯一冲突误判为非冲突 → 业务错误降级成 500。修：`errors.As(*pgconn.PgError)` + code 23505。
- **旧二进制掩盖新逻辑**：验收时 :8081 跑的是改码前的 `go run` 进程，B1 增量逻辑"看似失效"，重启后端即正常。验收前务必重建/重启后端。

待完善（转 FR）：
- `location.view` 前端无可见菜单/按钮门（`/locations` 管理页本身未建，Batch 4 FR 4.2 遗留）；Happy #7/#8 的"location 入口随权限消失"目前无 UI 体现，C1 仅在 API+Redis 层验证。决策：建 `/locations` 页时一并补 location.view 门。
- 端到端 Playwright 回归（T11）仍为手动 Chrome 验收，未脚本化。
- code-review 转移项：见 backend/web `.learnings/FEATURE_REQUESTS.md`（共享 isUniqueViolation 已修；GetRole 双查、缓存逐 key DEL、:id 参数命名、brand_owner 检查散落等）。

### Batch 8：品牌门店管理页 `/locations` + location.view 门 + Playwright 回归 ✅

详细契约：[pds/batches/batch-08-locations-page.md](batches/batch-08-locations-page.md)

契约状态：**已完成**（2026-06-12 端到端验收通过：Playwright `e2e/batch-08-locations.spec.ts` 6/6 连跑两次稳定）

背景：补齐 Batch 7 遗留的 `location.view` 前端门 + Batch 4 FR 4.2 的 `/locations` 管理页。**无后端改动**（location 后端 Batch 4 已完整：CRUD + 额度 guard + audit + 软删 + data_scope + 唯一名）。

grill 决策：Playwright 真实跑通整个栈（真登录 owner + 真 CRUD + 真 DB + teardown 软删）｜门店删除引用保护转 FR｜UpdateStatus 保持 location.edit 门（toggle_status 暂闲置，转 FR）。

完成内容：
- 前端（`dev`，commits `5f62d5d..e698b8a`）：`/locations` 独立管理页（表格 name/address/phone/status badge + 状态筛选 + 分页 + 空状态）；行操作 编辑/停用切换(`LocationStatusToggle`)/删除(`ConfirmDialog`)，全部按 location.create/edit/delete gate + Hint；导航「门店管理」入口 + `NAV_HREF_PERMISSIONS['/locations']=location.view`（= Batch 7 缺的 location.view 可见门）。复用既有 location API hooks/类型/form-dialog/status-toggle，无新增。
- e2e（`dev`，commits `9b947e9..2a85cad`）：`web/e2e/batch-08-locations.spec.ts` 真实栈回归 H1–H5+E1，自带 teardown 软删清理。
- **验收期修 2 个共享底层 bug**（e2e 跑出来的，非门店功能本身）：
  - `fe27ace` **API client 不认 204 空 body**：`client.ts` 无条件 `response.json()`，后端 DELETE 返 204 → SyntaxError → **所有 DELETE（staff/locations）静默失败**（弹窗不关、toast 删除失败，但后端实际已删）。修：先 `response.text()`，空 body ok→undefined、非 ok→通用错，非空再 parse。三端共用，向后兼容。
  - `5b5001d` **硬 URL 水合竞态**：deep-link/刷新 protected route 时 zustand persist 未 rehydrate → `isAuthenticated` 瞬时 false → layout 跳 /login → middleware 拿 cookie 弹回 /dashboard。修：`packages/api/auth.ts` 加 SSR-safe `useAuthHydrated()`（初值 false，仅 client effect 读 persist），layout 跳转 effect + null gate 等 hydrated 后再判。顺带给 `packages/api` 补 react peerDependency（一直经 react-query 隐式依赖）。

本批踩坑：
- **dev server 不全量 type-check，`build` 才会**：`useAuthHydrated` 直接 `import 'react'` 在 dev 下能跑、e2e 也过，但 `packages/api` 没声明 react 依赖 → 生产 `pnpm build` 报 `Cannot find module 'react'`。教训：共享包新增直接依赖要同步 package.json；验收除 e2e 外必须跑一次 prod build。
- ConfirmDialog 确认按钮文案是「删除」「停用」（actionLabel）而非「确定」，且与触发按钮同名 → e2e 确认点击必须 `getByRole('dialog')` scope。

待完善（转 FR，见 web `.learnings/FEATURE_REQUESTS.md`）：
- `app`/`admin` 两端 protected layout 可能有同款水合竞态，本批只修了 brand。
- ~~门店删除引用保护（`LOCATION_IN_USE`）~~ → ✅ Batch 9 已做。
- `UpdateStatus` 改 gate `location.toggle_status`（现 location.edit，seed 的 toggle_status 闲置）。
- 后端 location list 加 name 搜索（前端目前只有状态筛选+分页）。
- 可选：加「店长权限门」e2e 用例（断言 location_manager 在 /locations 上写操作按钮全 disabled）。

### Batch 9：门店删除引用保护（LOCATION_IN_USE）✅

详细契约：[pds/batches/batch-09-location-delete-guard.md](batches/batch-09-location-delete-guard.md)

契约状态：**已完成**（2026-06-12 端到端验收通过：`e2e/batch-09-location-delete-guard.spec.ts` G1+G2 全过）

背景：Batch 8 转 FR。门店软删（UPDATE deleted_at）不触发 FK 行为，留下悬空引用。镜像 Batch 7 A4 补引用保护。

grill 决策：阻止删除的 active 引用 = 员工任职(`staff_location_assignments`) + 门店级角色任职(`brand_user_role_assignments.location_id`)；不前向检查 class_sessions/recurring_schedules（现无 CRUD/无数据）；active 状态过滤（两表硬删重插，只存 active 行）。

完成内容：
- 后端（`dev`，commits `e1bb789..8cd46c9`）：错误码 `LOCATION_IN_USE`(409)；repo `CountActiveReferences(brandID, locationID)`（两表 COUNT 求和，brand_id 隔离，status='active'）+ DB 单测（员工/角色引用、跨租户隔离、inactive 排除）；service.Delete 在 scope guard 后、SoftDelete 前插引用检查 + fake-repo 单测（拒删/放行）。
- 前端（`dev`，commit `bef33d0`）：`errors.ts` 加 `LOCATION_IN_USE` + locations 页 confirmDelete 加 case → toast「该门店仍有员工任职或角色绑定，请先移除后再删除」，弹窗保持打开（同 ROLE_IN_USE）。
- e2e（`dev`，commit `8fc6021`）：`batch-09-location-delete-guard.spec.ts`，UI 登录拿 token → page.request API 建门店+派员工 → G1 删被拦（toast+弹窗不关+行还在）、G2 移除任职后删成功，API teardown。

转 FR：课程/场次批次落地后把 class_sessions/recurring_schedules 纳入同一 `CountActiveReferences`。

### Batch 10：小债清扫（4 项）✅

详细契约：[pds/batches/batch-10-debt-sweep.md](batches/batch-10-debt-sweep.md)

契约状态：**已完成**（2026-06-15 验收通过：e2e 1/1 + 手动 2 项）

技术债打包清扫，来源 Batch 7/8/9 验收期 FR：
- **T01 app/admin 水合竞态推广**（前端 `e16a375`）：`apps/app/.../layout.tsx` + `apps/admin/.../admin-guard.tsx` 接 Batch 8 的 `useAuthHydrated()`，消除 hard-load 误跳 /login（admin 守卫确有同款 race）。验收：owner deep-link+刷新 /locations 采样 300+ 次无一次跳 /login。
- **T02/T06 location name 搜索**（前端 `ab48747` + 后端 `a403be5`）：后端 list 加 `q`（`name ILIKE '%q%'`，参数化）+ DB 单测；前端门店页加搜索框 + `useBrandLocations` 加 q。验收：搜索框 + 直连 API 同结果，确认 server-side 过滤。
- **T03 GetRole 单查**（后端 `ff7a807`）：新 `GetBrandRoleWithPermissions`，去掉 GetRole 的 O(N) ListBrandRoles 扫描。
- **T04 Checker.InvalidateMany**（后端 `d684667`）：角色缓存批量失效改单次多 key DEL（`cacheStore.Del` 加 variadic）；`invalidateRoleHolders` 改用。
- **T05 role.IsOwnerRole()**（后端 `3c5ec78`）：收敛散落的 `"brand_owner"` 字面量；error-code 优先级（OWNER_PROTECTED 先于 ROLE_IS_SYSTEM）保持不变。
- **T07 只读权限门 e2e**（`ee67e62`）：`batch-10-location-permission-gate.spec.ts`，只读账号 13900139777 在 /locations 写按钮全 disabled。

T03/T04/T05 为行为不变重构（旧 RBAC 单测全绿，未改断言）。push：backend `a403be5`(dev)、web `ee67e62`(dev)、pds 本次。

### Batch 11：课程模板 + 单场次排课（CourseCategory / CourseTemplate / ClassSession）✅

详细契约：[pds/batches/batch-11-course-template-session.md](batches/batch-11-course-template-session.md)
测试场景：[pds/batches/batch-11-course-template-session-tests.md](batches/batch-11-course-template-session-tests.md)

契约状态：**已完成**（2026-06-16 端到端验收通过：`e2e/batch-11-course-session.spec.ts` 6/6 全绿）

grill 设计树定论（用户拍板，均推荐项）：D1 范围切「分类 + 模板 + 单场次」，推迟 RecurringSchedule 循环排课 + Location Resource 资源管理到 Batch 12｜D2 放开 `courses.difficulty/type` NOT NULL，新模板用通用 `level_label`（丢弃健身枚举）｜D3 原地替换 legacy brand `/courses`（api-app C 端只读 ListPublished 不动），状态词沿用 draft/published/archived｜D4 不挂额度门（blueprint §4.1）｜D5 session 接 data_scope（all_brand / assigned_locations），own_sessions 转 FR｜D6 class_sessions 纳入门店删除 guard。

完成内容：
- 后端（`dev`，commits `fb590ea..dfda127`）：
  - migration `000007`：difficulty/type DROP NOT NULL + courses_status_valid CHECK + 9 细粒度权限码（course_category.* / course.create·edit·delete / session.view·create·cancel）+ role_template 映射 + 存量 brand backfill（镜像 000006，template_id JOIN）
  - 错误码扩 11 个（CATEGORY_NOT_FOUND / CATEGORY_NAME_DUPLICATED / COURSE_NOT_ACTIVE / COURSE_IN_USE / COURSE_LOCATION_UNAVAILABLE / SESSION_NOT_FOUND / SESSION_TIME_INVALID / SESSION_INSTRUCTOR_CONFLICT / SESSION_CANCEL_NOT_ALLOWED / INSTRUCTOR_NOT_SCHEDULABLE，COURSE_NOT_FOUND 复用）
  - 三个新域 coursecategory / coursetemplate / classsession（domain + application + persistence + handler）：CourseCategory CRUD；CourseTemplate CRUD（分类绑定 + course_location_availability 内联硬删重插，批量 list 避 N+1，gorm.DeletedAt 软删，CountScheduledSessions guard）；ClassSession 单场次（tx 内校验 course published / 门店可用 / 教练可排课，落 scheduled，教练时段重叠 DB EXCLUDE 23P01→SESSION_INSTRUCTOR_CONFLICT 不裸 500，list/detail JOIN 反范式名 + data_scope，cancel）
  - brand handler 退役 legacy `/courses` 写路径 + 注册 3 新 handler，wire 干净重生（api-app 不动）
  - onboarding CourseTemplate 计数 `active`→`published` 对齐；location CountActiveReferences 纳入 scheduled/in_progress class_sessions
  - 11 个 repo DB 单测（course template + class session 全路径，含 23P01 冲突、软删、引用 guard）
  - 验收期补 `GET /api/v1/brand/instructors?schedulable=true`（commit `dfda127`）：前端排课弹窗「可排课教练」下拉依赖此端点，后端原缺失→下拉恒空→排课阻断（见下踩坑）
- 前端（`dev`，commits `708a4e2..417c884`）：
  - types + 11 错误码常量 + 9 PERMISSIONS 常量；3 个 api client（course-categories / courses / class-sessions）
  - `/course-categories`（表+状态切换+dialog）；`/courses`（替换 legacy 健身页：表/分类 chips/级别/时长·容量/可用门店数/状态，发布·归档/删除 COURSE_IN_USE，分类+门店多选 dialog）；`/courses/[id]`（详情+近期场次+编辑）；`/schedule`（场次表+门店/状态筛选，排课 dialog 门店→已发布课程→可排课教练→日期·时长，冲突 inline+toast，取消）
  - 导航加「课程分类/课程模板/排课」+ NAV_HREF_PERMISSIONS 门；onboarding 第 4/5/7 步真实 CTA
  - e2e `batch-11-course-session.spec.ts` 真实栈 H1/H3+H4/H5/E7/onboarding/E11，API teardown 自清理

Post-impl code-review（high）：2 个 confirmed bug 当批修：
- 后端 resolveLocationIDs：更新时传空 location_ids 误回填全部门店（应尊重「清空」）→ 加 defaultAllWhenEmpty，仅 create 默认全选
- 前端课程 dialog：默认全选门店的 effect keying on locationIds.length，create 模式取消最后一个会被自动回填→改 ref 一次性默认

验收（端到端通过）：
- e2e 6/6：建分类→建课程（绑分类+默认全选门店）→发布→排单场次→同教练同时段冲突拦截→onboarding 4·5·7 completed→只读账号 13900139777 写按钮全 disabled
- 后端 live curl 烟测同样全过（COURSE_NOT_ACTIVE / SESSION_INSTRUCTOR_CONFLICT / COURSE_IN_USE / cancel→204）

本批踩坑：
- **前端假设的端点后端没实现**：`packages/api/src/instructor.ts` 早标注 `ASSUMPTION (backend must match): GET /instructors?schedulable=true`，后端无此路由→排课教练下拉恒空→H5/E7/onboarding 全阻断；数据本身正常（张三 id=1 active+schedulable）。我的后端 curl 烟测**硬编码 instructor_profile_id=1 绕过了下拉**，所以没暴露，是 e2e 跑通 UI 全链路才抓到。教训：契约里「前端依赖的后端端点」必须显式列入 API 接口表并后端落地；纯 API 烟测不能替代走 UI 选择器的 e2e。
- restart 后端用最新二进制（旧二进制掩盖改动）；前端 filter 用包名 `@mini-schedule/brand`（`--filter=brand` 报 No package found）；stale `.next` 致登录不跳转/退化成原生 GET，须 rm -rf .next 重启 + curl 预热。

待完善（转 FR，见各仓库 `.learnings/FEATURE_REQUESTS.md`）：
- RecurringSchedule 循环批量排课 + recurring_schedule_weekdays（Batch 12）
- Location Resource CRUD + 资源时间冲突（session.location_resource_id 本批恒 null，Batch 12）
- `own_sessions` 数据权限（教练只看自己授课场次）
- 场次改期（reschedule）；课程分类 DELETE 接口（当前仅停用，e2e teardown 只能置 inactive）
- Booking/Waitlist/Attendance/Entitlement（学员预约批次）

### Batch 12a：Location Resource 资源管理（拆自 Batch 12，先做）✅

详细契约：[pds/batches/batch-12a-location-resource.md](batches/batch-12a-location-resource.md)
测试场景：[pds/batches/batch-12a-location-resource-tests.md](batches/batch-12a-location-resource-tests.md)

契约状态：**已完成**（2026-06-16 会话内业务验收通过：后端契约——错误码分流 / 容量优先级 input>资源>课程 / 反范式字段 / 软删 / 跨引用拒删——与前端 UI——toast 文案 / 弹窗不关 / 按钮门控 / 下拉过滤 / 容量自动填充——全部对齐）。契约 + 验收均会话内确认。

grill 定论（用户拍板，均推荐项）：Batch 12 拆 12a（资源管理，本批）+ 12b（循环排课）｜资源端点扁平 `/location-resources?location_id=`｜新增 `location_resource.view/create/edit/delete` 4 码 + role 映射 + backfill（migration 000008，不动表结构）｜资源 data_scope 接 assigned_locations｜软删带引用保护 RESOURCE_IN_USE（被 scheduled/in_progress 场次或 active 循环排课引用时拒删）｜资源/教练冲突按约束名分流（新 `SESSION_RESOURCE_CONFLICT`）｜单场次可选绑资源 + 容量优先级 input>资源>课程默认｜门店删除 guard 纳入 active 资源｜前端新建 `/resources` 导航页 + 排课弹窗加资源级联下拉。

完成内容：
- 后端（`dev`，commits `da6f363..<fix>`）：migration `000008`（4 权限码 + role 映射 + 存量 backfill，不动表）；5 错误码（RESOURCE_NOT_FOUND/NAME_DUPLICATED/IN_USE/NOT_AVAILABLE + SESSION_RESOURCE_CONFLICT）；新域 `locationresource`（domain/service/persistence/handler，镜像 location/classsession，data_scope + 软删 + 反范式 location_name + Delete 引用保护）；class-session 绑资源（`isExclusionViolation`→`exclusionConstraint` 返约束名 + `sessionConflictError` 按 `class_sessions_resource_no_overlap` 分流资源冲突；容量优先级 input>资源>课程；LEFT JOIN resource_name）；location `CountActiveReferences` 纳入 active 资源；14 个新 DB/service 单测全绿。
- 前端（`dev`）：types + `location-resources` api client/hooks + 5 错误码 + 4 权限码 + nav「资源管理」+ NAV_HREF_PERMISSIONS 门；`/resources` 页（门店筛选+表+表单弹窗+停用切换+删除确认 RESOURCE_IN_USE）；排课弹窗资源级联下拉 + 选中自动填容量 + 新错误映射；/schedule 表加资源列。
- code-review（high，3 finder agents）：2 项当批修（资源 id<=0 归一未绑定；Update 显式 brand_id 防御性隔离）；其余转 FR（见下）。
- 验证：`go build ./...` + `go test ./...` 全绿（含 Postgres 集成）；`pnpm --filter @mini-schedule/brand lint + build` 全过。

待业务验收（e2e）：场景见测试文件 H1-H7 + E1-E10（建资源/重名/绑资源排课/资源冲突拦截/停用拒排/RESOURCE_IN_USE/只读 disabled/门店 guard）。e2e 由用户另开 session 跑真实栈。

code-review 转 FR（见各仓库 `.learnings`）：exclusionConstraint 空约束名退化路径（pgx 实际恒填，理论项）；Delete guard TOCTOU（与现有 LOCATION_IN_USE/COURSE_IN_USE 同类，DB 约束兜底重叠但不挡删后绑）；inactive 资源仍被未来场次引用（blueprint §20.5 设计：已排未来场次不自动取消，编辑场次时提示资源已停用——提示待实现）。

### Batch 12b：RecurringSchedule 循环排课（拆自 Batch 12，后做）✅

详细契约：[pds/batches/batch-12b-recurring-schedule.md](batches/batch-12b-recurring-schedule.md)
测试场景：[pds/batches/batch-12b-recurring-schedule-tests.md](batches/batch-12b-recurring-schedule-tests.md)
回归结论：[pds/batches/batch-12b-regression-result.md](batches/batch-12b-regression-result.md)

契约状态：**已完成**（2026-06-17 E2E 回归 17/17 全过：H1-H5 + E1-E11）。契约 + 验收均会话内确认。

完成内容：
- 后端（`dev`）：新域 `recurringschedule`（domain/service/persistence/handler）。**无 migration**（表 000003 已建，权限复用 `session.view/create/cancel`）。service 校验（weekday/XOR 结束条件/26 周·200 节上限/start_date≥今天/Asia/Shanghai +08:00 occurrence 生成）；repo 外层 tx 插 recurring+weekdays → 批级校验一次 → 逐 occurrence GORM 嵌套 tx(SAVEPOINT) 插场次，撞 EXCLUDE 按约束名分流 instructor/resource_conflict 跳过记清单、继续；0 成功整批 abort RECURRING_ALL_CONFLICT 不落空壳；复用 12a `exclusionConstraint`/`sessionRow`/容量优先级 input>资源>课程；非级联 cancel；门店 guard 纳入 active recurring。3 错误码。8 service + 7 DB 单测全绿。
- 前端（`dev`）：/schedule 加「单场次/循环排课」Tabs；循环弹窗（周几多选+起止二选一+资源级联+容量填充+生成结果清单）+ 列表/详情/非级联取消确认；复用 session.* 门控。
- code-review（high，3 agents）：3 修（repeat_weeks 不回填派生 end_date 保 XOR / 容量自动填充优先级守卫 / 结束条件客户端校验）；exclusionConstraint 退化沿用 12a FR。
- 验证：`go test ./...` + `pnpm --filter @mini-schedule/brand lint+build` 全过。
- 回归观察（非阻断）：>200 节守卫被 26 周上限遮蔽（182<200，纵深防御）；class-sessions list DTO 不暴露 recurring_schedule_id（GET :id 聚合正确）；全冲突错误内联呈现非 toast。

grill 定论：单外层 tx + 逐 occurrence SAVEPOINT 部分成功（冲突跳过返清单）｜0 成功生成时整批 abort 返 409 RECURRING_ALL_CONFLICT+skipped 不落空壳｜非级联 cancel（status→cancelled 不动已生成场次，blueprint 不做整批取消）+ 复用 session.view/create/cancel 权限（无新权限 migration）｜生成时区 Asia/Shanghai（per-brand TZ 转 FR）｜周几=time.Weekday 0-6｜结束条件 end_date/repeat_weeks 二选一｜上限 26 周 & 200 节｜容量 input>资源>课程｜门店删除 guard 纳入 active recurring_schedules。

### Batch 13：学员预约闭环（Entitlement / Booking / Waitlist / Attendance）⏳ 进行中

交接 prompt：[pds/batches/batch-13-handoff.md](batches/batch-13-handoff.md)

grill + 拆批拍板（2026-06-17 会话内，均推荐项）：

- **schema 现实**：18 张学员/预约表已在 `000003` 建好，**零代码**（brand `/users`、`/trainings` 是 legacy `app_users` / 健身训练记录，与 `brand_learner_profiles` 无关）。`SubscriptionGuard.ResourceLearner` / `MaxLearners` 已就绪。api-app C 端极薄（仅 `GET /courses`）。
- **拆批**：5 子批，单向依赖链 **13a 学员档案 → 13b 权益产品+发放 → 13c 预约下单+规则+场次取消级联回改 → {13d 候补, 13e 签到+履约+爽约}**。
- **先做哪端**：全程 brand 后台 staff_assisted（复用 RBAC/data_scope/audit/UI 零新基建）；C 端 api-app + 微信自助预约延后到独立批（Batch 14），届时只是 `source=learner_self_service` 的薄认证包装。
- **13a 范围**：核心档案 CRUD + 冻结/解冻 + 标签；无微信学员用合成 open_id `manual:<brand>:<phone>`（find-or-create by phone）；员工服务关系延后。
- **取消后重约**：bookings / waitlist_entries 的 `unique(session,learner)` 是全量唯一 → 13c 加 migration 改 **partial unique**（仅 active 状态），重约 INSERT 新行保留取消历史。
- **核心难点（13c）**：下单原子性——单 tx 内 `SELECT FOR UPDATE` session 行（booked_count<capacity）+ entitlement 行（lock 模型）+ INSERT booking(unique) + hold(unique)，行锁+unique+CHECK 兜超卖/抢最后课时。
- **跨批回改（13c 必做）**：B11/12b 的 `ClassSession.Cancel` 当前只置 `status=cancelled`，bookings 落地后必须级联 cancel 所有 active booking（cancel_source=session_cancelled）+ release holds + cancel waitlist。

#### Batch 13a：学员档案管理（BrandLearnerProfile + 标签）✅

详细契约：[pds/batches/batch-13a-learner-profile.md](batches/batch-13a-learner-profile.md)
测试场景：[pds/batches/batch-13a-learner-profile-tests.md](batches/batch-13a-learner-profile-tests.md)

契约状态：**已完成**（2026-06-18 会话内业务验收通过：e2e H1–H9 + E1–E11 全过，两条回归修复验证生效；仅 2 条非阻断文档/可选 finding）。

完成内容：
- 后端（`dev`，5 commits `117fb53..e3dea8c`）：migration `000009`（`learner.create/edit/delete/freeze` 4 写码 + role 映射仅 brand_owner/brand_admin + 存量 backfill，镜像 000008；`learner.view` 沿用 000003，粗码 `learner.manage` 转 inert）+ `000010`（`idx_brand_learner_profiles_brand_identity` 改 partial `WHERE deleted_at IS NULL`，修软删后无法重建）；6 错误码；新域 `learner`（domain/service/persistence/handler 镜像 location/locationresource）——Create 单 tx：`CheckAndCount(ResourceLearner)` 配额 → find-or-create identity by phone（合成 open_id `manual:<phone>`，并发回查）→ INSERT profile（两 unique 按约束名分流 `LEARNER_ALREADY_EXISTS`/`LEARNER_NO_DUPLICATED`）→ tag 硬删重插（校验 ⊆ brand active）→ audit；data_scope on `primary_location_id`（assigned 员工建档须指定 scope 内主门店）；软删 + `countLearnerActiveReferences`（active 权益 + 未结束预约，13b/13c 前恒 0）；标签 CRUD（品牌级，仅停用不硬删）。**DB 单测 11 + service 单测 10 全绿。**
- 前端（`dev`，3 commits `7848492..13f3a89`）：types + 2 api client（登记 exports）+ 6 错误码 + 5 权限码；`/learners`（搜索 q/状态/门店筛选 + 分页 + 行操作 gate+Hint + `LEARNER_IN_USE` 弹窗保持）+ LearnerFormDialog（手机号 create 必填/edit 锁定、主门店/学号/备注/标签 chip、QUOTA toast+inline）+ LearnerStatusToggle（freeze 门）+ `/learners/[id]` 详情（基础信息 + 标签 + 权益/预约/履约 Tab 空态占位）+ `/learner-tags` 页；导航 legacy `/users` 换真实 `/learners` + 新增 `/learner-tags` + `NAV_HREF_PERMISSIONS` 接 learner.view 门。

Post-impl code-review（2 review agent）→ 5 项当批修：
- P1 软删后无法重建（`brand_identity` unique 缺 `deleted_at IS NULL` filter）→ migration 000010 + 回归单测。
- P1 编辑弹窗主门店 seeding：无主门店学员在列表筛着门店时编辑被静默写入筛选门店；主门店已停用时下拉缺该选项静默清空 → 编辑只用学员当前值 + 并入当前（含停用）门店选项。
- P2 标签改名/改色失效 `brand-learners`/`brand-learner`（内嵌标签快照）；P2 inactive 学员隐藏冻结入口；P2 跨表失效。

验收（e2e 全过）：建分类档案/标签/筛选/权限门可视化走 UI；额度门 E5 / data_scope E6 / 无效标签 E4 走 API 直连。两条非阻断 finding：E5 `QUOTA_EXCEEDED` 实际 409（非契约误写的 403，已改文档对齐，自 Batch 4 约定）；E9 空手机号文案「请输入合法手机号」（已对齐测试文档）。

转 FR（见各仓库 `.learnings`）：学员列表搜索框 debounce；repo `UpdateStatus` 源态加固（service 已挡 inactive→frozen，repo 层未挡）；员工服务关系 `learner_staff_assignments`（顾问/主教练/跟进人，喂 instructor「自己相关学员」data_scope）；学员批量导入；微信 openid 回填合成身份；legacy `/users`(app_users) 页退役。下一子批：**13b 权益产品 + 发放**。

#### Batch 13b：权益产品 + 发放（EntitlementProduct / LearnerEntitlement / Transaction）✅

详细契约：[pds/batches/batch-13b-entitlement.md](batches/batch-13b-entitlement.md)
测试场景：[pds/batches/batch-13b-entitlement-tests.md](batches/batch-13b-entitlement-tests.md)

契约状态：**已完成**（2026-06-18 会话内业务验收通过：e2e 21 场景全过 + settle 落库 psql 实查确认；阻断缺陷 F1 已修并复验）。

完成内容：
- 后端（`dev`，6 commits `45ed444..f62c263`）：migration `000011`（复用 000003 的 `entitlement.view/manage/adjust`，仅补 §21.1 缺失的 3 条角色映射 brand_admin.adjust/course_operator.view/finance_support.adjust + 存量 backfill，**不加新码、不动表**）；7 错误码；新域 `entitlement`（domain/service/persistence/handler）——产品 CRUD（启停不软删；scope all/specific + location/course 关联硬删重插，**存在性校验**；count-based total 必填 / membership total NULL；限额 0→NULL；issued_count + location_ids/course_ids 批量回填避 N+1）；grant 单 tx（校验学员属本 brand + 产品 active，快照额度，expires=starts+validity_days，落权益 + grant 流水 + audit）；**settle 落库**（`SettleStatus` 纯函数 + 读触发 per-learner sweep + grant/adjust/status 后落库，无 cron）；adjust `SELECT FOR UPDATE` + remaining±delta（<0 或不限次→INSUFFICIENT）+ settle + 流水 + audit；status freeze/cancel/reactivate（cancelled 终态）；无 data_scope（品牌级，§21）。**DB 单测 18 + service 单测 12 + domain 单测全绿。**
- 前端（`dev`，3 commits `9a12a8f..16790c0`）：types + 2 api client（登记 exports）+ 7 错误码 + 3 权限码；`/entitlement-products` 产品页（表 + 状态筛选 + 启停 + 表单弹窗：type 锁定/total 仅 count-based/频次 0=不限/门店·课程 scope 多选）；学员详情「权益」Tab 落地（替换 13a 占位）：列表（剩余/总·不限次·到期·状态 badge）+ 发放弹窗 + 调整弹窗（±delta+原因，INSUFFICIENT inline）+ 冻结/作废/恢复 + 流水弹窗；导航「权益产品」+ NAV_HREF_PERMISSIONS。

Post-impl code-review（2 review agent，无 P0）→ 5 项当批修 + 验收期 F1：
- 后端 P2：门店 scope 校验放宽为存在性（同课程，避免编辑产品时停用门店/归档课程被强拒）；ListEntitlementsByLearner 加学员属本 brand 守卫→404；`SettleStatus` 加表驱动单测。
- 前端 P1：产品表单把当前 scope 里已停用门店/已归档课程并入选项（标「已停用/已归档」，可见可取消，镜像 learner-form）；发放后失效 `brand-entitlement-products` 修 issued_count 滞后。
- **F1（验收阻断，用户 session 定位+修，本端收编+补回归测）**：ListProducts 漏填 location_ids/course_ids（仅 GetProduct 填）→ specific 产品列表显示「0 门店」且从列表行编辑 scope 不回填→保存校验失败。修：加 `loadScopeIDs` 批量回填。

验收（e2e 21 场景全过）：产品建/改/启停、发放（次卡/会员卡不限次）、调整±/超额 INSUFFICIENT/不限次拒、冻结/作废/恢复、cancelled 终态、stale 产品 scope 编辑、issued_count 刷新；**settle 落库 psql 实查确认** expired/depleted 真写 DB；权限 backfill（course_operator/finance view、调整 403）。

转 FR（见各仓库 `.learnings`）：发放生效日时区（与 12b per-brand TZ 合并）；产品次数 zod min(0) 文案（仅顶层 apiError，无 inline）；权益到期/额度提醒通知。下一子批：**13c Booking 下单**（依赖 13b 权益 lock 模型——hold 复用 SettleStatus + `SELECT FOR UPDATE`）。

#### Batch 13c：预约下单 + 代取消 + 场次取消级联（Booking / EntitlementHold）✅

交接 prompt：[pds/batches/batch-13c-handoff.md](batches/batch-13c-handoff.md)
详细契约：[pds/batches/batch-13c-booking.md](batches/batch-13c-booking.md)
测试场景：[pds/batches/batch-13c-booking-tests.md](batches/batch-13c-booking-tests.md)

契约状态：**已完成**（2026-06-22 会话内业务验收通过：Happy + Edge + 级联 + 权限/越权全过；code-review P0 已修复并复验）。

完成内容：
- 后端（`dev`，6 commits `4d623ae..aabd7bf`）：migration `000012`（bookings/waitlist 全量 unique → partial `status<>'cancelled'`；**无权限迁移**——booking.* 三码及全角色映射 000003 base 已 seed 齐、存量已 copy）+ 14 错误码；新域 `booking`（domain/service/persistence/handler/wire 镜像 classsession/entitlement）——**TX-1 下单**单 tx：锁 session(scope/scheduled/窗口/容量) → 学员可预约 → auto(§5.7 纯函数排序)/manual(锁行全校验)/none 占位 → booked_count++ → INSERT booking(partial unique 分流 BOOKING_DUPLICATE) → 锁 entitlement(remaining--/locked++/re-settle) → hold + 流水(带 booking_id/hold_id；不限次 Δ0/balance NULL)；**TX-2 代取消**：锁 session+booking → allow_cancel/deadline → cancelled+退名额 → hold release(退额) / forfeit(release_on_cancel=false→consumed)；**TX-3 场次取消级联回改** `class_session_repository.Cancel`：加 SELECT FOR UPDATE + 级联 cancel active booking(session_cancelled)+release holds(恒退)+booked_count=0。统一锁序「先 session 后 entitlement」三事务串行化；频次叠加取最严(policy∪product，月限独家产品，UTC 日/周/月+concurrent，policy 级预检挡误报)；data_scope on session.location(越权 404)；brand-default 策略 GET/PUT(复用 schedule.view/manage)。**DB 单测 16 + service 单测 8 + domain 单测全绿。**
- 前端（`dev`，2 commits `71ec5ca..0fd1c4e`）：types + 2 api client（登记 exports）+ 14 错误码 + 权限码(BOOKING_*+SCHEDULE_*)；`/bookings`（DataTable+状态/待补权益筛选+分页+权限门，代预约弹窗：学员/场次选择器+权益模式 radio[自动预览/手动下拉/占位原因]+usable-entitlements+14 错误码 inline+代取消 ConfirmDialog）+ `/booking-policy`（brand-default 策略表单，可空限额留空=不限，gate schedule.manage 写）+ 学员详情「预约」Tab 替换 13a 占位 + 导航 + NAV_HREF_PERMISSIONS。跨查询失效 bookings/class-sessions/learner-entitlements。

Post-impl code-review（2 review agent）→ 1 项 P0 当批修：
- **P0 Policy 缺 json tag**：domain `Policy` struct 无 json tag + Gin 默认 encoding/json → `GET /booking-policy` 返回 PascalCase，前端按 snake_case 读全 undefined → 预约规则页读/写/存全坏（后端单测用 struct 直传未暴露，仅 HTTP 层现形）。修：10 字段补 snake_case tag（可空 *int 不加 omitempty 保留 null=不限）+ TestPolicyJSONSnakeCase 回归。

验收（全过）：代预约(自动/手动/占位)、列表/待补权益筛选、代取消、取消后重约(partial unique)、满员/并发抢名额/窗口/频次/重复/手动非法不回退/占位不绕容量、**场次取消级联 psql 实查**(全 booking cancelled+holds released+退额+booked_count=0)、权限门(course_operator 403)、越权 404、**release/不限次卡 hold psql 实查落库**；/booking-policy snake_case 响应复验。

转 FR（见各仓库 `.learnings`）：23514 CHECK 违反裸 500（正常路径行锁+预检已挡，建议加兜底分流，镜像 23505）；频次限额跨场次 TOCTOU（软策略，无数据损坏，需 advisory lock 才严格）；代预约弹窗关闭态仍拉一次 class-sessions（`useBrandClassSessions` 缺 enabled 门）；cancel_deadline 对员工代取消同样生效（员工绕过留 FR）；预约窗口/截止 per-brand 时区（合并 12b TZ FR）。**下一子批：13d 候补（waitlist）**——满员转候补 + 取消转正 + 扩 TX-3 级联取消 waitlist_entries。

grill 拍板（2026-06-22 会话内，5 决策点）：
- **策略配置**：仅 brand-default `GET/PUT` 单行 upsert（复用 `schedule.view/manage`，零新权限码）；per-location + 场次 override CRUD 延后，解析层仍读已存覆盖行。
- **权益选择**：自动选择（§5.7）+ 允许员工手动指定（仍全校验，非法报错不回退）。
- **满员**：返 `SESSION_FULL`，13c 完全不碰 waitlist（留 13d）。
- **取消语义**：默认 release（退名额+退权益）；`release_on_cancel=false`→forfeit consume；场次取消恒退；no_show 留 13e。
- **越权**：404 不泄漏存在性（镜像既有约定）。
- **schema 复核关键发现**：booking.* 权限映射在 000003 base 已对全角色 seed 齐（吻合 §21.1/21.2，存量已 backfill）→ **13c 无需权限迁移**；migration **000012** 仅 bookings/waitlist 全量 unique → partial(`status<>'cancelled'`)。`brand_booking_policies` 无 monthly 列（月限独家来自权益产品）。
- **核心难点**：下单原子性——统一锁序「先锁 session 行、再锁 entitlement 行」，TX-1 下单 / TX-2 代取消 / TX-3 场次取消级联三事务串行化同一 session 行；行锁+unique+CHECK 兜超卖/抢课时。
- **跨批回改（必做）**：`class_session_repository.go:238 Cancel` 加 `SELECT FOR UPDATE` + 级联 cancel 所有 active booking(`session_cancelled`) + release holds + booked_count=0。

#### Batch 13d：候补机制（Waitlist 加入 / 手动转正 / 跳过取消 / 场次取消级联）✅

详细契约：[pds/batches/batch-13d-waitlist.md](batches/batch-13d-waitlist.md)
测试场景：[pds/batches/batch-13d-waitlist-tests.md](batches/batch-13d-waitlist-tests.md)
验收报告：[pds/batches/batch-13d-waitlist-acceptance-report.md](batches/batch-13d-waitlist-acceptance-report.md)

契约状态：**已完成**（2026-06-22 会话内业务验收通过：Happy + Edge + 级联 + 权限/越权全过；验收期修 2 项 UX 瑕疵）。

完成内容：
- 后端（`dev`，6 commits `211fcc2..2d19729`）：**零 migration、零权限码**（waitlist 表 000003 + partial unique 000012 + booking.* seed 齐全就绪）。**抽 13c `Create` 步骤 3-6 为共享 `placeBooking`**（Create staff_assisted / Promote waitlist_promotion 共用，零回归——13c 全测试复跑绿）；新域 `waitlist`（domain/service/persistence/handler/wire）+ 6 错误码——**W1 加入**（锁 session 串行化 position/满员/allow_waitlist/limit(0=不限)/已约/已候补/不锁权益）｜**W2 手动转正**（锁 session+entry/校 waiting+容量/`placeBooking`→entry promoted+booking_id）｜**W3 跳过(skipped+reason)/取消(cancelled)**｜**W4 名单**（按 position+data_scope）｜**TX-3 级联扩展**（场次取消再 cancel 活跃候补）。`waitlistRepository` 持 `bookingRepository` 复用 placeBooking（同包，方法只用 tx，事务边界完整）。data_scope 同 13c（越权 404）。**DB 单测 6（含级联）+ service 4 + 路由冲突冒烟 + 13c 全回归绿；`go build/test ./...` 全绿（29 包）。**
- 前端（`dev`，3 commits `9e63dc7..635a732`）：types + waitlist api client（登记 exports）+ 6 错误码；WaitlistDrawer（名单 + 转正/跳过/取消 + 权限门 + 满员转正禁用）+ PromoteDialog（复用 13c 权益模式子块）+ `/schedule` 场次行「候补 (N)」入口 + 代预约弹窗满员「加入候补」。**lint 0 err + prod build 通过。**

Post-impl code-review（2 review agent）→ **零 P0/P1/P2**（特别确认 `waitlist.Entry` 有 snake_case json tag，13c 同类 P0 不复现——吸取教训从一开始就加）。

验收（全过）：满员加入候补(position 递增)、名单 drawer、腾位后手动转正(auto 选权益)、转正落库 psql 实查(source=waitlist_promotion/booked_count/扣额/promoted_booking_id)、转正无权益占位/跳过、未满/超限/重复/已约边界、满员转正 SESSION_FULL、非 waiting NOT_PROMOTABLE、**场次取消级联候补 psql 实查**、权限门 403、越权 404。

验收期修 2 项 UX 瑕疵（前端 + 全栈，非阻断）：① drawer 头部容量/转正门控用冻结快照不刷新 → schedule 存 `waitlistRef` + `useMemo` 从 live items 派生（转正失效 brand-class-sessions → 实时刷新）；② 场次行「候补」无人数徽标 → 后端 list baseQuery 加 `waitlist_count` 相关子查询（命中 partial index）+ 前端「候补 (N)」+ join/skip/cancel 失效 brand-class-sessions。（第 3 项「禁用按钮缺 Hint」复查为 e2e 误判，代码已有 `<Hint>`，无需改。）

转 FR（见各仓库 `.learnings`）：候补转正微信订阅通知；自动转正（cancel 即顺位）；候补位置手动调整/插队；满员场次受 list page_size=100 限制（既有非回归）。**下一子批：13e 签到 + 履约 + 爽约**（booked→attended/pending_no_show/no_show、hold→consumed、no_show_consumes 真扣课、占位预约签到提醒）。

grill 拍板（2026-06-22 会话内，4 决策点，均推荐项）：
- **转正模型**：手动转正 + 容量门（promote 锁 session 校 `booked_count<capacity` 锁后复验）；**不做 booking 取消时自动标 eligible_to_promote**（零 cancel↔waitlist 耦合，状态机实际只用 waiting）。
- **加入入口**：独立 endpoint `POST /bookings/waitlist`；13c 代预约弹窗满员时显示「加入候补」。
- **候补配置**：effective policy（allow_waitlist + waitlist_limit 解析 override，同 13c）；`waitlist_limit=0`=不限（allow_waitlist 控开关）。
- **候补 UI**：场次维度 drawer（场次行→候补名单 + 转正/跳过/取消）。
- **关键发现**：**13d 零 migration、零权限迁移**——waitlist_entries 表(000003)+两条 partial unique(000012 已建)+booking.*(seed 齐) 全就绪。
- **核心复用**：转正=抽 13c `Create` 步骤 3-6 为共享 `placeBooking`（Create staff_assisted / Promote waitlist_promotion 共用）；候补不锁权益、转正才锁(§22.4)。
- **跨批回改**：扩 13c `ClassSession.Cancel` 级联——cancel 活跃候补(waiting/eligible→cancelled)。
- 锁定默认：转正无权益走 auto/manual/none+跳过；brand staff_assisted（C 端 14）；data_scope 同 13c；W1 锁 session 串行化 position。

#### Batch 13e：签到 / 履约 / 爽约（标到课 + 结束场次 + 确认爽约 + hold 收口）✅

详细契约：[pds/batches/batch-13e-attendance.md](batches/batch-13e-attendance.md)
测试场景：[pds/batches/batch-13e-attendance-tests.md](batches/batch-13e-attendance-tests.md)

**2026-06-24 验收通过（Happy H1-H9 + Edge E1-E10 + 权限 P1-P4 + TX-3 回归 C1-C2 全 PASS；30 DB 单测全绿；prod build exit 0）。已 push + dev FF main。** Batch 13（学员预约闭环）**五子批全闭环**，下一批 **Batch 14 C 端微信自助预约**（source=learner_self_service 薄包装）。

- **关键发现**：**13e 零 migration、零权限码**——attendance_records/entitlement_consumptions/session_records 三表 + 两条 unique（防重签/防重复消费同 hold）+ bookings/holds/txns 状态枚举 + attendance.* 三码及角色映射，全在 000003 已就绪（dev DB 保持 v12）。第 2 个零 migration 批（继 13d）。
- **grill 拍板（2026-06-24，4 决策点，均推荐项）**：①PendingNoShow 触发=**显式「结束场次」action**（session scheduled/in_progress→completed + 未签到 booked 批量→pending_no_show；completed 不可取消 → **TX-3 零改动**）②签到/爽约**并入 booking 域**（非独立域）③UI=**/schedule 场次行 drawer**（镜像 13d 候补 drawer）④占位预约（requires_entitlement_fix）**可签到但不消费、保留 fix 标志**（§13.2）。
- **核心复用**：consume 收口 = 新 sibling 自由函数 `settleHoldForOutcome`（与取消的 `settleHoldOnCancel` 并列）——consume 路径 = forfeit 骨架 + 加 `entitlement_consumptions`(unique hold) + 关联 attendanceID + txn 带 consumption_id；release 路径 = 镜像取消 release。`insertBookingTransaction` 加 `consumption_id` 形参（3 处取消调用点传 nil，复跑 13c/13d 全测试证零回归）。
- **状态机**：CanAttend(booked|pending_no_show)——后者为「结束场次后补签纠错」天然替代撤销误签到；CanConfirmNoShow(仅 pending_no_show)。签到/结束/爽约**不动 booked_count**（只取消退名额）。锁序沿用 session→booking→entitlement。并发靠 attendance unique + consumption unique + 行锁。
- **结束场次权限**=`attendance.mark`（非 session.cancel/schedule.manage——前台/instructor 有 attendance.* 但无 session.*，签到员工须能结束场次）；端点落 `POST /class-sessions/:id/end`（命名空间一致）。
- **code-review（2 路 reviewer）零 P0/P1**：唯一 P1 经两路互证为非 bug——爽约**退课**响应 `hold=null`（baseQuery 既有 join 过滤 released，**与 13c 取消退课一致**），前端 RecordsTab 据 `requires_entitlement_fix`(占位) vs `no_show`+null-hold(退课) 准确区分，履约 Tab 显示正确。当批修 2 个 P2：行级签到禁用（共享 isPending 误禁整列）、invalidateBooking 补 entitlement-transactions 失效。
- 转 FR（见各仓库 `.learnings`）：占位签到 toast/标签文案弱于契约建议（落库正确，仅文案）；爽约退课响应直出 released 产品名（须改 baseQuery DISTINCT ON 并回归 13c）；撤销误签到（§20.12 人工权益调整）；批量一键签到；场次行 attended/no_show 计数徽标；session_records 富履约数据。**Batch 13 闭环完成，转 Batch 14。**

### Batch 14：C 端微信自助预约（学员自助预约 / 桥接 app_user→brand_learner_profile）⏳ 进行中

交接 prompt：[pds/batches/batch-14-handoff.md](batches/batch-14-handoff.md)

**第一个 C 端批**（api-app :8082 + app 前端 :3003）：把已建好的 booking/entitlement/waitlist 域（brand 侧 13a–13e）暴露给学员自助。⚠️ **不是薄包装**——核心难点是 auth 桥接：C 端跑 legacy `app_users`，booking 域操作 `brand_learner_profiles`，两套并行身份 key on openid 但无 FK。

grill + 拆批拍板（2026-06-25 会话内，4 决策点，**均推荐项**）：
- **① Auth 桥接**：登录 find-or-create `learner_identity`(by openid)+`brand_learner_profile`(by brand+identity)，`brand_learner_profile_id` 入 JWT payload（合 §7.1）。沿用 dev 占位 openid `"dev_"+code`，真实 WeChat `code2session`/手机号绑定 → FR；app_user 双轨并存。
- **② 服务形态**：新 app 侧 `LearnerBookingService` 复用 **RBAC-free repo**（brand `booking.Service` 把 RBAC 焊在 service 层、repo 层干净）；ownership 校验替 RBAC，mode=auto/source=`learner_self_service`/scope=nil；镜像 13d `{db,bk}` 同事务复用。
- **③ 范围/拆批**：拆 **14a 核心环**（桥接+课程表+自助预约+我的预约+自助取消+权益预览，证桥接闭环）+ **14b 增量**（我的权益+上课记录+加入候补）；订阅消息/真实微信/app_users 退役/候补转正 → FR。
- **④ §22.1 时间冲突**：学员路径加跨场次 `[starts,ends)` 重叠校验（§22.1「同一时间已有预约」13c 缺失）；只加在 learner 路径（不动共享 `placeBooking`，staff 代预约可故意双约）。
- **关键 schema 发现（全批零 migration，dev DB 保持 v12）**：`bookings.source` 已含 `learner_self_service`、`cancel_source` 已含 `learner`、`operation_logs.actor_type` CHECK 已含 `'learner'`（仅补 Go `audit.ActorLearner` 枚举）；`brand_learner_profiles` partial unique(000010) 使 find-or-create 幂等；JWT 加 `profile_id` claim 无状态。
- **复用 `placeBooking` 的 3 摩擦点**：①`assisted_by` 无条件赋值且 **FK→brand_users**（自助须传 NULL，塞 learner id 会 23503）→ 参数化 `assistedBy *int64`；②审计焊死 `ActorBrandUser` → 补 `ActorLearner`；③`Cancel` 无所有权参数 + 焊死 `CancelSourceStaff` → 新 learner 取消路径加 tx 内 ownership + `cancel_source=learner`。改后复跑 13c–13e 全 DB 单测证零回归。
- **桥接副作用**：13a 的 identity FoC 是 **by-phone**（要求 phone），桥接需**新增 by-openid 路径**；同人 brand 侧 by-phone 建档与 C 端 by-openid 登录是两个 profile，v1 不合并 → 验收须「学员先 C 端登录建 profile P，brand 再给 P 发权益」。

#### Batch 14a：自助预约核心环（桥接 + 课程表 + 下单 + 我的预约 + 取消）✅

详细契约：[pds/batches/batch-14a-self-booking.md](batches/batch-14a-self-booking.md)
测试场景：[pds/batches/batch-14a-self-booking-tests.md](batches/batch-14a-self-booking-tests.md)
验收 prompt：[pds/batches/batch-14a-acceptance-prompt.md](batches/batch-14a-acceptance-prompt.md)

**2026-06-25 冒烟验收通过（主线程自测 API+psql+chrome-devtools 浏览器全流程全过）。全批零 migration（dev DB v12）、零权限码。**

完成内容：
- 后端（`dev`）：JWT `profile_id` claim + `audit.ActorLearner`（DB CHECK 已含 'learner'，纯 Go）；`placeBooking`/`applyCancel`/`settleHoldOnCancel` 参数化 `actor→*int64`（assisted_by/cancelled_by/operated_by 均 brand_users FK，学员传 NULL 避 23503，staff 行为不变）；桥接 `FindOrCreateProfileByOpenID`（by-openid、phone-less、幂等、quota 门、audit=learner，区别于 13a by-phone）；`CreateByLearner`（auto/source=learner_self_service/§22.1 跨场次重叠校验 hasOverlappingBooking + 锁 profile 行串行化并发）+ `CancelByLearner`（tx 内 ownership + cancel_source=learner）；新域 `application/learnerbooking`（无 RBAC，profile 收口，复用 RBAC-free repo，课程表复用 classsession repo.List）；app endpoints `/api/v1/app`(GET class-sessions[+:id]/POST bookings/GET bookings/GET usable-entitlements/POST bookings/:id/cancel) + wechatLogin 串桥接 + middleware GetProfileID + wire。新错误码 `BOOKING_TIME_CONFLICT`。**`go build ./... && go test ./...` 全绿（60 包），13c/13d/13e 零回归。**
- 前端（`dev`）：`packages/api/app.ts` C 端 hooks + 类型(id 用 number 对齐后端 int64) + `appBookingErrorText` 学员友好失败文案；`/class-sessions`（场次卡片 + 预约确认弹窗 §5.7 权益预览 + 失败文案）+ `/bookings`（状态筛选 + 自助取消 ConfirmDialog）+ 底部导航对齐 §14.4。**prod build exit 0。**

Post-impl code-review（2 并行 agent）→ 3 项当批修：后端 P1 并发自助下单 TOCTOU（锁 profile 行）；前端 P0 App* id `string→number`（后端 int64 序列化为 JSON number）；前端 P1 补 2 权益错误码。

冒烟（主线程自测，= 业务验收）暴露并修 **3 个 C 端 greenfield 既有阻断**（桥接前 C 端从未端到端跑过）：① `app_users.vip_level` 列 tag 缺失 → 新用户登录 INSERT 42703；② app router 无 CORS（brand/admin 都有）→ 浏览器跨域被拦；③ 登录 `brand_id` 发 string → 后端 int64 bind 失败。另：api-app 启动需 `CONFIG_PATH=configs/config-app.yaml`（默认 config.yaml 跑成 :8081）。

桥接核心锚点验证（psql 实查）：C 端 `dev_alice`→identity→profile 14，自助 booking `source=learner_self_service`/`assisted_by` NULL/`audit actor=learner`，与 brand 侧 /bookings 同一 profile；ownership(bob 取消 alice→404)、无权益、时间冲突、取消(cancel_source=learner/hold released) 全过。浏览器全流程（登录→课程表→预约确认弹窗→预约成功→我的预约→取消）通过。

转 FR（见三库 `.learnings`）：真实 WeChat code2session + 手机号绑定；同人 by-phone↔by-openid identity 合并；app_users 退役；并发首登 savepoint；硬导航水合门；订阅消息。**下一子批：14b 我的权益 + 上课记录 + 加入候补。**

## 6. 验收命令

后端：

```bash
cd /Users/liushan/Documents/zkw/mini_schedule/backend
go test ./...
```

前端 admin：

```bash
cd /Users/liushan/Documents/zkw/mini_schedule/web
pnpm --filter @mini-schedule/admin lint
pnpm --filter @mini-schedule/admin build
```

数据库：

- 从空库顺序执行 `000001`、`000002`、`000003` up。
- 至少验证 `000003` down 不破坏前置表。

## 7. 每轮工作记录模板

```text
## YYYY-MM-DD Session

目标：

### 契约（Batch 开始时填写，用户 approve 后方可开工）

**API 接口**

| 方法 | 路径 | 请求字段 | 响应字段 |
|---|---|---|---|
|  |  |  |  |

**前端页面模块**

| 页面/模块 | 类型 | 关键字段/操作 |
|---|---|---|
|  |  |  |

契约状态：待 approve / 已 approve

---

完成：

修改文件：

验证：

风险/未完成：

下一步：

提交：
```

## 2026-05-28 Session

目标：

- 完成 Batch 1：公开注册和首购订单后端。

完成：

- 新增公开注册首购订单 API：`POST /api/v1/public/signup/orders`。
- 支持手机号、短信验证码、密码、品牌资料、SaaS Plan、billing cycle、payment channel。
- 创建 `pending` Brand。
- 创建 BrandUser 负责人占位，并标记 `is_owner = TRUE`。
- 创建 `pending_payment` SaaSPlanOrder。
- 订单金额在事务内从 active SaaSPlan 读取。
- 开发/mock 场景支持固定短信验证码，生产环境未配置真实 Provider 时不放行。

修改文件：

- `/Users/liushan/Documents/zkw/mini_schedule/backend/internal/domain/commercial/repository.go`
- `/Users/liushan/Documents/zkw/mini_schedule/backend/internal/application/commercial/service.go`
- `/Users/liushan/Documents/zkw/mini_schedule/backend/internal/infrastructure/persistence/commercial_repository.go`
- `/Users/liushan/Documents/zkw/mini_schedule/backend/internal/interfaces/admin/commercial_handler.go`
- `/Users/liushan/Documents/zkw/mini_schedule/pds/PROGRESS.md`

验证：

- `cd /Users/liushan/Documents/zkw/mini_schedule/backend && go test ./...`

风险/未完成：

- 尚未接入真实短信 Provider；当前生产环境真实 Provider 会返回“尚未接入”。
- 尚未接入微信支付 Native，下单二维码属于 Batch 2。
- 支付成功后的 Brand 激活、订阅快照、回调幂等属于 Batch 3。

下一步：

- Batch 2：微信支付 Native 下单。

提交：

- 待提交。

## 8. 风险和注意事项

- 根目录 `CLAUDE.md` 仍有“健身 SaaS”旧表述；当前产品定位以 `/pds` 为准。
- `backend` 里存在部分未提交变更时，提交前必须区分本轮改动和用户改动。
- 支付实现不能只依赖前端支付结果，必须以后端回调和主动查询为准。
- 套餐额度硬限制只禁止新增资源，不删除、不停用存量数据。
- 学员端通知必须考虑微信订阅消息授权限制，通知失败不能影响主流程。
