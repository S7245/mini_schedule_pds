# Batch 16：订阅生命周期自动化（BrandSubscription 自动 grace/restricted，复用 Batch 15 asynq worker）

> 「3 批后端/品牌端收尾」第 2 批。交接：[batch-16-handoff.md](batch-16-handoff.md)。
> grill 设计树（难点 A status 门审计 / B 状态机+grace_ends_at / C 复用 worker / D 续期拉回）+ AskUserQuestion 拍板（2026-06-26 会话内，4 决策点**全取推荐项**）后写就。契约 approve 前不写代码。

## 1. 背景

`BrandSubscription` 状态目前**全手动、无任何时间驱动**：

- Batch 3 回调建 sub 只设 `status='active'` + `expires_at`（[commercial_repository.go:1133](../../backend/internal/infrastructure/persistence/commercial_repository.go)），`grace_ends_at` **恒 NULL**（续期/改状态路径也置 nil）。
- 平台手动 `ManualRenewBrandSubscription` / `UpdateBrandSubscriptionStatus` 改状态/续期。
- **无 cron、无自动转换**：过期 sub 的 `status` 永远停在 `'active'`，且 `grace_ends_at` 恒 NULL → **没有真正的宽限期在生效**。

蓝图要求（source of truth）：

- §4.5：到期→进宽限期（宽限内继续可用，提示续费）→宽限结束→受限状态（允许登录/查历史/处理已有预约，禁止新增）。
- §19/20.1（行 1330-1336）：默认宽限期 **7 天，可系统配置**；到期→`GracePeriod`，宽限结束→`Restricted`；`Frozen` 不自动延长。
- §20.3（行 1385-1393）：`Active → GracePeriod → Restricted → Frozen → Expired`。
- §23.1：套餐到期进入宽限期，宽限期结束后禁止新增预约和资源。
- §24.3：品牌订阅状态变更须写 OperationLog。
- `GO_BACKEND_LANGUAGE_DESIGN.md` §11：**明确把「BrandSubscription 自动进入 GracePeriod/Restricted/Expired 的定时任务」列为缺口**——本批正是填它。

本批接入 **Batch 15 已落地的 asynq worker**（加第 2 个 periodic task），把订阅主状态机从手动转为时间驱动，并**放宽 `SubscriptionGuard` 让宽限期保持可用**。

## 2. 拍板结果（决策点，全取推荐项）

| # | 决策 | 结论（推荐项） |
|---|---|---|
| 1 | 本批范围 | **仅 `active→grace_period→restricted` 自动化**，`restricted` 即**终态**（数据留存、拒新增、仍可续期）。`expired`/`cancelled` 留人工/平台 override；自动 `restricted→expired`、支付异常人工补偿 **留 FR** |
| 2 | restricted 执行面 | **仅放宽现有 `SubscriptionGuard`**（Location/Staff/Learner 供给门）。grace=可用、restricted/expired/frozen/cancelled=拦。§4.5「禁止新增预约/场次/权益」的订阅门（现这三条路径从不查订阅 status）= **执行面缺口留 FR**（pre-existing，非本批引入） |
| 3 | grace_ends_at 来源 + 配置 | **cron 翻 `active→grace` 时算 `grace_ends_at = expires_at + grace_days`**（零 migration，grace 窗从到期起算）。`grace_days` = 新 `worker.grace_days`（默认 **7**，`MINI_SCHEDULE_WORKER_GRACE_DAYS` 可覆盖） |
| 4 | 前端 | **零 FE**：admin dashboard summary 自动受益（restricted 计数自动转准，镜像 Batch 15「进行中」徽标 13e 预置）；grace 可在订阅列表按 status 筛。brand 受限/续费 banner 留 FR |

## 3. 范围

### 做
- **放宽 `SubscriptionGuard.CheckAndCount`**：`status = 'active'` → `status IN ('active','grace_period')`，宽限仍由时间门 `(grace_ends_at > now OR (grace_ends_at IS NULL AND expires_at > now))` 把关。
- 新建 `internal/application/subscriptionlifecycle` 用例服务（**无 RBAC**，系统执行）：`RunSweep(ctx, now)`。
- 新建 `internal/interfaces/worker` 入站适配器：`subscription:sweep` handler。
- 扩 `commercial.Repository`（domain + 实现）4 个方法（两段 list + 两段逐 sub 系统事务转换）。
- `config.WorkerConfig` 加 `GraceDays` + `SubscriptionSweepCron` + keysToBind + 4 个 dev yaml 的 `worker` 段。
- `cmd/worker/main.go` 加第 2 个 service + handler + Scheduler entry（手动 DI，与 Batch 15 同进程）。

### 不做（留 FR）
- 自动 `restricted→expired`（§1389 列 Expired 态，但 §1335 只要求到 Restricted；自动触发语义未定，留 FR）。
- **§4.5 受限态对「新预约/新场次/新权益」的订阅门**（现 booking/classsession/entitlement repo 不查订阅 status；补全需在 3-5 个写入点加门 + 跨 13c–14b 回归，独立批/FR）。
- 支付异常人工补偿开通订阅（PROGRESS §3 P0，独立 admin 流程，非自动化）。
- restricted 品牌再购撞 `idx_brand_subscriptions_one_current` 唯一索引（pre-existing 边界，留 FR：再购前须平台手动 cancel/expire 旧 sub）。
- 到期前提醒通知（§4.5「到期前提醒」=站内/微信订阅消息，留 Batch 18 通知中心 / FR）。
- brand 后台受限 banner、grace 在 summary 单独计数。

## 4. 架构

### 4.1 worker 进程（复用 `cmd/worker`，加第 2 条 periodic task）
```
config.Load → logger → 手动 DI:
  db → bookingRepo  → sessionautomation.Service   → worker.SweepHandler            (Batch 15，不动)
  db → commercialRepo → subscriptionlifecycle.Service(graceDays) → worker.SubscriptionSweepHandler  (Batch 16，新增)
  asynq.Server(mux[session:sweep, subscription:sweep])           // 同一 Server 消费两类任务
  asynq.Scheduler.Register(sweep_cron, session:sweep)            // Batch 15
                 .Register(subscription_sweep_cron, subscription:sweep)  // Batch 16，新增
  同时 Run，SIGINT/SIGTERM 优雅退出
```
- **不跑 migration**（worker 是 schema 消费者）。
- Railway：仍是第 4 服务 `worker`，**replicas=1**（Scheduler 单例铁律）。本地 `CONFIG_PATH=configs/config-app.yaml go run ./cmd/worker`。
- 两条 cron 共享同一 asynq Server（concurrency 共用）；mux 按 task type 路由；asynq key 前缀 `asynq:`。

### 4.2 数据流（一次 subscription sweep）
```
Scheduler 每 ~1h（可配）→ enqueue Task("subscription:sweep")
  → Server → SubscriptionSweepHandler(ctx, task)
    → subscriptionlifecycle.Service.RunSweep(ctx, now=time.Now().UTC())
      phase 1（active→grace_period）:
        ids := repo.ListSubscriptionsDueForGrace(now)
              SELECT id FROM brand_subscriptions WHERE status='active' AND expires_at <= now ORDER BY id
        for id: repo.TransitionSubscriptionToGrace(id, now, graceDays)   // 每 sub 独立 tx，audit，失败隔离+幂等
      phase 2（grace_period→restricted）:
        ids := repo.ListSubscriptionsDueForRestricted(now)
              SELECT id FROM brand_subscriptions WHERE status='grace_period' AND grace_ends_at IS NOT NULL AND grace_ends_at <= now ORDER BY id
        for id: repo.TransitionSubscriptionToRestricted(id, now)         // 每 sub 独立 tx，audit，失败隔离+幂等
      返回 Summary{Graced, Restricted, Skipped, Failed} → handler 记日志
```
> **一轮到位**：长期过期的 active sub（`expires_at` 与 `expires_at+grace_days` 均 ≤ now，如 grace_days=0 或久未跑 cron）会在同一轮 phase1 翻 grace（grace_ends_at≤now）、phase2 再翻 restricted（自愈，镜像 Batch 15 短课 `scheduled→completed`）。phase1 先于 phase2。

### 4.3 逐 sub 系统事务转换（镜像 Batch 15 `EndSessionSystem`）
两个转换方法各自：`SELECT FOR UPDATE` 锁 sub 行（**按 id，无 brand 过滤，系统跨品牌**）→ **状态守卫**（幂等关键，并发已被平台改动则空操作）→ UPDATE → `audit.Write(actor=system)`：

- `TransitionSubscriptionToGrace(tx, id, now, graceDays)`：守卫 `status='active' AND expires_at<=now`（否则 ROLLBACK，返 `(false,nil)` = 良性 skip）→ `UPDATE status='grace_period', grace_ends_at = expires_at + grace_days 天` → audit `action="brand_subscription.auto_grace_period"`、`actor_type='system'`、`actor_id` NULL、`brand_id` 填、before/after={status,grace_ends_at}。
- `TransitionSubscriptionToRestricted(tx, id, now)`：守卫 `status='grace_period' AND grace_ends_at IS NOT NULL AND grace_ends_at<=now`（否则 skip）→ `UPDATE status='restricted'` → audit `action="brand_subscription.auto_restricted"`、actor=system。

返回 `(transitioned bool, err error)`：守卫不过 = `(false,nil)` 良性 skip（无新错误码，比 Batch 15 error-sentinel 更干净，因转换仅 worker 调用）；DB 错 = `(false,err)`。`now` 参数化（可控时钟）。

> **frozen/cancelled 天然不碰**：phase1 WHERE `status='active'`、phase2 WHERE `status='grace_period'`，扫描查询本就排除 frozen/cancelled/expired/restricted（人工权威，镜像 Batch 15 §22.6 不自动扣课的边界）。`grace_ends_at = expires_at + N天` 用 `AddDate(0,0,graceDays)`（日历日，与 `computeSubscriptionExpiry` 一致）。
> **唯一索引安全**：`idx_brand_subscriptions_one_current UNIQUE(brand_id) WHERE status IN(active,grace,restricted,frozen)`——两个转换都在该集合内移动**同一行**，不新增行 → 不撞唯一约束。

### 4.4 SubscriptionGuard 放宽（难点 A，唯一执行面改动）
门审计结论（全仓 grep）：**`SubscriptionGuard.CheckAndCount`（[subscription_guard.go:75](../../backend/internal/application/commercial/subscription_guard.go)）是唯一一个把 `subscription.status='active'` 当资源供给门的点**；class_session / entitlement / booking repo 完全不查订阅 status。改动：
```go
// 旧
Where("brand_id = ? AND status = ? AND (grace_ends_at > ? OR (grace_ends_at IS NULL AND expires_at > ?))",
    brandID, "active", now, now)
// 新
Where("brand_id = ? AND status IN ? AND (grace_ends_at > ? OR (grace_ends_at IS NULL AND expires_at > ?))",
    brandID, []string{"active", "grace_period"}, now, now)
```
六态走查（time 门提供纵深防御，cron 滞后不漏）：

| sub 状态 | grace_ends_at / expires_at | status IN | 时间门 | 结果 |
|---|---|---|---|---|
| active | expires_at 未来、grace NULL | ✓ | true | **可用** |
| active | expires_at 已过、grace NULL（cron 未跑）| ✓ | false | **拦**（读时硬限保留）|
| grace_period | grace_ends_at 未来 | ✓ | true | **可用**（§4.5 宽限可用）|
| grace_period | grace_ends_at 已过（cron 将翻 restricted）| ✓ | false | **拦**（防御）|
| restricted / expired / frozen / cancelled | —— | ✗ | —— | **拦** |

### 4.5 续期/解冻拉回 active（难点 D：核对结论 = 已正确，本批不改）
`ManualRenewBrandSubscription`（[:360-368](../../backend/internal/infrastructure/persistence/commercial_repository.go)）已 `status != frozen → status='active'` + `grace_ends_at=nil`，restricted/grace 续期即回 active；`UpdateBrandSubscriptionStatus` 手动置 active 也清 grace_ends_at（:469-470）。**交接担心的「未拉回 active」实测已处理，本批零改动**（仅复跑回归证放宽 guard 后续期路径仍正确）。

## 5. 契约

**asynq 任务**

| 任务类型 | payload | 触发 | 处理器 |
|---|---|---|---|
| `subscription:sweep` | 空（`nil`） | Scheduler cron `@every 1h`（可配 `worker.subscription_sweep_cron`） | `worker.SubscriptionSweepHandler` → `subscriptionlifecycle.Service.RunSweep(now)` |

> session 时间分钟级敏感（`@every 1m`），订阅转换天级粒度（grace 7 天）→ 默认 `@every 1h` 足够、更轻。重试默认 asynq 策略；systemic 查询失败返 error 触发重试，单 sub 转换失败 log+continue（下一 tick 自愈）。handler 幂等。

**新增/改动方法**

| 层 | 文件 | 方法 | 说明 |
|---|---|---|---|
| app（改）| `internal/application/commercial/subscription_guard.go` | `CheckAndCount` 的 WHERE | `status='active'` → `status IN ('active','grace_period')` |
| domain（改）| `internal/domain/commercial/repository.go` | `Repository` 加 4 法 | `ListSubscriptionsDueForGrace` / `TransitionSubscriptionToGrace` / `ListSubscriptionsDueForRestricted` / `TransitionSubscriptionToRestricted` |
| infra（改）| `internal/infrastructure/persistence/commercial_repository.go` | 实现 4 法 | 锁行 + 状态守卫 + audit(system) + `(bool,error)` |
| app（新）| `internal/application/subscriptionlifecycle/service.go` | 窄 `Repository` 接口 + `NewService(repo,graceDays,log)` + `RunSweep(ctx,now) (Summary,error)` | 无 RBAC；按 now 编排两段 |
| interfaces（新）| `internal/interfaces/worker/subscription_sweep_handler.go` | `TaskSubscriptionSweep` / `NewSubscriptionSweepTask` / `NewSubscriptionSweepHandler` / `Handle` | asynq 入站适配器 |
| config（改）| `internal/infrastructure/config/config.go` | `WorkerConfig.GraceDays` + `.SubscriptionSweepCron` + keysToBind | 见下 |
| cmd（改）| `cmd/worker/main.go` | 第 2 个 service + handler + scheduler entry | 手动 DI |

**配置（`config.WorkerConfig` 扩展）**

```yaml
worker:
  sweep_cron: "@every 1m"                  # Batch 15：场次扫描周期（不动）
  subscription_sweep_cron: "@every 1h"     # Batch 16：订阅生命周期扫描周期（天级粒度，小时足够）
  concurrency: 4                           # asynq Server 并发（共用）
  grace_days: 7                            # Batch 16：默认宽限期天数（§1334 可系统配置；env 覆盖）
```
keysToBind 加 `worker.subscription_sweep_cron`、`worker.grace_days`；env 覆盖 `MINI_SCHEDULE_WORKER_SUBSCRIPTION_SWEEP_CRON`、`MINI_SCHEDULE_WORKER_GRACE_DAYS`。Redis 复用既有 `redis` 段。默认兜底：cron 空→`@every 1h`，grace_days≤0→7。

**前端页面模块**

| 页面/模块 | 类型 | 关键字段/操作 |
|---|---|---|
| （无改动）| —— | admin dashboard summary 已读 status 统计（restricted/frozen 计数 [:610](../../backend/internal/infrastructure/persistence/commercial_repository.go)、即将到期=active∈[now,+7d] [:605]）。本批让自动化开始产出 grace_period/restricted，summary 自动转准（restricted 计数从恒≈0 变实际）。grace_period sub 可在订阅列表按 status 筛查。 |

**前端实现约束**：本批零前端改动。

## 6. 幂等 / 并发 / 失败模式

| 风险 | 处理 |
|---|---|
| Scheduler 多副本重复 enqueue | 独立 worker replicas=1（部署铁律）+ handler 幂等 |
| sweep 重叠 / 重试 | 两个转换方法状态守卫（`active`/`grace_period` WHERE + 锁后复验）均幂等；已翻 sub 掉出下轮扫描；重跑空操作（`(false,nil)` skip）|
| 单 sub 转换失败 | 每 sub 独立 tx，失败隔离；log+continue，下一 tick 自愈 |
| cron 翻 status 与平台手动 `UpdateBrandSubscriptionStatus`/`ManualRenew` 并发 | 逐 sub `SELECT FOR UPDATE` 锁行 + 锁后状态守卫复验：平台已改（如续期回 active）→ 守卫不过 → 良性 skip，不覆盖人工权威 |
| frozen/cancelled 人工态 | phase1 WHERE `status='active'`、phase2 WHERE `status='grace_period'` 天然排除（人工权威，镜像 Batch 15 §22.6）|
| 时钟可控（测试）| `now` 作 repo 扫描 + 转换守卫查询参数；单测注入固定 now；live 验收用「过去 expires_at/grace_ends_at 数据」免等真实墙钟 |
| 唯一索引 | `idx_brand_subscriptions_one_current` 仅约束 brand 内「一个 current sub」，转换在集合内移动同一行不新增 → 不触发 |
| 放宽 guard 后过期绕过 | 时间门 `(grace_ends_at>now OR (grace_ends_at IS NULL AND expires_at>now))` 不变 → active+已过期仍读时拦（纵深防御）|

## 7. SubscriptionGuard 放宽透明性（grill + code-review 收口，难点 A = Batch 15 F1 重演）

Batch 15 教训：**引入「显示态」枚举要审计所有 `status='xxx'` 守卫**。本批把 `grace_period` 从「纯枚举值」变成「guard 视同可用」的活跃态：

- **门审计已穷尽**：`SubscriptionGuard.CheckAndCount` 是唯一订阅 status 资源门；其余 `BrandSubscriptionStatusActive` 引用全是续期/summary/回调**写入**路径（非门），放宽 guard 不波及。
- **放宽语义**：`active+grace=可用`（grace 由时间门 `grace_ends_at>now` 把关），`restricted/expired/frozen/cancelled=拦`。保留 Batch 16 前「active+已过期=拦」（时间门）。
- **零回归证明**：复跑 Batch 4(Location)/5(Staff)/13a(Learner) 配额 + 过期绕过测试 + guard 自身单测；admin summary 行 605/610 语义核对（即将到期仍只数 active；restricted 计数自动转准；grace 暂不入 summary 桶，留 FR）。

## 8. 验收方式（会话内，主线程自测 = 业务验收）

可控时钟靠**数据**（过去 `expires_at`/`grace_ends_at`），非墙钟：
1. 起 `CONFIG_PATH=configs/config-app.yaml MINI_SCHEDULE_WORKER_SUBSCRIPTION_SWEEP_CRON='@every 5s' MINI_SCHEDULE_WORKER_GRACE_DAYS=7 go run ./cmd/worker`（连 dev DB v12 + Redis）。
2. 造数据（psql，brand21 owner 18816820405 / 讯美广场 loc1）：
   - sub A：`status='active'`、`expires_at` 过去 → 期望 sweep 后 `grace_period` + `grace_ends_at = expires_at + 7d`。
   - sub B：`status='grace_period'`、`grace_ends_at` 过去 → 期望 sweep 后 `restricted`。
   - sub C：`status='active'`、`expires_at` 过去很久（如 -30d，使 +7d 仍 ≤ now）→ 期望**一轮内** `restricted`。
   - sub D：`status='frozen'`、`expires_at` 过去 → 期望 **不动**（人工权威）。
3. 等 1 个 sweep tick（或手动 enqueue）。
4. psql 实查：A=`grace_period`（grace_ends_at 已设）；B/C=`restricted`；D=`frozen`；`operation_logs` 有 `brand_subscription.auto_grace_period` / `auto_restricted`、`actor_type='system'`、`actor_id` NULL。
5. 幂等：再跑一轮，A/B/C 状态不变、**无新 audit**（已翻 sub 掉出扫描）。
6. **guard 放宽验证**：对 grace_period 品牌（grace_ends_at 未来）经 brand API 加 Location → 成功（时间门未过）；对 restricted 品牌加 Location → `SUBSCRIPTION_RESTRICTED`。
7. **续期拉回**：平台对 restricted sub `ManualRenew` → psql 实查 `status='active'` + `grace_ends_at=NULL` + `expires_at` 顺延 → 该品牌又可加 Location。

## 9. 测试（DB 单测，真实 Postgres `newMigratedTestDB`）

- `ListSubscriptionsDueForGrace`：active+expires_at≤now 命中；active+未来、grace_period、frozen、cancelled 不命中。
- `TransitionSubscriptionToGrace`：active+过期→grace_period+grace_ends_at=expires_at+N天（actor=system，audit actor_id NULL）；并发已非 active（守卫不过）→`(false,nil)`；幂等（再调空操作）；不动 frozen。
- `ListSubscriptionsDueForRestricted`：grace_period+grace_ends_at≤now 命中；grace_ends_at 未来/NULL、active 不命中。
- `TransitionSubscriptionToRestricted`：grace_period+grace 过期→restricted（audit system）；守卫不过→skip；幂等。
- `RunSweep`（app service，fake/real）：注入 now，断言 Graced/Restricted/Skipped/Failed 计数 + 两段编排顺序（含「一轮 active→grace→restricted」自愈用例）。
- `SubscriptionGuard`（放宽）：active+未来=放行；active+过期=`SUBSCRIPTION_RESTRICTED`；grace_period+grace 未来=放行；grace_period+grace 过期=拦；restricted/expired/frozen/cancelled=拦；超额仍 `QUOTA_EXCEEDED`。
- **零回归复跑**：Batch 4/5/13a 的 Location/Staff/Learner 配额测试、guard 既有单测、commercial 续期/改状态测试、Batch 15 `sessionautomation` 全绿。

## 10. 流程

grill ✅ → AskUserQuestion 拍板 ✅ → **本契约（会话内 approve）** → 测试场景 `batch-16-subscription-lifecycle-tests.md` → 主线程逐 task TDD commit → `go build ./... && go test ./...`（零回归）→ `/code-review`（high，多 finder + 验证）→ 业务验收（会话内主线程自测）→ 更新 PROGRESS + 三库 `.learnings` → 三仓库 push + backend `dev` FF `main`（pds 直接 main）。

- 预计**零 migration**（status CHECK + 索引 + grace_ends_at 列全就绪，dev DB 保持 v12）、**零新权限码**、**零新错误码**（转换守卫用 `(bool,error)`）、**零前端**。
- **后端为主**批（worker 第 2 task + 状态机 + guard 放宽）。
