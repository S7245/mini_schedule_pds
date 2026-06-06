# Batch 03 测试场景 — 微信支付回调开通订阅

执行方式：Go 单元/集成测试（`go test ./...`）+ 手动 curl 模拟回调

前置条件：
- api-brand 在 localhost:8081 运行
- 数据库已迁移，至少一个 `pending_payment` 的 SaaSPlanOrder 存在（可由 Batch 1/2 流程生成）
- 微信支付为 **mock 模式**（`payment.wechat.app_id` 为空或 `payment.wechat.allow_mock=true`）

mock 模式下的约定（实现层与本测试文件保持一致）：
- `Wechatpay-Signature: mock_signature` 视为验签通过
- `Wechatpay-Signature: invalid_xxx` 视为验签失败
- 请求体直接传明文 JSON（不做 AES-GCM 解密），结构同微信解密后的 `resource` 内容
- Timestamp 仍然校验（≤ 5 分钟）

---

## Happy Path

| # | 步骤 | 预期结果 |
|---|---|---|
| H1 | 准备一个 `pending_payment` 订单（out_trade_no = `MSTEST001`，amount = `9900`） | 订单存在，Brand 状态 `pending` |
| H2 | POST `/api/v1/public/payment/callback`，headers 含 `mock_signature` + 当前 timestamp，body `{out_trade_no:"MSTEST001", transaction_id:"wx_tx_001", amount:{total:9900,currency:"CNY"}, trade_state:"SUCCESS", success_time:"..."}` | 返回 200 `{code:200, message:"success"}` |
| H3 | 查询订单 | `status=paid`，`paid_at` 非空，`third_party_trade_no="wx_tx_001"` |
| H4 | 查询 Brand | `status=active` |
| H5 | 查询 BrandSubscription | 存在一条新记录，`status=active`，`starts_at` ≈ 当前时间，`expires_at` 按 plan 周期推算（月套餐 +1 月 / 年套餐 +1 年），max_locations / max_staff_seats / max_learners 与 plan 一致 |
| H6 | 查询 PaymentTransaction | 存在一条 `status=succeeded`、`transaction_type=payment`、`amount=9900`、`third_party_trade_no="wx_tx_001"` |
| H7 | 查询 PaymentCallbackLog | 存在一条 `status=processed`、`processed_at` 非空 |
| H8 | 查询 OperationLog | 存在一条 `action=payment_callback_success`、`target_type=saas_plan_order`、`target_id=订单ID` |

---

## Edge Cases — 幂等

| # | 场景 | 操作 | 预期结果 |
|---|---|---|---|
| E1 | 同 out_trade_no 重复回调 | H2 完成后，再次发起同样的 callback 请求 | 返回 200 `success`；订单状态不变（仍为 paid）；PaymentTransaction 仍只有 1 条；新增 1 条 PaymentCallbackLog `status=processed` 标识重复 |
| E2 | 并发回调（同一 out_trade_no 同时两次） | 并行发起 2 个相同 callback | 只有一次更新生效；两次都返回 200；最终订单状态 `paid` |

---

## Edge Cases — 金额不一致

| # | 场景 | 操作 | 预期结果 |
|---|---|---|---|
| E3 | 回调金额 < 订单金额 | 准备订单 amount=9900，发起 callback amount.total=100 | 返回 200；订单 `status=exception`；**不创建** BrandSubscription；**不激活** Brand；PaymentCallbackLog `status=failed` + `error_message` 含 "amount mismatch"；OperationLog `action=payment_amount_mismatch` |
| E4 | 回调金额 > 订单金额 | 同 E3，金额改为 99999 | 同 E3 处理（exception） |
| E5 | 货币不一致 | 回调 currency="USD" | 返回 200；订单 `status=exception`；CallbackLog `status=failed` |

---

## Edge Cases — 验签 / 防重放

| # | 场景 | 操作 | 预期结果 |
|---|---|---|---|
| E6 | 验签失败 | header `Wechatpay-Signature: invalid_xxx` | 返回 401；订单状态不变；PaymentCallbackLog `status=failed` + `error_message` 含 "signature" |
| E7 | Timestamp 过期 | header `Wechatpay-Timestamp` 设为 10 分钟前 | 返回 401；订单状态不变；CallbackLog `status=failed` + `error_message` 含 "timestamp" |
| E8 | Timestamp 未来太多 | timestamp 设为 10 分钟后 | 同 E7 |
| E9 | 缺失签名头 | 不带 `Wechatpay-Signature` | 返回 401；CallbackLog `status=failed` |
| E10 | 缺失 timestamp 头 | 不带 `Wechatpay-Timestamp` | 返回 401；CallbackLog `status=failed` |

---

## Edge Cases — 订单状态异常

| # | 场景 | 操作 | 预期结果 |
|---|---|---|---|
| E11 | 订单不存在 | out_trade_no=`MSNOTFOUND` | 返回 200（微信要求）；CallbackLog `status=ignored` + brand_id/order_id 为 NULL |
| E12 | 订单已 `paid`（非首次回调，状态已是 paid） | 对 H3 后的订单再次回调 | 同 E1，幂等返回 200，不重复创建 Subscription/Transaction |
| E13 | 订单已 `closed` | 准备订单状态=closed，发起回调 | 返回 200；订单状态不变；CallbackLog `status=ignored` + `error_message` 含 "order closed" |
| E14 | 订单已 `exception` | 准备订单状态=exception | 返回 200；不修改；CallbackLog `status=ignored` |

---

## Edge Cases — trade_state ≠ SUCCESS

| # | 场景 | 操作 | 预期结果 |
|---|---|---|---|
| E15 | trade_state=USERPAYING | body 内 `trade_state="USERPAYING"` | 返回 200；订单状态不变（仍 pending_payment）；CallbackLog `status=ignored` + `error_message` 含 "non-success trade_state" |
| E16 | trade_state=CLOSED | body 内 `trade_state="CLOSED"` | 返回 200；订单状态 → `closed`；CallbackLog `status=processed`；**不创建** Subscription / 激活 Brand |
| E17 | trade_state=PAYERROR | body 内 `trade_state="PAYERROR"` | 返回 200；订单 `status=failed`；CallbackLog `status=processed` |

---

## Edge Cases — 事务回滚

| # | 场景 | 操作 | 预期结果 |
|---|---|---|---|
| E18 | 创建 BrandSubscription 失败（mock 注入错误） | 通过测试钩子在 Subscription 插入时抛错 | 返回 500；订单状态保持 `pending_payment`；Brand 状态保持 `pending`；无 BrandSubscription 创建；PaymentTransaction 回滚；CallbackLog 单独写入 `status=failed`（事务外） |
| E19 | 激活 Brand 失败（Brand 已被删除等） | brand 记录被人为标记为 deleted | 返回 500；订单 / Transaction / Subscription 均回滚 |

---

## 执行方式

**单元测试**（`go test ./internal/application/commercial/...`）：
- 覆盖 E3 / E4 / E5 / E6 / E7 / E11 / E13 / E15 / E16 / E17 等纯逻辑分支
- mock Repository + mock WeChatPaymentAdapter

**集成测试**（`go test ./internal/infrastructure/persistence/... -tags=integration`）：
- 覆盖 H1–H8、E1、E2、E12、E18、E19 等需要真实数据库事务的场景
- 使用 testcontainer 或 dev 数据库

**手动 curl 验证**（mock 模式）：

```bash
# Happy path
curl -X POST http://localhost:8081/api/v1/public/payment/callback \
  -H "Wechatpay-Signature: mock_signature" \
  -H "Wechatpay-Timestamp: $(date +%s)" \
  -H "Wechatpay-Nonce: nonce123" \
  -H "Wechatpay-Serial: serial001" \
  -H "Content-Type: application/json" \
  -d '{
    "out_trade_no": "MSTEST001",
    "transaction_id": "wx_tx_001",
    "trade_state": "SUCCESS",
    "amount": {"total": 9900, "currency": "CNY"},
    "success_time": "2026-06-04T10:30:45+08:00"
  }'
```
