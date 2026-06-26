# Batch 16 实现交接 prompt —— 订阅生命周期自动化（契约已写、待 approve→TDD 实现）

> 新 session 直接读本文件按此执行。**本批 grill + 拍板 + 契约已全部完成**（上一 session 2026-06-26）。本 session 的工作 = **会话内 approve 契约 → 逐 task TDD 实现 → 验收 → push**。
> ⚠️ **不要重新 grill、不要重写契约、不要重开 AskUserQuestion**。4 决策点已拍板（全取推荐项，见下）。契约是 source of truth：[batch-16-subscription-lifecycle.md](batch-16-subscription-lifecycle.md)。

## 关键铁律（13a–15 惯例，严格遵守）
- **会话内 approve / 验收，不发邮件**（CLAUDE.md 写的「邮件停止点」被 classifier 拦，本项目已改会话内确认，13a–15 惯例）。
- 仓库：backend/web 在 `dev`，pds 在 `main`。DB 单测用真实 Postgres（`newMigratedTestDB`）。
- **主线程逐 task TDD commit**（先红→绿→单 task commit；用户不 spawn 实现 subagent）。
- 收尾：`go build ./... && go test ./...` → `/code-review`（high，多 finder + 验证）→ 业务验收（会话内主线程自测）→ 更新 PROGRESS + 三库 `.learnings` → 三仓库分别 push → backend 的 `dev` FF `main`（`git push origin dev:main`，pds 直接 main，本批零前端故 web 不动）。

## 先读（按序）
1. **本批契约**（source of truth，全部设计细节在此）：[batch-16-subscription-lifecycle.md](batch-16-subscription-lifecycle.md)。
2. `pds/CLAUDE.md`（每轮流程）+ `pds/PROGRESS.md` 的 **Batch 15 段**（姊妹批，同 asynq worker 范式）。
3. **Batch 15 实现 = 要镜像的范式**：
   - `backend/cmd/worker/main.go`（asynq Server+Scheduler，手动 DI，本批加第 2 个 service+handler+scheduler entry）。
   - `backend/internal/application/sessionautomation/service.go`（`RunSweep` 范式：扫描 + 逐行系统事务 + 失败隔离 + 幂等 + Summary）。
   - `backend/internal/interfaces/worker/sweep_handler.go`（asynq 入站适配器）。
   - `backend/internal/infrastructure/persistence/booking_repository.go:1081-1188`（`applyEndSession` / `EndSessionSystem` / `MarkSessionsInProgress` / `ListDueSessionIDs`——逐 sub 系统事务的精确范式）。
   - `backend/internal/audit/audit.go`（`ActorSystem` + `audit.Write`，actor_id=0 → NULL）。
4. **要改的现有代码**：
   - `backend/internal/application/commercial/subscription_guard.go:75`（唯一执行面改动，见设计 ①）。
   - `backend/internal/domain/commercial/repository.go:182`（`Repository` 接口，加 4 法）+ `commercial.go:51-60`（6 个 `BrandSubscriptionStatus*` 常量已齐）。
   - `backend/internal/infrastructure/persistence/commercial_repository.go`（实现 4 法；参考既有 `ManualRenewBrandSubscription`:347 / `UpdateBrandSubscriptionStatus`:454 的 tx+audit 写法）。
   - `backend/internal/infrastructure/config/config.go:114`（`WorkerConfig`）+ `:162`（keysToBind）。

## 拍板结论（已锁定，不再问）
| # | 决策 | 结论 |
|---|---|---|
| 1 | 本批范围 | **仅 `active→grace_period→restricted` 自动化**，`restricted`=终态。expired/cancelled 留人工；自动 restricted→expired、支付异常补偿留 FR |
| 2 | restricted 执行面 | **仅放宽现有 `SubscriptionGuard`**（Location/Staff/Learner）。§4.5「禁止新增预约/场次/权益」的订阅门=执行面缺口留 FR（现这三路径从不查订阅 status，pre-existing） |
| 3 | grace_ends_at + 配置 | cron 翻 grace 时算 `grace_ends_at = expires_at + grace_days`；`worker.grace_days` 默认 **7**，env `MINI_SCHEDULE_WORKER_GRACE_DAYS` 可覆盖 |
| 4 | 前端 | **零 FE**（admin summary 自动转准；brand banner 留 FR）|

## 已核实事实（上一 session 勘察，不必重查）
- **零 migration**：`migrations/000003` 已含 `brand_subscriptions_status_valid` CHECK（6 态全）、`grace_ends_at TIMESTAMPTZ`（nullable）、`idx_brand_subscriptions_status_expires_at(status, expires_at)`。dev DB 保持 v12。
- **零新权限码 / 零新错误码 / 零前端**（转换守卫用 `(bool,error)` 返良性 skip，无需新错误码）。
- **难点 A 门审计已穷尽**：`SubscriptionGuard.CheckAndCount` 是唯一把 `subscription.status='active'` 当资源供给门的点；其余 `BrandSubscriptionStatusActive` 引用全是续期/summary/回调写入（非门）。
- **难点 D 已正确无需改**：`ManualRenewBrandSubscription:360-368` 已 `status != frozen → 'active'` + `grace_ends_at=nil`（restricted/grace 续期即回 active）；`UpdateBrandSubscriptionStatus:469-470` 置 active 也清 grace_ends_at。**交接里「未拉回 active 需补齐」是过时担心，实测已处理。本批不改它，仅复跑回归证放宽 guard 后仍正确。**
- **唯一索引安全**：`idx_brand_subscriptions_one_current UNIQUE(brand_id) WHERE status IN(active,grace,restricted,frozen)`——转换在集合内移动同一行，不撞。
- `persistence.NewCommercialRepository(db) commercial.Repository` 已存（worker DI 用它，同 `NewBookingRepository`）。

## 设计（已定，照此实现）

### ① 放宽 SubscriptionGuard（`subscription_guard.go:75`，唯一执行面改动）
```go
// 旧
Where("brand_id = ? AND status = ? AND (grace_ends_at > ? OR (grace_ends_at IS NULL AND expires_at > ?))",
    brandID, "active", now, now)
// 新
Where("brand_id = ? AND status IN ? AND (grace_ends_at > ? OR (grace_ends_at IS NULL AND expires_at > ?))",
    brandID, []string{"active", "grace_period"}, now, now)
```
语义：active+grace=可用（grace 由时间门 `grace_ends_at>now` 把关），restricted/expired/frozen/cancelled=拦，active+已过期=时间门拦（读时硬限保留，纵深防御）。

### ② commercial.Repository 加 4 法（domain 接口 + commercial_repository.go 实现）
```go
ListSubscriptionsDueForGrace(ctx, now) ([]int64, error)
    // SELECT id FROM brand_subscriptions WHERE status='active' AND expires_at <= now ORDER BY id ASC
TransitionSubscriptionToGrace(ctx, id, now, graceDays) (bool, error)
    // tx: SELECT FOR UPDATE WHERE id; 守卫 status='active' AND expires_at<=now（否则 return false,nil 良性 skip）;
    //     UPDATE status='grace_period', grace_ends_at = expires_at.AddDate(0,0,graceDays);
    //     audit.Write(ActorSystem, action="brand_subscription.auto_grace_period",
    //                 BrandID=&row.brand_id, Target{"brand_subscription", id}, before/after={status,grace_ends_at}); return true,nil
ListSubscriptionsDueForRestricted(ctx, now) ([]int64, error)
    // SELECT id FROM brand_subscriptions WHERE status='grace_period' AND grace_ends_at IS NOT NULL AND grace_ends_at <= now ORDER BY id ASC
TransitionSubscriptionToRestricted(ctx, id, now) (bool, error)
    // tx: 锁行; 守卫 status='grace_period' AND grace_ends_at IS NOT NULL AND grace_ends_at<=now（否则 false,nil）;
    //     UPDATE status='restricted'; audit.Write(ActorSystem, action="brand_subscription.auto_restricted", ...); return true,nil
```
镜像 `EndSessionSystem`（按 id 锁、无 brand 过滤、系统跨品牌、锁后状态守卫=幂等+并发安全）。`grace_ends_at` 用 `AddDate(0,0,graceDays)`（日历日，与 `computeSubscriptionExpiry` 一致）。

### ③ 新 application/subscriptionlifecycle/service.go（镜像 sessionautomation）
```go
type Repository interface { /* 上面 4 法（窄接口，commercial.Repository 满足） */ }
type Summary struct { Graced, Restricted, Skipped, Failed int }
func NewService(repo Repository, graceDays int, log *slog.Logger) *Service   // graceDays<=0 → 7
func (s *Service) RunSweep(ctx, now) (Summary, error)
    // phase1: ids:=ListSubscriptionsDueForGrace(now)（systemic err→return）; for id: TransitionSubscriptionToGrace(id,now,graceDays)
    //         ok→Graced++ / false→Skipped++ / err→Failed++ +log+continue
    // phase2: ids:=ListSubscriptionsDueForRestricted(now); for id: TransitionSubscriptionToRestricted(id,now) 同上 Restricted++
```
phase1 先于 phase2（长期过期 sub 一轮 active→grace→restricted 自愈）。

### ④ 新 interfaces/worker/subscription_sweep_handler.go
```go
const TaskSubscriptionSweep = "subscription:sweep"
func NewSubscriptionSweepTask() *asynq.Task                     // asynq.NewTask(TaskSubscriptionSweep, nil)
type SubscriptionSweepHandler struct{ svc *subscriptionlifecycle.Service; log *slog.Logger }
func NewSubscriptionSweepHandler(svc, log) *SubscriptionSweepHandler
func (h) Handle(ctx, *asynq.Task) error                         // svc.RunSweep(ctx, time.Now().UTC())，记 Summary 日志
```

### ⑤ config.WorkerConfig 扩展（config.go）
```go
type WorkerConfig struct {
    SweepCron             string `mapstructure:"sweep_cron"`              // 不动
    SubscriptionSweepCron string `mapstructure:"subscription_sweep_cron"` // 新：默认 @every 1h
    Concurrency           int    `mapstructure:"concurrency"`             // 不动
    GraceDays             int    `mapstructure:"grace_days"`              // 新：默认 7
}
// keysToBind 加 "worker.subscription_sweep_cron", "worker.grace_days"
```
含 `worker:` 段的 yaml 各加两行（`grep -rl '^worker:' backend/configs/` 找全；至少 config-app.yaml，worker 用它启动）：
```yaml
worker:
  subscription_sweep_cron: "@every 1h"
  grace_days: 7
```

### ⑥ cmd/worker/main.go 加第 2 条 periodic task（手动 DI）
```go
graceDays := cfg.Worker.GraceDays; if graceDays <= 0 { graceDays = 7 }
subCron := cfg.Worker.SubscriptionSweepCron; if subCron == "" { subCron = "@every 1h" }
subSvc := subscriptionlifecycle.NewService(persistence.NewCommercialRepository(db), graceDays, log)
subSweep := worker.NewSubscriptionSweepHandler(subSvc, log)
mux.HandleFunc(worker.TaskSubscriptionSweep, subSweep.Handle)
scheduler.Register(subCron, worker.NewSubscriptionSweepTask())   // 在既有 session:sweep Register 之后
```

## 逐 task TDD 顺序（单 task commit）
1. **放宽 SubscriptionGuard**（设计 ①）：改/加 guard 单测（active+未来放行、active+过期拦、grace 窗内放行、grace 窗外/restricted/expired/frozen/cancelled 拦、超额仍 QUOTA_EXCEEDED）→ 改 WHERE → 绿 → 复跑 Batch 4/5/13a 配额测试证零回归 → commit。
2. **repo 4 法 + DB 单测**（设计 ②）：真实 PG（`newMigratedTestDB`），断言转换 + grace_ends_at 计算 + audit actor=system/actor_id NULL + 守卫不过 skip + 幂等 + frozen 不动 → commit。
3. **subscriptionlifecycle.Service.RunSweep + 单测**（设计 ③）：fake/real 注入 now，断言 Graced/Restricted/Skipped/Failed + 两段编排 + 「一轮自愈」用例 → commit。
4. **worker handler + config + cmd/worker 接线**（设计 ④⑤⑥）：handler 路由冲突冒烟；config 默认兜底 → commit。
5. `go build ./... && go test ./...` 全绿，复跑 **13c/13d/13e/14a/15** + Batch 4/5/13a 证零回归 → 若有修单独 commit。

## 验收方式（会话内主线程自测 = 业务验收，靠数据控时钟免等墙钟）
1. 起：`CONFIG_PATH=configs/config-app.yaml MINI_SCHEDULE_WORKER_SUBSCRIPTION_SWEEP_CRON='@every 5s' MINI_SCHEDULE_WORKER_GRACE_DAYS=7 go run ./cmd/worker`（连 dev DB v12 + Redis）。**先重启确保最新二进制**（旧二进制掩盖改动，见 memory）。
2. 造数据（psql；brand21 owner `18816820405 / admin123`、门店 讯美广场 loc1）：
   - A：`status='active'`、`expires_at` 过去 → 期望 `grace_period` + grace_ends_at=expires_at+7d。
   - B：`status='grace_period'`、`grace_ends_at` 过去 → 期望 `restricted`。
   - C：`status='active'`、`expires_at`=-30d（+7d 仍≤now）→ 期望**一轮内** `restricted`。
   - D：`status='frozen'`、`expires_at` 过去 → 期望**不动**。
3. 跑 1 轮 → psql 实查状态 + `operation_logs`（`brand_subscription.auto_grace_period`/`auto_restricted`、actor_type='system'、actor_id NULL）。
4. **幂等**：再跑一轮，A/B/C 不变、无新 audit。
5. **guard 放宽**：grace 品牌（grace_ends_at 未来）经 brand API 加 Location→成功；restricted 品牌加 Location→`SUBSCRIPTION_RESTRICTED`。
6. **续期拉回**：平台对 restricted sub `ManualRenew` → psql 查 `active` + grace_ends_at NULL + expires_at 顺延 → 又可加 Location。

## 留 FR（写入三库 `.learnings/FEATURE_REQUESTS.md`）
- **§4.5「禁止新增预约/场次/权益」的订阅门**（booking/classsession/entitlement repo 不查订阅 status，本批 restricted 只拦 Location/Staff/Learner 供给）——比本批更大的执行面缺口，pre-existing。
- 自动 `restricted→expired`（§1389 列 Expired 态、§1335 只到 Restricted，触发语义未定）。
- 支付异常人工补偿开通订阅（PROGRESS §3 P0）。
- restricted 品牌再购撞 `idx_brand_subscriptions_one_current`（再购前须平台手动 cancel/expire 旧 sub）。
- brand 后台受限/续费 banner；grace_period 在 admin summary 单独计数（现 grace 暂不入任何桶）；到期前提醒通知（Batch 18）。

## 收尾 push
backend：`go test` 全绿后 commit → push `dev` → `git push origin dev:main`。pds：更新 `PROGRESS.md`（Batch 16 段，紧接 Batch 15）+ 本批 learnings → push `main`。web：本批零改动。

---
**第一步：读契约 [batch-16-subscription-lifecycle.md](batch-16-subscription-lifecycle.md)，向用户简述要点请其会话内 approve（强制停止点，不发邮件）。approve 后按上面「逐 task TDD 顺序」开工。grill/拍板/契约已完成，勿重做。**
