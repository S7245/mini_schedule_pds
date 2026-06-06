# Batch 03 — 微信支付回调开通订阅

状态：**契约草稿** ⏳（待用户 approve）
更新时间：2026-06-02

---

## 业务背景

Batch 2 完成了 Native 二维码下单，生成了 `pending_payment` 状态的 SaaSPlanOrder。本批次实现微信侧的支付回调接收、验签、幂等处理，以及支付成功后的完整事务闭环：

```text
微信支付回调
  → 验签（RSA-SHA256）
  → 幂等查重（out_trade_no）
  → 金额校验
  → 事务处理：
      - 写 PaymentCallbackLog（记录回调）
      - 写 PaymentTransaction（记录成功的支付事务）
      - 更新 SaaSPlanOrder（pending_payment → paid）
      - 创建 BrandSubscription（快照）
      - 激活 Brand（pending → active）
  → 返回 "success"
```

异常场景：
- 验签失败 → 返回失败，不修改任何状态
- 金额不一致 → SaaSPlanOrder 进入 `exception` 状态，写操作日志
- 重复回调（same out_trade_no） → 幂等返回成功
- 订单不存在 → 返回失败，写 PaymentCallbackLog（status=ignored）

---

## API 接口契约

### POST /api/v1/public/payment/callback

**功能**：接收微信支付回调，验签、更新订单、开通订阅。

**请求头**：
| Header | 说明 |
|---|---|
| `Wechatpay-Signature` | 微信支付签名（RSA-SHA256） |
| `Wechatpay-Timestamp` | 时间戳（秒级） |
| `Wechatpay-Nonce` | 随机字符串，防重放 |
| `Wechatpay-Serial` | 微信支付平台证书序列号 |

**请求体**（JSON，微信 API v3 格式）：

```json
{
  "id": "12345678-1234-1234-1234-123456789012",
  "create_time": "2026-06-02T10:30:45+08:00",
  "resource": {
    "original_type": "transaction",
    "algorithm": "AEAD_AES_256_GCM",
    "ciphertext": "...",
    "associated_data": "transaction",
    "nonce": "..."
  },
  "event_type": "TRANSACTION.SUCCESS"
}
```

解密后的 `resource` 内容（JSON）：

```json
{
  "id": "wechat_transaction_id",
  "mchid": "merchant_id",
  "out_trade_no": "MS20260602103045abc123def456",
  "appid": "wx...",
  "trade_type": "NATIVE",
  "trade_state": "SUCCESS",
  "trade_state_desc": "支付成功",
  "amount": {
    "total": 9900,
    "payer_total": 9900,
    "currency": "CNY",
    "payer_currency": "CNY"
  },
  "payer": {
    "openid": "..."
  },
  "success_time": "2026-06-02T10:30:45+08:00"
}
```

**响应**（幂等，成功始终返回 200 OK）：

```json
{
  "code": 200,
  "message": "success"
}
```

---

## 核心验收标准

**验签与安全**：
- [x] 验证 `Wechatpay-Signature` 正确（使用微信平台证书、RSA-SHA256）
- [x] 验证 `Wechatpay-Timestamp` 与当前时间误差 ≤ 5 分钟（防重放）
- [x] 解密 `resource.ciphertext`（AES-256-GCM）

**幂等处理**：
- [x] 同一 `out_trade_no` 多次回调，第一次更新状态，后续幂等返回成功
- [x] 通过 `out_trade_no` 唯一索引 (`UNIQUE INDEX idx_saas_plan_orders_out_trade_no`) 保证
- [x] PaymentCallbackLog 记录每次回调

**金额校验**：
- [x] 比对 `amount.total`（单位分）与 SaaSPlanOrder.amount（单位分）
- [x] 若不一致，SaaSPlanOrder.status → `exception`，不执行后续步骤
- [x] 记录不一致的信息到 OperationLog（action=payment_amount_mismatch）

**事务处理**（任一步失败，整体回滚）：
1. [x] 校验订单存在且状态为 `pending_payment`
2. [x] 校验金额匹配
3. [x] 插入 PaymentCallbackLog（status=received）
4. [x] 插入 PaymentTransaction（status=succeeded）
5. [x] 更新 SaaSPlanOrder（status→paid, paid_at→当前时间, third_party_trade_no→微信 transaction_id）
6. [x] 创建 BrandSubscription（快照）
7. [x] 激活 Brand（status→active）
8. [x] 更新 PaymentCallbackLog（status→processed）
9. [x] 写 OperationLog（action=payment_callback_success）

**错误场景**：
- [x] 验签失败 → 返回 400 或 500（根据错误类型）+ PaymentCallbackLog(status=failed)
- [x] 订单不存在 → 返回 200（微信要求）+ PaymentCallbackLog(status=ignored)
- [x] 订单非 pending_payment → 返回 200 + PaymentCallbackLog(status=processed)（重复回调）
- [x] 解密失败 → 返回 500 + PaymentCallbackLog(status=failed)

---

## 数据库变更

### 1. PaymentCallbackLog 表（已存在）

```sql
CREATE TABLE IF NOT EXISTS payment_callback_logs (
    id                BIGSERIAL PRIMARY KEY,
    brand_id          BIGINT REFERENCES brands(id) ON DELETE SET NULL,
    order_id          BIGINT REFERENCES saas_plan_orders(id) ON DELETE SET NULL,
    transaction_id    BIGINT REFERENCES payment_transactions(id) ON DELETE SET NULL,
    payment_channel   VARCHAR(50) NOT NULL,
    out_trade_no      VARCHAR(255),
    third_party_trade_no VARCHAR(255),
    callback_request_id VARCHAR(255),
    status            VARCHAR(50) NOT NULL CHECK (status IN ('received', 'processed', 'failed', 'ignored')),
    processed_at      TIMESTAMP,
    error_message     TEXT,
    created_at        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_payment_callback_logs_out_trade_no ON payment_callback_logs(out_trade_no);
CREATE INDEX idx_payment_callback_logs_status_created_at ON payment_callback_logs(status, created_at DESC);
```

### 2. PaymentTransaction 表（已存在）

```sql
CREATE TABLE IF NOT EXISTS payment_transactions (
    id                 BIGSERIAL PRIMARY KEY,
    brand_id           BIGINT REFERENCES brands(id) ON DELETE SET NULL,
    order_id           BIGINT REFERENCES saas_plan_orders(id) ON DELETE SET NULL,
    payment_channel    VARCHAR(50) NOT NULL,
    transaction_type   VARCHAR(50) NOT NULL CHECK (transaction_type IN ('payment', 'refund')),
    status             VARCHAR(50) NOT NULL CHECK (status IN ('pending', 'succeeded', 'failed', 'closed', 'refunding', 'refunded', 'exception')),
    amount             BIGINT NOT NULL,
    currency           VARCHAR(3) NOT NULL DEFAULT 'CNY',
    out_trade_no       VARCHAR(255) NOT NULL,
    third_party_trade_no VARCHAR(255),
    provider_request_id VARCHAR(255),
    callback_received_at TIMESTAMP,
    paid_at            TIMESTAMP,
    failure_reason     TEXT,
    created_at         TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at         TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_payment_transactions_order_id ON payment_transactions(order_id);
```

### 3. SaaSPlanOrder 表（已存在，需确认 third_party_trade_no 字段）

```sql
-- 已存在，包含：
-- - id, brand_id, plan_id, amount, currency
-- - status: pending_payment / paid / closed / failed / exception
-- - out_trade_no: 唯一约束
-- - third_party_trade_no: 微信 transaction_id（更新时填充）
-- - paid_at: 支付完成时间
```

### 4. BrandSubscription 表（已存在）

```sql
-- 已存在，包含：
-- - id, brand_id, plan_id, order_id
-- - status: active / grace_period / restricted / frozen / expired / cancelled
-- - starts_at, expires_at
-- - max_locations, max_staff_seats, max_learners
```

### 5. Brand 表（已存在）

```sql
-- 已存在，包含：
-- - id, status: pending / active / inactive / frozen
```

---

## 后端四层实现范围

### Domain（业务规则）

**文件**：`backend/internal/domain/commercial/repository.go` + `commercial.go`

**新增接口方法**：

```go
type Repository interface {
  // ... 现有方法 ...

  // 支付回调接收和处理
  ProcessWeChatCallback(ctx context.Context, input ProcessWeChatCallbackInput) (*ProcessWeChatCallbackResult, error)
}

type ProcessWeChatCallbackInput struct {
  OutTradeNo         string    // 订单号（唯一键）
  ThirdPartyTradeNo  string    // 微信交易ID
  Amount             string    // 金额（分）
  PaymentChannel     PaymentChannel
  CallbackRequestID  string    // 回调请求ID（防重放）
  ReceivedAt         time.Time
}

type ProcessWeChatCallbackResult struct {
  Success             bool
  OrderID             int64
  BrandID             int64
  Message             string
}

// 新增领域常量
const PaymentCallbackReplayWindow = 5 * time.Minute
```

**验签逻辑**：
- RSA-SHA256 签名校验（使用微信平台证书）
- Timestamp 防重放（≤ 5 分钟）

### Application（业务编排）

**文件**：`backend/internal/application/commercial/service.go`

**新增方法**：

```go
func (s *Service) ProcessWeChatPaymentCallback(
  ctx context.Context,
  headers map[string]string,
  body []byte,
) (*ProcessWeChatCallbackResult, error) {
  // 1. 验签（调用 infrastructure adapter）
  // 2. 解密回调体
  // 3. 校验 Timestamp（防重放）
  // 4. 调用 repo.ProcessWeChatCallback()
  // 5. 处理事务和错误
  return result, nil
}
```

### Infrastructure（适配器和数据库）

**文件**：`backend/internal/infrastructure/persistence/commercial_repository.go` + `commercial_models.go`

**新增持久层方法**：

```go
func (r *commercialRepository) ProcessWeChatCallback(
  ctx context.Context,
  input commercial.ProcessWeChatCallbackInput,
) (*commercial.ProcessWeChatCallbackResult, error)
```

**实现细节**：
- 开启数据库事务（Tx）
- 查询 SaaSPlanOrder（by out_trade_no）
- 幂等查重（若已 paid，返回成功）
- 金额校验
- 插入 PaymentCallbackLog
- 插入 PaymentTransaction
- 更新 SaaSPlanOrder
- 创建 BrandSubscription
- 激活 Brand
- 更新 PaymentCallbackLog 为 processed
- 写 OperationLog
- 提交事务

**微信支付 Adapter**：

新增文件 `backend/internal/infrastructure/payment/wechat_adapter.go`

```go
type WeChatPaymentAdapter struct {
  cfg *config.WeChatPayConfig
  // 微信平台证书等
}

func (a *WeChatPaymentAdapter) VerifyCallback(
  ctx context.Context,
  headers map[string]string,
  body []byte,
) (*CallbackNotification, error)

func (a *WeChatPaymentAdapter) DecryptResource(
  ctx context.Context,
  ciphertext, nonce, associatedData string,
) ([]byte, error)
```

### Interfaces（HTTP 处理器）

**文件**：`backend/internal/interfaces/admin/commercial_handler.go`

**新增路由**：

```go
func (h *Handler) RegisterPublicRoutes(r *gin.RouterGroup) {
  // ... 现有路由 ...
  r.POST("/payment/callback", h.handleWeChatPaymentCallback)
}

func (h *Handler) handleWeChatPaymentCallback(c *gin.Context) {
  // 1. 提取请求头
  // 2. 读取请求体
  // 3. 调用 service.ProcessWeChatPaymentCallback()
  // 4. 返回微信要求的格式（200 OK）
}
```

---

## 测试场景概览

（详细测试场景由用户 approve 契约后编写在 `batch-03-wechat-callback-tests.md`）

### Happy Path
- [x] 正常回调 → 订单状态更新为 paid，Brand 激活，BrandSubscription 创建

### Edge Cases
- [x] 重复回调（same out_trade_no）→ 幂等成功
- [x] 金额不一致 → exception 状态 + OperationLog
- [x] 验签失败 → 400/500 + PaymentCallbackLog(status=failed)
- [x] 订单不存在 → 200 + PaymentCallbackLog(status=ignored)
- [x] Timestamp 过期（> 5 min）→ 400 + PaymentCallbackLog(status=failed)

---

## 前端范围（暂无）

Batch 2 已完成的支付页会在回调成功后显示"支付成功"状态。无额外前端需求。

---

## 验收清单

**后端验收**：
- [ ] `go build ./...` 通过
- [ ] `go test ./...` 通过（包括 callback 单元测试、事务回滚测试）
- [ ] 数据库迁移支持从空库顺序执行 up
- [ ] callback 幂等测试通过
- [ ] 金额不一致测试通过
- [ ] 验签失败测试通过

**业务逻辑验收**：
- [ ] 真实微信回调能成功验签
- [ ] 支付成功后 Brand 状态自动激活
- [ ] BrandSubscription 快照时间范围正确
- [ ] PaymentCallbackLog 和 PaymentTransaction 记录完整
- [ ] OperationLog 审计完整

---

## 契约状态

- **状态**：待 approve ⏳
- **用户需确认**：
  1. API 接口是否满足微信 v3 格式要求？
  2. 数据库表和字段是否完整？
  3. 事务处理的步骤顺序和回滚策略是否合理？
  4. 是否需要额外的重放防护（如 Nonce 记录到 Redis）？
  5. 异常订单补偿流程何时启动（下一批次）？

---

## 参考资料

- 微信支付 API v3 回调文档：https://pay.weixin.qq.com/wiki?id=221482582996220304
- Batch 2 下单实现：`pds/batches/batch-02-wechat-pay.md`
- 支付设计指南：`pds/GO_BACKEND_LANGUAGE_DESIGN.md` Section 9
