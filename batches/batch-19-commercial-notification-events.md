# Batch 19 契约 —— 商业化通知事件扩展（§20.13 剩余 2 类）

## Lifecycle

| 阶段 | 时间戳 | 触发人 |
|---|---|---|
| draft | 2026-07-12 | claude |
| approved | 2026-07-12（2 决策点会话内拍板，全取推荐） | user |
| implementing | 2026-07-12（T1–T4，5 commits） | claude |
| static-verified | 2026-07-12（go test ./... 38 包绿；pnpm --filter brand build 通过） | claude |
| scenario-verified | 2026-07-12（RunSweep restricted→subscription_abnormal emit 单测；isQuotaExceeded 检测单测） | claude |
| done | 2026-07-12 | claude |

## 范围与决策

接 Batch 18 通知基建，补 §20.13 后台站内通知剩余 2 类事件（复用 Emitter/Resolver/notifications 表/`/notifications` 页，无新页面、无 migration）：

1. **额度即将超限 `quota_near_limit`**（决策 1 = 推荐）：location/staff/learner 创建**因 `QUOTA_EXCEEDED` 被拦截时** emit「额度已达上限」。零 usage plumbing（只在创建失败处检错码），语义为「已达上限」。dedup 留 FR。
2. **订阅异常 `subscription_abnormal`**（决策 2 = 推荐）：`subscriptionlifecycle` sweep 把订阅转入 **restricted（受限）** 时 emit。仅 restricted（真受限），不含 grace_period 预警。
3. **收件人**（既定）：两类事件**无 location** → RecipientResolver 传 `locationID=nil` 只命中**品牌级角色**（owner/admin/course_operator/finance_support，持 notification.view）；location 级角色（店长/前台/instructor）不收。
4. **集成**（既定）：best-effort async（`EmitAsync`），同 Batch 18。支付回调失败的「支付异常」留 FR（当前支付 mock）。

## 现状核查（grounded）

- `SubscriptionGuard.CheckAndCount`（`subscription_guard.go`）返回 `apperr.ErrQuotaExceeded`（409）当 count≥max；深埋 repo `Create` tx 内。→ 在 location/staff/learner **brand handler** create 处检 `errors.As` + `Code==ErrQuotaExceeded` 即可 emit，无需上浮 usage。
- `subscriptionlifecycle.RunSweep`（`subscriptionlifecycle/service.go`，worker）phase2 `TransitionSubscriptionToRestricted(id) (ok, err)` → **无 brand_id**。新增窄接口方法 `GetSubscriptionBrandID(id)` 供 restricted 成功后查 brand_id emit（一次轻量查询/受限转换，rare）。
- 收件人解析 `locationID=nil` 路径已在 Batch 18 实现（`scopeCovers(all_brand, nil)=true`）。

## 契约

### 新增 EventType（扩 `domain/notification`）

| event_type | DefaultTitle | 触发点 | related_type/id |
|---|---|---|---|
| `quota_near_limit` | 套餐额度已达上限 | location/staff/learner create 被 QUOTA_EXCEEDED 拦截（brand handler） | subscription / 0 |
| `subscription_abnormal` | 订阅受限 | subscriptionlifecycle sweep → restricted（worker，actor=system） | brand_subscription / subID |

**收件人**：品牌级角色（owner/admin/course_operator/finance_support）持 notification.view；`ExcludeBrandUserID` = 触发 actor（quota：创建者 brand_user；subscription：系统 0）。

### 无 API / migration / 前端页面变更

- 无新端点（读走 Batch 18 的 `/notifications`）。
- 无 migration（复用 notifications 表 + 已有权限码；event_type 无 CHECK 约束）。
- 前端仅：`/notifications` 页 `EVENT_LABELS` 加 2 条中文映射（`quota_near_limit`→「额度已达上限」、`subscription_abnormal`→「订阅受限」）。

## 任务拆解（plan，TDD 逐 task commit）

**Stage A — 后端**（完成跑 `/code-review`）
- **T1** domain：`EventType` 加 2 值 + `Valid()` + `DefaultTitle()`；`inputs.go` 加 `QuotaNearLimitInput(brandID, kindLabel, actorID)` + `SubscriptionAbnormalInput(brandID, subID)`；单测。
- **T2** quota emit：location/staff/learner **brand handler** 注入 emitter，create 失败且 `Code==ErrQuotaExceeded` → emit（kind 中文标签）；Wire。handler/helper 测试。
- **T3** subscription emit：`subscriptionlifecycle.Repository` 加 `GetSubscriptionBrandID` + commercial repo 实现；Service 加 emitter setter，RunSweep restricted 成功后 emit；worker wire 复用 Batch 18 emitter 链。RunSweep 单测断言 emit + 复跑 16 sweep 零回归。

**Stage B — 前端**（完成跑 `/code-review`）
- **T4** `/notifications` 页 `EVENT_LABELS` 加 2 条；`pnpm --filter @mini-schedule/brand build`。

**验收**：`go test ./...` 全绿零回归；集成/单测断言 restricted sweep 落 subscription_abnormal 行（品牌级收件人）、create 超额落 quota_near_limit 行。
