# 课程预约实现进度

更新时间：2026-05-29

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
| P0 | Staff / Learner 额度硬限制 | Batch 4 完成 Location 端 quota；Staff / Learner POST 接口落地时一并接 SubscriptionGuard |
| P1 | Brand 初始化向导后续步骤 | Batch 4 完成骨架 + 第 1-2 步；3-8 步（员工/教练/课程分类/课程模板/权益/场次/小程序二维码）随后续 batch 逐步落地 |
| P1 | Brand 角色权限 | Staff、InstructorProfile、菜单权限、数据权限、操作权限 |
| P1 | 课程和场次 | CourseTemplate、ClassSession、资源占用 |
| P1 | 学员权益 | Entitlement Product、Learner Entitlement、Hold、Consume、Adjust |
| P1 | 学员预约 | 微信小程序品牌空间、课程表、预约、取消 |
| P2 | 候补机制 | 第一版可员工手动转正，自动候补后期补 |
| P2 | 通知 | 员工站内通知，学员微信订阅消息 |

## 4. Now / Next / Later

### Now

1. Staff / InstructorProfile CRUD + 同款 SubscriptionGuard quota（向导第 3 步实做）。
2. Brand 权限模型第一版（菜单 / 数据 / 操作权限）。

### Next

1. 课程分类 / 课程模板 / 课程场次（向导第 4-7 步）。
2. 小程序二维码生成（向导第 8 步）。
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
