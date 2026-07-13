# Batch 20 契约 —— 支付异常人工补偿（平台 admin）

## Lifecycle

| 阶段 | 时间戳 | 触发人 |
|---|---|---|
| draft | 2026-07-12 | claude |
| approved | 2026-07-12（会话内，采用推荐设计） | user |
| implementing | 2026-07-12/13（T1–T3，5 commits） | claude |
| done | 2026-07-13（go test ./... 38 包绿；pnpm --filter admin build 通过；含 /code-review 并发锁修复） | claude |

## 背景与范围

支付回调失败 / 从未到达（订单卡 `pending_payment`）或金额不符（订单 `exception`）时，品牌已付款但订阅未开通。平台 admin 核实后**人工补偿**：从该订单开通 BrandSubscription。

**现状核查（grounded）**：
- admin 已有：订单列表（可 filter status）、订阅列表、`manualRenew`（续期**已存在**订阅）、限额/状态调整、支付流水 + 回调日志 + 操作日志观测。
- **缺口**：从**首购**卡单（无订阅）开通订阅的动作 —— `manualRenew` 只能续既有订阅。
- 订阅开通 happy-path 在 `commercial_repository.go:ProcessWeChatCallback`（6.1–6.8：load plan → 建 BrandSubscription 快照 → 激活 brand → audit）。

**设计决策（保护关键路径）**：
- 新增独立 `CompensateSaaSPlanOrder` 仓储方法，**不改动重度测试的支付回调**（零 payment-path 回归风险）。订阅激活逻辑与 callback 重复约 40 行 → 记 FR「统一 activation」。
- 范围 = **首购补偿**：品牌**无有效订阅**（active/grace）时才补；已有 → 拒绝并提示改用续期（`manualRenew`）。
- 补偿动作镜像 `manualRenew`：不造假支付流水，仅 开通订阅 + 订单→paid + audit `saas_plan_order_compensated`（actor=platform_admin）。

## 契约

### 后端

**仓储** `CompensateSaaSPlanOrder(ctx, input) (*result, error)`（单事务）：
1. 锁订单；不存在 → 404。
2. 状态守卫：仅 `pending_payment` / `exception` 可补；paid → 幂等返回既有订阅；其它终态（closed/failed/refunded）→ 409。
3. 品牌有效订阅守卫：已有 active/grace → 409「品牌已有有效订阅，请用续期」。
4. load plan（不存在 → 404）；建 BrandSubscription(active，快照 limits + features，按 billing_cycle 算 expires_at)。
5. 订单 → paid（paid_at=now）；激活 brand（pending→active）。
6. audit `saas_plan_order_compensated`（actor=platform_admin，reason）。

**API**（admin，鉴权走 cookie）：

| 方法 | 路径 | 请求 | 响应 |
|---|---|---|---|
| POST | `/api/v1/admin/saas-plan-orders/:id/compensate` | `reason` | `order_id, brand_id, subscription_id` |

（订单列表已支持 filter；异常订单看板复用现有 `GET /saas-plan-orders?status=exception`。）

### 前端（admin app）

| 页面/模块 | 类型 | 关键操作 |
|---|---|---|
| 订单列表页 | 现有页增强 | status 筛选加「异常/待支付」；异常/卡单行显「补偿开通」按钮 |
| 补偿弹窗 | 弹窗 | 填 reason → 确认 → 调 compensate → 成功刷新列表 + toast |

## 任务拆解（TDD 逐 task commit）

- **T1** domain input/result + Repository 方法 + `CompensateSaaSPlanOrder` 仓储实现；DB 测试（exception 订单补偿→订阅 active + 订单 paid + brand active；幂等；无效状态/已有订阅守卫）。
- **T2** application service `CompensateSaaSPlanOrder` + admin handler 端点 + 路由 + Wire。
- **T3**（admin 前端）订单页补偿动作 + 弹窗 + API 客户端。完成跑 `/code-review`。

**验收**：`go test ./...` 全绿；集成测试断言补偿开通；`pnpm --filter admin build` 通过。
