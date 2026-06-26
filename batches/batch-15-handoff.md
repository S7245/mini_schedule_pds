# Batch 15 交接 prompt —— 场次状态自动化（asynq 定时任务框架接入）

> 新 session 直接读本文件按此执行。**这是「3 批后端/品牌端收尾」的第 1 批（顺序 15→16→17）。**
> ⚠️ **第一步不是 grill，是 brainstorm**：memory [[asynq-cron-adoption]] 的明确 How-to-apply 是「真正接入 asynq 时**先用 brainstorming skill 做针对性 brainstorm**，再定范围/首个自动项/worker 部署，不要照搬预案」。先 brainstorm → 再 grill 设计树 → 再 AskUserQuestion 拍板 → 再写契约。**拍板前不写代码、不写契约。**

## 先读（按序）
1. `pds/CLAUDE.md` —— 每轮流程。**注意**：CLAUDE.md 写「邮件停止点」被 classifier 拦，本项目已改**会话内 approve / 验收，不发邮件**（13a–14b 惯例）。读 PROGRESS 确认本批实际编号（约定 15，若有插批以 PROGRESS 为准）。
2. memory `asynq-cron-adoption`（**关键**：已查清的接入上下文 + §22.6 冲突 + 3 个待定决策点）。
3. `pds/PROGRESS.md` §5 **Batch 13e**（签到/履约/爽约：`EndSession`/`ConfirmNoShow` 是「手动版」状态机，自动化要覆盖/共存的就是它们）+ Batch 14a/14b（C 端，已闭环）。
4. `COURSE_BOOKING_BUSINESS_BLUEPRINT.md`（source of truth）：**§9.1 场次状态**（Draft→Scheduled→InProgress→Completed→Cancelled + 派生 Full/Closed）、**§22.6 爽约流程**（「第一版避免自动扣课造成争议，**默认需要员工确认**」← 硬约束）、§22.5 签到、§10 排课规则、§24.2 并发一致性。
5. `pds/GO_BACKEND_LANGUAGE_DESIGN.md`（后端分层铁律）。
6. 三库 `.learnings` 的 **13e 段**（`EndSession` 显式 action 决策、TX-3 零改动、锁序 session→booking→entitlement、零 migration 纪律）。
7. 核对代码：`internal/infrastructure/persistence/booking_repository.go`（`EndSession`/`ConfirmNoShow`/`Attend` 三事务）、`class_session_repository.go`（状态、Cancel）、`internal/infrastructure/cache/redis.go`（`NewRedisClient`，asynq 底座）、`cmd/`（现 3 个 API 服务，**无 worker**）、`go.mod`（**asynq 未装**）。

## 现状（已勘察，2026-06-26）
- **场次状态全手动**：创建直接落 `scheduled`（跳过 draft）；13e 加了**显式手动**「结束场次」`EndSession`（scheduled/in_progress→completed + 未签到 booked 批量→pending_no_show）+「确认爽约」`ConfirmNoShow`。**无 `scheduled→in_progress` 转换、无任何时间驱动**。派生 Full（booked≥capacity）/Closed（超预约截止）目前是**预约校验时即时算**，未落库也未显示。
- **asynq 未接**：`go.mod` 无 `hibiken/asynq`；backend 无任何后台任务/cron/worker（仅 api-brand:8081 / api-app:8082 / api-admin:8083）。**Redis 已接好**（`go-redis/v9`，RBAC 缓存在用）→ asynq 底座现成。
- dev DB 现 v12（14a/14b 零 migration）；本批若需 migration 从 **000013** 起。

## 核心难点（brainstorm + grill 必须透）
### A. §22.6 冲突 —— 哪些能自动、哪些不能
blueprint 硬约束「爽约默认需员工确认」。故：**自动 `scheduled→in_progress`（纯显示态，安全）**、**自动「结束场次」→completed + 产 pending_no_show（纯状态，可接受）** 是安全递增；**自动「确认爽约 no_show + 扣课」与 source-of-truth 冲突**，v1 不做（保留 13e 手动入口为权威）。13e 的手动 `EndSession`/`ConfirmNoShow` 在自动化后**保留为 override/兜底**。
### B. asynq 引入与 worker 部署
asynq = Redis-backed 队列 + 周期调度器（Scheduler）。**`asynq.Scheduler` 多副本会重复 enqueue → 必须单例**。部署两条路：①独立 `cmd/worker`（+ Railway 第 4 服务，Scheduler 单例最干净）②嵌入某个现有 API 进程的 goroutine（须选一端跑或加分布式锁）。任务处理器（Handler）复用既有 repo 事务（`EndSession` 等）。
### C. 幂等 / 并发 / 时钟
自动转换须**幂等**（重复 enqueue/重试安全）——复用 13e 事务的状态守卫（`EndSession` 仅 scheduled/in_progress→completed，重复调用第二次落 SESSION_NOT_ENDABLE/空操作）。periodic 扫描「到点未结束的场次」批量处理 vs 每场次定时任务（前者简单幂等，推荐）。测试需**可控时钟**（注入 now / 扫描 query 用参数化时间，避免依赖真实墙钟）。

## grill 设计树重点
- **范围切片**：①仅基础设施 + 最小自动项（asynq 接入 + 自动 scheduled→in_progress 显示态）②+ 自动结束场次（in_progress/scheduled 到 ends_at 后 → completed + pending_no_show，等价定时调 13e 的 EndSession）③自动确认爽约扣课（⚠️冲突，**不做**，留 FR）。
- **复用**：自动「结束场次」= 定时扫描 `ends_at < now AND status IN (scheduled,in_progress)` → 对每场次调既有 `EndSession`（actor=系统）。**actor 是 system**（audit `ActorSystem` 已存，非 brand_user，无 FK 问题）。
- **派生态**：Full/Closed 读时算（现状）vs 落库列（需 migration + 维护）——倾向**继续读时算**（零 migration），自动化只动主状态机。
- **跨批回改检查**：自动 EndSession 后 completed 场次不可取消（13e 已证 TX-3 闭合）；自动化复用同事务故 TX-3 仍零改动。复跑 13c–13e + 13d 全 DB 单测证零回归。

## 拆批 / 范围决策点（brainstorm + grill 后 AskUserQuestion 拍板，≤4 问）
1. **本批范围**：(a) 仅 asynq 接入 + 自动 scheduled→in_progress（最小，先立框架）/ (b) + 自动结束场次产 pending_no_show（推荐：一个真有业务价值的自动项）/ (c) 含自动爽约扣课（**不建议**，§22.6 冲突）。
2. **worker 部署**：(a) 独立 `cmd/worker` + Railway 第 4 服务（Scheduler 单例最干净，**推荐**）/ (b) 嵌入现有 API 进程 goroutine（省服务但须防多副本重复 enqueue）。
3. **自动化实现形态**：(a) periodic 扫描「到点未处理」批量（简单幂等，**推荐**）/ (b) 每场次注册定时任务（精确但任务量大、改约要重排）。
4. **migration**：派生态 Full/Closed 是否落库（落库需 000013 + 维护；倾向**读时算零 migration**）；asynq 是否需元数据表（asynq 自建 Redis 结构，通常无需 PG 表）。

## 流程铁律（13a–14b 惯例，严格遵守）
brainstorm（asynq）→ grill 设计树 → 写契约 `pds/batches/batch-15-*.md` → **会话内 approve**（不发邮件）→ 测试场景 `batch-15-*-tests.md` → **主线程逐 task TDD commit**（先红→绿→单 task commit；用户不 spawn 实现 subagent）→ `go build ./... && go test ./...`（+ 若动前端 `pnpm --filter @mini-schedule/brand build`）→ `/code-review`（2 并行 review agent）→ **业务验收**（会话内；可主线程自测：起 worker + 造到点场次看自动转换 + psql 实查；时钟可控）→ 更新 PROGRESS + 三库 `.learnings` → 三仓库分别 push → backend/web 的 `dev` FF `main`（`git push origin dev:main`，pds 直接 main）。
- 仓库：backend/web 在 `dev`，pds 在 `main`。DB 单测用真实 Postgres（`newMigratedTestDB`）。
- 这是**后端为主**批（worker/状态机），前端改动小（最多品牌端场次列表显示 in_progress/completed 态徽标）。

## 测试账号 / 数据（沿用 brand21）
- owner `18816820405 / admin123`。门店 讯美广场 loc1。
- 自动化验证须**可控时钟**：造 `ends_at` 已过去的 scheduled 场次（含 booked 未签到）→ 跑 worker 一轮 → 验证 → completed + booked→pending_no_show（psql 实查），且**不**自动 no_show（§22.6）。13e 手动入口仍可用（override）。

---
**第一步：先 brainstorm（asynq 范围/首个自动项/worker 部署，按 memory 指令），再 grill 上述难点 + 决策点，AskUserQuestion 拍板，再写契约。**
