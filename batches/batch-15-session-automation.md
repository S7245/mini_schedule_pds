# Batch 15：场次状态自动化（asynq 定时任务框架接入）

> 「3 批后端/品牌端收尾」第 1 批。交接：[batch-15-handoff.md](batch-15-handoff.md)。
> brainstorm（brainstorming skill）+ grill + AskUserQuestion 拍板（2026-06-26 会话内，4 决策点**全取推荐项**）后写就。契约 approve 前不写代码。

## 1. 背景

场次状态目前**全手动**：`Create` 直落 `scheduled`（跳过 draft），13e 加了**显式手动** `EndSession`（`scheduled/in_progress→completed` + 未签到 `booked` 批量→`pending_no_show`）与 `ConfirmNoShow`（爽约扣课/退课）。**无 `scheduled→in_progress` 转换、无任何时间驱动**。本批接入 **asynq**（Redis-backed 队列 + 周期调度器）作为后期定时自动化框架的**首个落点**，把场次主状态机从手动转为时间驱动。

asynq 未装（`go.mod` 无 `hibiken/asynq`）；backend 现 3 个 API 服务、**无 worker**；Redis 已接好（`go-redis/v9`，RBAC 缓存在用）→ asynq 底座现成。

## 2. 拍板结果（决策点）

| # | 决策 | 结论（推荐项） |
|---|---|---|
| 1 | 本批范围 | **框架 + 自动 `scheduled→in_progress`（显示态）+ 自动「结束场次」（→`completed` + 产 `pending_no_show`，定时调系统版 EndSession）**。自动确认爽约/扣课 **不做**（§22.6 硬约束，留 FR） |
| 2 | worker 部署 | **独立 `cmd/worker`**（asynq Scheduler + Server 同进程），Railway 加第 4 服务 replicas=1（Scheduler 单例最干净） |
| 3 | 自动化形态 | **periodic 扫描批量**（Scheduler 每 ~1min enqueue 一个 sweep 任务；handler 批量处理「到点未处理」场次） |
| 4 | migration / 派生态 | **零 migration**：Full/Closed 继续读时算（现状），asynq 全状态存 Redis 无需 PG 表；`in_progress` 已在 `class_sessions_status_valid` CHECK 内。dev DB 保持 v12 |

## 3. 范围

### 做
- 引入 `github.com/hibiken/asynq` 依赖。
- 新建 `cmd/worker`（独立进程）：asynq `Server`（消费 sweep 任务）+ `Scheduler`（按 cron enqueue）+ 优雅退出。
- 新建 `internal/application/sessionautomation` 用例服务（**无 RBAC**，系统执行）：`RunSweep(ctx, now)`。
- 新建 `internal/interfaces/worker` 入站适配器：asynq sweep handler（`func(ctx, *asynq.Task) error`）。
- 扩 `booking.Repository`（domain + 实现）3 个方法：
  - `MarkSessionsInProgress(ctx, now) (int64, error)` —— 批量 `scheduled→in_progress`。
  - `ListDueSessionIDs(ctx, now) ([]int64, error)` —— 列「到点未结束」场次 id（跨品牌）。
  - `EndSessionSystem(ctx, sessionID) (*EndSessionResult, error)` —— 系统版结束场次（actor=system）。
- 抽 `EndSession` 与 `EndSessionSystem` 的共享核心 `applyEndSession(tx, sess, actor)`（actor 参数化，镜像 14a `placeBooking` 模式）。
- `config.WorkerConfig` + keysToBind + 4 个 dev yaml 的 `worker` 段。

### 不做（留 FR）
- 自动确认爽约 + 扣课（§22.6「第一版避免自动扣课造成争议，默认需员工确认」硬约束）。13e 手动 `EndSession`/`ConfirmNoShow` **保留为 override/兜底**。
- Full/Closed 派生态落库。
- BrandSubscription 自动进 grace/expired（设计文档 §11，未来同框架复用）。
- 候补自动转正、per-session 精确定时、按品牌时区调度。

## 4. 架构

### 4.1 worker 进程（`cmd/worker`）
```
config.Load → logger → initializeSessionAutomation(Wire: db→bookingRepo→sessionautomation.Service)
  → asynq.Server(RedisClientOpt{cfg.Redis}, mux[sweepHandler])  // 消费
  → asynq.Scheduler(RedisClientOpt{cfg.Redis}).Register(cfg.Worker.SweepCron, NewTask("session:sweep"))  // 生产
  → 同时 Run，SIGINT/SIGTERM 优雅退出（先停 Scheduler 再 Shutdown Server）
```
- **不跑 migration**（worker 是 schema 消费者；API 服务负责 migrate）。
- asynq 用 `RedisClientOpt`（Addr/Password/DB 来自 `cfg.Redis`，与 app 的 `*redis.Client` 各自连接同一 Redis）；asynq key 前缀 `asynq:`，与 RBAC 缓存 `rbac:` 无冲突。
- Railway：第 4 服务 `worker`，**replicas=1**（Scheduler 单例铁律）。本地 `go run ./cmd/worker`（需 `CONFIG_PATH=configs/config-app.yaml` 或任一含 redis/database 段的 config）。

### 4.2 数据流（一次 sweep）
```
Scheduler 每 1min → enqueue Task("session:sweep")
  → Server 取任务 → sweepHandler(ctx, task)
    → sessionautomation.Service.RunSweep(ctx, now=time.Now().UTC())
      1) started := repo.MarkSessionsInProgress(now)
         UPDATE class_sessions SET status='in_progress'
         WHERE status='scheduled' AND starts_at <= now AND ends_at > now   // 批量、幂等、无 audit（显示态）
      2) ids := repo.ListDueSessionIDs(now)
         SELECT id FROM class_sessions WHERE status IN ('scheduled','in_progress') AND ends_at <= now
      3) for id in ids: repo.EndSessionSystem(id)   // 每场次独立 tx，失败隔离 + 幂等
      4) 返回 Summary{started, ended, skipped, failed} → handler 记日志
```

### 4.3 EndSessionSystem（复用 13e 事务）
`EndSessionSystem(ctx, sessionID)` 锁 session 行（`WHERE id=?`，**无 brand 过滤、无 scope**，系统跨品牌）→ 读 `brand_id` → 调共享核心 `applyEndSession(tx, &sess, audit.Actor{Type: ActorSystem})`：
- 状态守卫：仅 `scheduled|in_progress→completed`，否则 `SESSION_NOT_ENDABLE`（**幂等关键**：第二遍是 completed → 空操作）。
- `UPDATE class_sessions SET status='completed'`。
- `UPDATE bookings SET status='pending_no_show' WHERE class_session_id=? AND status='booked'`（RowsAffected = count）。
- `audit.Write(actor=system)` → `operation_logs.actor_type='system'`、`actor_id` NULL（`ActorSystem` 已存，DB CHECK 已含 'system'，无 brand_users FK 问题）。
- **不动** `booked_count`（只取消退名额）、**不动** hold、**不**产 no_show（§22.6）。

`EndSession`（13e 现有，brand_user 路径）改为调同一 `applyEndSession(tx, &sess, audit.Actor{ActorBrandUser, actorID})` —— 行为不变，复跑 13e 测试证零回归。

## 5. 契约

**asynq 任务**

| 任务类型 | payload | 触发 | 处理器 |
|---|---|---|---|
| `session:sweep` | 空（`{}`） | Scheduler cron `@every 1m`（可配） | `worker.SweepHandler` → `sessionautomation.Service.RunSweep(now)` |

> enqueue 加 `asynq.Unique(2*sweepInterval)`：重叠 tick 不堆积（即便堆积，handler 幂等亦安全）。重试默认 asynq 策略；systemic 查询失败返 error 触发重试，单场次 EndSession 失败 log+continue（下一 tick 自愈）。

**新增/改动方法**

| 层 | 文件 | 方法 | 说明 |
|---|---|---|---|
| domain | `internal/domain/booking/booking.go` | `Repository` 加 `MarkSessionsInProgress` / `ListDueSessionIDs` / `EndSessionSystem` | 接口扩 3 法；`EndSessionResult` 复用 |
| infra | `internal/infrastructure/persistence/booking_repository.go` | 实现 3 法 + 抽 `applyEndSession(tx,sess,actor)` | `EndSession` 改调共享核心 |
| app | `internal/application/sessionautomation/service.go`（新） | `NewService(repo)` / `RunSweep(ctx,now) (Summary,error)` | 无 RBAC；按 now 编排 |
| interfaces | `internal/interfaces/worker/sweep_handler.go`（新） | `NewSweepHandler(svc)` / `Handle(ctx,*asynq.Task) error` | asynq 入站适配器 |
| cmd | `cmd/worker/{main.go,wire.go,wire_gen.go}`（新） | `initializeSessionAutomation` + asynq runtime | 独立进程 |
| config | `internal/infrastructure/config/config.go` | `WorkerConfig` + keysToBind | 见下 |

**配置（`config.WorkerConfig`）**

```yaml
worker:
  sweep_cron: "@every 1m"   # Scheduler 周期；session 时间非秒级敏感
  concurrency: 4            # asynq Server 并发 worker 数
```
keysToBind 加 `worker.sweep_cron`、`worker.concurrency`；env 覆盖 `MINI_SCHEDULE_WORKER_SWEEP_CRON` 等。Redis 复用既有 `redis` 段。

**前端页面模块**

| 页面/模块 | 类型 | 关键字段/操作 |
|---|---|---|
| （无改动） | —— | `/schedule` **已**支持 `in_progress`「进行中」徽标（绿）+ `completed`「已完成」徽标 + 状态筛选（13e 预置）。本批仅让自动化开始产出这些状态，UI 自动渲染。 |

**前端实现约束**：本批零前端改动。

## 6. 幂等 / 并发 / 失败模式

| 风险 | 处理 |
|---|---|
| Scheduler 多副本重复 enqueue | 独立 worker replicas=1（部署铁律）+ `asynq.Unique` |
| sweep 重叠 / 重试 | `MarkSessionsInProgress`（`WHERE status='scheduled'`）+ `EndSessionSystem`（状态守卫）均幂等；重跑空操作 |
| 单场次 EndSession 失败 | 每场次独立 tx，失败隔离；log+continue，下一 tick 自愈 |
| 短课跨 tick 直接结束 | `ends_at <= now AND status IN (scheduled,in_progress)` 覆盖「从未 in_progress 就该结束」的场次（守卫允 `scheduled→completed`） |
| 时钟可控（测试） | `now` 作 repo 扫描查询参数；`EndSessionSystem` 自身时间无关（纯状态）。单测注入固定 now；live 验收用「过去 ends_at 数据」免等真实墙钟 |
| §22.6 冲突 | 自动**只产 `pending_no_show`**，绝不自动 `no_show`/扣课；staff `ConfirmNoShow` 仍为唯一扣课权威 |
| TX-3 跨批回改 | **零改动**（13e 已证）：`completed` 不可取消 + 仅 EndSession 产 `pending_no_show` → `Cancel` 的 `status='booked'` 查询永不触及。自动化复用同事务 |

## 7. auto-`in_progress` 透明性（brainstorm + code-review 收口）

`booking.WithinBookingWindow`（booking.go:196）在 `starts_at - BookAheadMinMinutes`（≤ `starts_at`）即关闭预约，故 `now ≥ starts_at`（in_progress 触发点）时**下单已被时间窗关闭**——in_progress 对 booking Create 无新增限制。

**但 code-review（F1）发现**：booking `Create`/`CreateByLearner` 与 waitlist `Join`/`Promote` 的守卫是 `status != 'scheduled'`，其中 **waitlist `Promote` 不过时间窗**（候补已承诺）——场次自动转 `in_progress` 后会**阻断「课已开始、有人腾位、把候补者转正」**这一真实工作流（Batch 15 前 session 永为 scheduled，promote 一直可用）。故把这 4 处守卫放宽为 **`status NOT IN ('scheduled','in_progress')`**：in_progress **视同 scheduled**，下单仍由**时间窗**、转正仍由**容量**把关，完整保留 Batch 15 前行为。learner 课程表 `List(From=now)` 本就按时间排除已开始场次，无需改。13c/13d/14a 全回归绿。

## 8. 验收方式（会话内，主线程自测）

可控时钟靠**数据**（过去 `ends_at`），非墙钟：
1. 起 `go run ./cmd/worker`（连 dev DB v12 + Redis）。
2. 造数据（psql / brand API）：
   - 场次 A：`starts_at` 过去、`ends_at` 未来 → 期望 sweep 后 `in_progress`。
   - 场次 B：`ends_at` 过去、`status='scheduled'`、含 1 条 `booked` 未签到 booking → 期望 sweep 后 `completed` + 该 booking `pending_no_show`、**不** `no_show`、hold 不动、`booked_count` 不变。
3. 等 1 个 sweep tick（或手动 enqueue 一次）。
4. psql 实查：A=`in_progress`；B=`completed` + booking=`pending_no_show`；`operation_logs` 有 `session_ended` actor_type=`system` actor_id NULL。
5. 幂等：再跑一轮，B 仍 `completed`（空操作，无新 audit 重复扣）。
6. 13e 手动入口仍可用（override）：手动对 A 点「结束场次」正常。

## 9. 测试（DB 单测，真实 Postgres `newMigratedTestDB`）
- `MarkSessionsInProgress`：scheduled+到点→in_progress；未到点/已 completed/cancelled 不动；幂等。
- `ListDueSessionIDs`：scheduled/in_progress 且 ends_at≤now 命中；future/completed/cancelled 不命中。
- `EndSessionSystem`：scheduled→completed+pending_no_show（actor=system，audit actor_id NULL）；in_progress→completed；completed→`SESSION_NOT_ENDABLE`（幂等）；不动 booked_count/hold；不产 no_show。
- `RunSweep`（app service，fake/real）：注入 now，断言 started/ended/skipped 计数 + 调用编排。
- **零回归复跑**：13e `TestEndSession_*` / `TestConfirmNoShow_*`、13c `TestBooking_*`/`TestSessionCancel_*`、13d `TestWaitlist_*` 全绿（actor 参数化后）。

## 10. 流程
brainstorm ✅ → grill ✅ → AskUserQuestion 拍板 ✅ → **本契约（会话内 approve）** → 测试场景 `batch-15-session-automation-tests.md` → 主线程逐 task TDD commit → `go build ./... && go test ./...` → `/code-review`（2 agent）→ 业务验收（会话内）→ 更新 PROGRESS + 三库 `.learnings` → 三仓库 push + backend `dev` FF `main`。
