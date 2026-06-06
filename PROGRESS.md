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
| P0 | 订阅额度硬限制 | Location、Staff、Learner 新增前需检查 BrandSubscription 快照 |
| P1 | Brand 初始化向导 | 支付成功后引导品牌配置 Location、员工、课程、权益 |
| P1 | Brand 角色权限 | Staff、InstructorProfile、菜单权限、数据权限、操作权限 |
| P1 | 课程和场次 | CourseTemplate、ClassSession、资源占用 |
| P1 | 学员权益 | Entitlement Product、Learner Entitlement、Hold、Consume、Adjust |
| P1 | 学员预约 | 微信小程序品牌空间、课程表、预约、取消 |
| P2 | 候补机制 | 第一版可员工手动转正，自动候补后期补 |
| P2 | 通知 | 员工站内通知，学员微信订阅消息 |

## 4. Now / Next / Later

### Now

1. 订阅额度硬限制。
2. Brand 初始化向导（Batch 4）。

### Next

1. Brand 初始化向导。
2. Location / Staff / InstructorProfile。
3. Brand 权限模型第一版。
4. 课程模板和课程场次。

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
