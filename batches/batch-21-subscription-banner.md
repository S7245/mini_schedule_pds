# Batch 21 契约 —— 订阅受限态前端闭环（brand banner + 到期提醒）

## Lifecycle

| 阶段 | 时间戳 | 触发人 |
|---|---|---|
| draft | 2026-07-13 | claude |
| approved | 2026-07-13（会话内） | user |
| done | 2026-07-13（go test ./... 38 包绿；pnpm --filter brand build 通过） | claude |

## 范围

Batch 16 订阅生命周期自动化落地后，品牌端缺前端反馈。本批补：

1. **C1 后端读端点**：`GET /api/v1/brand/subscription` 返回品牌**最新**订阅（status / expires_at / grace_ends_at / limits / frozen_reason）。无订阅返回 `data:null`。repo `GetLatestSubscriptionByBrand`（无订阅 nil,nil）+ service `GetMySubscription` + brand `SubscriptionHandler` + Wire。
2. **C2 前端 banner**：受保护布局顶部 `SubscriptionBanner`：
   - `restricted / expired / frozen` → 红条（受限/过期/冻结，联系平台续期）
   - `grace_period` → 黄条（宽限期至 X，尽快续期）
   - `active` 且 ≤7 天到期 → 黄条（将于 X 到期）
   - `active` 未临近 / 无订阅 → 不显示
   - brand 自助续费未落地（§17.3），文案引导「联系平台续期」，无 fake 按钮。
3. **C3 到期前提醒通知**：生命周期 sweep 转 `grace_period` 时 emit `subscription_expiring`（新事件）给品牌级角色。`emitRestricted` 泛化为 `emitSubscriptionEvent`（受限/到期共用）。前端通知标签补 1 条。

## 决策

- 读端点无独立权限码：任意品牌成员可见本品牌订阅态（自有数据）。
- 「当前订阅」= brand_subscriptions 最新一行（id DESC）——补偿/回调建新行、续期改现有行，最新行即当前态。

## 验收

`go test ./...` 全绿（含 grace→subscription_expiring / 多订阅取最新 DB 单测）；`pnpm --filter brand build` 通过。

## FR

- 长期过期订阅一轮 sweep 内 active→grace→restricted 会同时发「即将到期」+「受限」两条通知（可 dedup）。
- brand 自助续费（§17.3）落地后，banner「联系平台续期」应改为直达续费流程。
