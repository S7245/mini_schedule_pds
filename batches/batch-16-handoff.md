# Batch 16 交接 prompt —— 订阅生命周期自动化（BrandSubscription 自动 grace/restricted，复用 Batch 15 asynq worker）

> 新 session 直接读本文件按此执行。**「后端/品牌端收尾」第 2 批（顺序 15 场次自动化 → 16 订阅自动化 → 17 报表 → 18 通知）。**
> ⚠️ **第一步是 grill，不是 brainstorm**：asynq 框架 Batch 15 已接入（memory [[asynq-cron-adoption]] 的 brainstorm 门是 Batch 15 的，已闭）。Batch 16 复用既有 worker，**直接走 CLAUDE.md §4 标准流程：grill 设计树 → AskUserQuestion 拍板 → 写契约**。拍板前不写代码、不写契约。

## 先读（按序）
1. `pds/CLAUDE.md` —— 每轮流程。**注意**：CLAUDE.md 写「邮件停止点」被 classifier 拦，本项目已改**会话内 approve / 验收，不发邮件**（13a–15 惯例）。读 PROGRESS 确认本批实际编号（约定 16）。
2. `pds/PROGRESS.md` **Batch 15**（场次状态自动化——本批是它的姊妹：同一 asynq worker、同样「自动状态机 + 复用既有事务 + 幂等 + 可控时钟」范式）+ §3 当前缺口（P0 支付异常补偿 / 商业化闭环）。
3. `COURSE_BOOKING_BUSINESS_BLUEPRINT.md`（source of truth）：**§4.5 套餐额度超限和到期**（到期前提醒/宽限期/受限状态，行 250-275）、**§19 已确认决策 行 1333-1335**（平台续期顺延规则、**默认宽限期 7 天可系统配置**、**到期→GracePeriod、宽限结束→Restricted**）、**生命周期 行 1389-1392**（Active→GracePeriod→Restricted→Expired）、**§23.1 行 1846**（宽限结束后禁止新增预约和资源）、§24.3 审计（订阅状态变更须 OperationLog）。
4. `pds/GO_BACKEND_LANGUAGE_DESIGN.md` §11（**明确把「BrandSubscription 自动进入 GracePeriod/Restricted/Expired 的定时任务」列为缺口**——本批正是填它）+ §2.1 cmd 约束 + §6 事务。
5. **Batch 15 契约 + 实现**（要镜像的范式）：`pds/batches/batch-15-session-automation.md`；代码 `cmd/worker/main.go`（asynq Server+Scheduler，加第 2 个 cron entry 即可）、`internal/application/sessionautomation/service.go`（RunSweep 范式：mark 批量 + 逐行系统事务 + 失败隔离 + 幂等）、`internal/interfaces/worker/sweep_handler.go`、`internal/audit`（ActorSystem）。
6. 三库 `.learnings` 的 **Batch 15 段**（system actor 复用既有事务 = actor 参数化；幂等三重；**引入「显示态」枚举要审计所有 `status='xxx'` 守卫**——本批最关键的复用教训）。
7. 核对代码：`internal/application/commercial/subscription_guard.go`（**`CheckAndCount` 的 `status='active' AND (grace_ends_at>now OR expires_at>now)` 门**）、`internal/infrastructure/persistence/commercial_repository.go`（`UpdateBrandSubscriptionStatus`:454、`GetPlatformSummary`:588 行 605/610 读 status 统计、Batch 3 回调建 sub :1122 不设 grace_ends_at、`computeSubscriptionExpiry`:1304）、`internal/application/commercial/service.go`（`UpdateBrandSubscriptionStatus`:208）、`internal/domain/commercial`（`BrandSubscriptionStatus*` 常量，核对 grace_period/expired 是否齐）。

## 现状（已勘察，2026-06-26）
- **schema 全就绪、零 migration（dev DB 保持 v12）**：`brand_subscriptions_status_valid` CHECK **已含** `('active','grace_period','restricted','frozen','expired','cancelled')`（镜像 Batch 15 的 in_progress 已在 CHECK）；**`idx_brand_subscriptions_status_expires_at(status, expires_at)` 已存**→到期扫描天然有索引（优于 Batch 15 无覆盖索引，无 F3 类 FR）；`grace_ends_at` 列已存（nullable）。
- **状态全靠人工 / 无任何时间驱动**：Batch 3 回调建 sub 只设 `status='active'`+`expires_at`，**`grace_ends_at` 恒 NULL**（续期/改状态路径也置 nil）。无 cron。故现在**没有真正的宽限期在生效**（grace_ends_at NULL → guard 用 expires_at 为硬界），且过期 sub 的 `status` 永远停在 `'active'` 不翻。
- **核心门是 read-time 时间比较 + status='active' 双条件**：`SubscriptionGuard.CheckAndCount`（Location/Staff/Learner 配额）= `status='active' AND (grace_ends_at>now OR (grace_ends_at IS NULL AND expires_at>now))`。**到期硬限制已经 read-time 生效**（即便 status 停在 active，时间条件 false 即拦）——但门**同时要求 status='active'**，这正是本批的中心张力（见难点 A）。
- **admin dashboard 已读 status 统计**：`GetPlatformSummary` 行 605「即将到期」= active 且 expires_at∈[now,+7d]；行 610「受限/冻结数」= status IN (restricted,frozen)。**现在 restricted 计数恒≈0（无人翻 status）**→自动化后这些指标自动准确（零 FE，镜像 Batch 15「进行中」徽标 13e 预置）。
- **frozen/cancelled 是人工 override**：`UpdateBrandSubscriptionStatus` 平台手动冻结/取消。cron 必须**绕开 frozen/cancelled**（人工权威，镜像 Batch 15 §22.6 不自动爽约扣课的边界）。

## 核心难点（grill 必须透）
### A. status 既是显示态又是门 —— 翻 status 会动 SubscriptionGuard（**Batch 15 F1 教训重演**）
`SubscriptionGuard` 要求 `status='active'`。一旦 cron 把到期 sub 翻成 `grace_period`，guard 的 `status='active'` 变 false → **宽限期内 brand 加资源被拦**，违反 blueprint §4.5「宽限期内继续可用」。**必须像 Batch 15 那样 `grep` 所有把 subscription.status 当门的点**，决定：把 guard 放宽为 `status IN ('active','grace_period')` 视同「可用」（grace 仍由 `grace_ends_at>now` 时间门把关），`restricted/expired/frozen/cancelled` 才拦。复跑 Batch 4/5/13a 配额测试证零回归。
### B. 状态机 + 时间触发 + grace_ends_at 从哪来
`active →(expires_at)→ grace_period →(grace_ends_at)→ restricted`。**`grace_ends_at` 现恒 NULL**→须决定：cron 翻 active→grace 时**计算 `grace_ends_at = expires_at + grace_days`**（grace_days 默认 7、系统配置 §1334）最干净（grace 窗从到期起算）；或建 sub 时即设。`restricted→expired` 是否再自动一跳？blueprint §1335 只说「宽限结束→Restricted」、§1389 又列 Expired——**grill 定 restricted 是否即终态**（数据留存、拒新增、仍可续期），expired/cancelled 留人工/后续。
### C. 复用 Batch 15 worker = 加第 2 个 periodic task
镜像 `sessionautomation`：新 `application/subscriptionlifecycle.Service.RunSweep(now)`（无 RBAC，系统执行）+ `interfaces/worker` 加 `subscription:sweep` handler + `cmd/worker` Scheduler 注册第 2 个 cron entry。sweep 两段：`active→grace_period`（WHERE active AND expires_at≤now，设 grace_ends_at）、`grace_period→restricted`（WHERE grace_period AND grace_ends_at≤now）。**审计**：订阅状态变更 §24.3 须 OperationLog→**逐 sub 系统事务**（actor=system，镜像 EndSessionSystem；sub 量小=每品牌一条，逐行无压力）而非纯批量。幂等同 Batch 15（WHERE status 守卫 + 已翻 sub 掉出扫描）。
### D. 续期/解冻要把 status 拉回 active + 清 grace_ends_at
平台续期（`UpdateBrandSubscription` 顺延 expires_at）/手动改状态后，若 sub 已 restricted/grace，须回 active 并清 `grace_ends_at`（现 renew 已置 grace_ends_at=nil，但**未把 status 从 restricted 拉回 active**——核对补齐，否则续了费仍显示受限）。

## grill 设计树重点
- **范围切片**：①仅 active→grace→restricted 自动化（最小闭环，受限即终态）②+ restricted→expired 自动再跳 ③+ 支付异常人工补偿（P0，平台从异常订单手动开通订阅——**独立关注点**，可留后续）④+ 到期前提醒通知（§4.5「到期前提醒」=站内/微信订阅消息——**留给 Batch 18 通知中心 / FR**，本批不做）。
- **门审计**：`grep -rn "status.*active" internal/**/*.go | grep -i subscription` 列全所有 subscription status 门（SubscriptionGuard 是已知一个；还有无别处？预约创建未必直接查 sub，但 onboarding/资源创建经 guard）。放宽策略统一：active+grace=可用，restricted+=拦。
- **复用 vs 重写**：RunSweep/handler/worker 注册全镜像 Batch 15；audit 用既有 `audit.Write(ActorSystem)`；状态常量复用 `commercial.BrandSubscriptionStatus*`（核对 grace_period/expired 常量是否齐，缺则补 Go 常量，DB CHECK 已含）。
- **跨批回改检查**：放宽 SubscriptionGuard 后复跑 Batch 4(Location)/5(Staff)/13a(Learner) 配额 + 过期绕过测试；admin summary 行 605/610 自动受益无需改但要核对语义（即将到期仍只数 active）。
- **失败模式**：cron 翻 status 与平台手动 `UpdateBrandSubscriptionStatus` 并发→逐 sub `SELECT FOR UPDATE`（同 EndSessionSystem 锁行）；frozen/cancelled 守卫（WHERE status IN active/grace_period，天然不碰人工态）；可控时钟靠扫描 query 参数化 now（同 Batch 15）。

## 拆批 / 范围决策点（grill 后 AskUserQuestion 拍板，≤4 问）
1. **本批范围**：(a) 仅订阅自动 active→grace→restricted（推荐：闭合 §11 缺口 + 商业化闭环最后一块自动化，受限即终态）/ (b) +restricted→expired 自动再跳 / (c) +支付异常人工补偿（P0，建议独立后续做）。
2. **SubscriptionGuard 放宽**：(a) 改为 `status IN (active,grace_period)` 视同可用、grace 由时间门把关（推荐，保留「宽限可用」语义 + 让 cron 翻 status 安全）/ (b) 其他门审计后再定。
3. **grace_ends_at 来源**：(a) cron 翻 active→grace 时算 `expires_at + grace_days`（推荐，grace 窗从到期起算，零 migration）/ (b) 建 sub 时即设（须改 Batch 3 回调 + 续期）。grace_days 配置：复用/新增 `worker.grace_days` 默认 7 vs 平台级 system config。
4. **前端**：(a) 零 FE——admin dashboard 已读 status 自动准确，brand 受限 banner 留 FR（推荐，后端为主）/ (b) 加 brand 后台「受限状态」banner（§4.5，小 FE）。

## 流程铁律（13a–15 惯例，严格遵守）
grill 设计树 → AskUserQuestion 拍板 → 写契约 `pds/batches/batch-16-*.md` → **会话内 approve**（不发邮件）→ 测试场景 `batch-16-*-tests.md` → **主线程逐 task TDD commit**（先红→绿→单 task commit；用户不 spawn 实现 subagent）→ `go build ./... && go test ./...` → `/code-review`（high，多 finder + 验证）→ **业务验收**（会话内主线程自测：起 worker + 造「过去 expires_at/grace_ends_at」订阅看自动转换 + psql 实查；时钟靠数据免等墙钟）→ 更新 PROGRESS + 三库 `.learnings` → 三仓库分别 push → backend/web 的 `dev` FF `main`（`git push origin dev:main`，pds 直接 main）。
- 仓库：backend/web 在 `dev`，pds 在 `main`。DB 单测用真实 Postgres（`newMigratedTestDB`）。
- **后端为主**批（worker/状态机/guard 放宽），前端预计零改动（admin summary 自动准确）。
- 预计**零 migration**（status CHECK + 索引 + grace_ends_at 列全就绪）、零新权限码。

## 测试账号 / 数据（沿用 brand21）
- owner `18816820405 / admin123`。门店 讯美广场 loc1。
- 自动化验证须**可控时钟**：造 `expires_at` 已过去的 active 订阅（grace_ends_at NULL 或已过去）→ 跑 worker 一轮 → 验证 active→grace_period（设 grace_ends_at）、grace_ends_at 已过的→restricted（psql 实查 + audit actor=system）；frozen/cancelled 订阅**不动**（人工 override）；续期把 restricted 拉回 active+清 grace_ends_at。SubscriptionGuard 放宽后：grace 期内仍可加 Location（时间门未过），restricted 后拒。

---
**第一步：先 grill 上述难点 + 决策点（重点透难点 A 的 status 门审计——Batch 15 F1 重演），AskUserQuestion 拍板，再写契约。复用 Batch 15 worker 范式，预计零 migration / 零前端。**
