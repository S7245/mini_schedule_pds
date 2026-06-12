# 课程预约实现进度

更新时间：2026-06-12

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
| P1 | 课程和场次 | CourseTemplate、ClassSession、资源占用 |
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
- 门店删除引用保护（`LOCATION_IN_USE`：有员工任职/未来场次引用时禁删）。
- `UpdateStatus` 改 gate `location.toggle_status`（现 location.edit，seed 的 toggle_status 闲置）。
- 后端 location list 加 name 搜索（前端目前只有状态筛选+分页）。
- 可选：加「店长权限门」e2e 用例（断言 location_manager 在 /locations 上写操作按钮全 disabled）。

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
