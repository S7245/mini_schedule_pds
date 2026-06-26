# Batch 18 交接 prompt —— 员工/品牌后台站内通知中心（§20.13）

> 新 session 直接读本文件按此执行。**「后端/品牌端收尾」第 4 批（顺序 15 场次自动化 → 16 订阅自动化 → 17 报表 → 18 通知）。**
> **第一步只做 grill 设计树 + 拆批/范围决策，不写代码。** 这是**最大 greenfield**（新域 + migration + 跨域事件集成），grill 要透数据模型/recipient/集成点三难点。
> ⚠️ **学员端微信订阅消息（§7.5）不在本批**——卡真实微信（per-brand 小程序模板审核），留 FR。本批只做**后台站内通知**（品牌/员工后台）。

## 先读（按序）
1. `pds/CLAUDE.md`（每轮流程；会话内 approve/验收不发邮件）。读 `pds/PROGRESS.md` 确认编号 + 全部已完成域（通知的事件源）。
2. `COURSE_BOOKING_BUSINESS_BLUEPRINT.md`：**§20.13 通知消息**（后台通知对象 5 类角色 + 事件 7 项 + 失败处理）、**§7.5 微信订阅消息**（学员端，本批**不做**，区分清）、**§14.2 品牌菜单「通知消息」** + **§14.3 员工视图「消息通知」**、§3 角色（recipient targeting）、§24.1 权限数据隔离。
3. 三库 `.learnings`（13c–15）：booking/waitlist/entitlement/session 各事务的**事件产出点**（哪里发生「新预约/取消/候补变化/待爽约/场次取消」）；`internal/audit`（OperationLog 是审计，**非用户通知**，对比但不复用）。
4. 核对代码：`internal/audit/audit.go`（audit.Write 范式，通知可镜像其「事务内写一条」结构）；各 repo 的写事务（`booking_repository.go` Create/Cancel/EndSession、`waitlist_repository.go` Join/Promote/Cancel、`class_session_repository.go` Cancel、`commercial/` 订阅/额度）= 通知 emit 的注入点；`commercial.SubscriptionGuard`（额度即将超限事件源）。
5. 核对 schema：`grep -nE "CREATE TABLE.*(notification|message|notice)" migrations/*.up.sql` —— **大概率无**（greenfield，须新表 000013+）。`grep -n "'notification" migrations/000003_*.up.sql` 看有无预留权限码。

## §20.13 规格（source of truth）
**后台站内通知对象（recipient 角色）**：品牌负责人、品牌管理员、店长、前台、Instructor。
**后台站内通知事件（v1 候选）**：新预约、取消预约、候补变化、待确认爽约、场次取消、支付/订阅异常、套餐额度即将超限。
**失败处理**：发送失败不影响主流程、失败记日志；「消息/通知记录」可展示。

## 现状（待核实）
- **零通知系统**：无 notification/message 表、无消息中心页、无未读徽标。OperationLog 存在但是审计流水（actor→target→before/after），**不是面向用户的收件箱**。
- **事件源散在各域**：新预约/取消（booking）、候补变化（waitlist）、待爽约（EndSession）、场次取消（class_session.Cancel 级联）、额度预警（commercial.SubscriptionGuard）、订阅异常（commercial）。
- dev DB v12（截至 Batch 15）→ 本批**必有 migration**（notification 表 + 权限码），编号取**下一个可用值**（16 订阅自动化预计零 migration；17 报表若加 `report.view` 则占 000013 → 本批顺延，以实现时 `ls migrations/` 为准）。

## 核心难点（grill 必须透）
### A. 数据模型 —— 持久化收件箱 vs 读时聚合
§20.13「消息记录可展示」+ 未读态 + 多角色定向 → 倾向**持久化 `notifications` 表**（id/brand_id/recipient 维度/event_type/payload JSONB/read_at/created_at + CHECK event_type + 索引）。区别于 onboarding 的纯读 COUNT 聚合（通知有未读状态须落库）。grill 定表结构 + recipient 维度（见 B）。
### B. recipient targeting（最难）
事件→谁收？三种模型：①**per-role 广播**（一条通知 targeting 角色集，读时按当前用户角色 + data_scope 过滤可见）②**per-user 落行**（emit 时 fan-out 给每个匹配用户写一行，未读态天然 per-user，但写放大）③混合。**data_scope**：店长/前台只收**本店**场次的事件（assigned_locations），owner/admin 收全品牌——须在 targeting 时按 location 过滤（事件带 location_id）。grill 定模型（per-user fan-out 的未读/已读最简单，但要控写放大）。
### C. 事件产出集成（跨域触点多）
在既有 booking/waitlist/session/commercial **写事务内**还是**事务后**emit 通知？事务内（同 tx 写 notification，强一致但耦合 + 失败回滚业务）vs 事务后/异步（§20.13「失败不影响主流程」倾向**异步/best-effort**）。**Batch 15 已接 asynq（worker 框架就位：`cmd/worker` + Scheduler/Server + `internal/interfaces/worker`）**——通知 emit 可走 asynq 队列（解耦最干净），或事务后 best-effort go func + 失败记日志。grill 定：同步事务内 vs 异步走 asynq。
### D. 读模型 + 未读徽标
列表（分页 + 已读/未读筛选）、未读数（徽标，复用 RBAC 那种轻量 count 端点）、标记已读/全部已读、data_scope 过滤。前端：品牌后台「通知消息」页 + 顶栏未读徽标（员工视图「消息通知」同源按权限/scope 过滤）。

## 拆批 / 范围决策点（grill 后 AskUserQuestion 拍板，≤4 问）
1. **数据模型**：(a) 持久化 `notifications` 表 + per-user fan-out 落行（未读态最简，**推荐**）/ (b) per-role 广播行 + 读时按角色/scope 过滤（写少读复杂）/ (c) 纯读时聚合（无未读态，不满足 §20.13）。
2. **事件范围 v1**：(a) 预约域事件（新预约/取消/候补变化/待爽约/场次取消，**推荐**——业务高频且事件源就在 13x）/ (b) + 商业化事件（订阅异常/额度预警，跨 commercial 域）/ (c) §20.13 全量。
3. **集成方式**：(a) 业务事务后 best-effort 异步 emit（§20.13「失败不影响主流程」，**推荐**）/ (b) 事务内同步写（强一致但耦合）/ (c) 走 Batch 15 的 asynq 队列（15 已接，worker 框架就位，最解耦）。
4. **migration + 权限**：`notifications` 表（000013，event_type/recipient CHECK + 索引）+ `notification.view`/`mark_read` 权限码（角色 seed/backfill 镜像 13a）。data_scope（店长本店）是否本批。

## 流程铁律（13a–14b 惯例）
grill → 契约 `pds/batches/batch-18-*.md` → **会话内 approve** → 测试场景 → **主线程逐 task TDD commit**（先红→绿→单 task commit；用户不 spawn 实现 subagent）→ `go build/test ./...` + `pnpm --filter @mini-schedule/brand build` → `/code-review`（2 并行）→ **业务验收**（会话内；触发业务事件→断言对应角色/scope 用户收到通知 + 未读徽标 + 标记已读，psql 实查 notifications；越权角色/越店不可见）→ 更新 PROGRESS + 三库 `.learnings` → 三仓库 push → backend/web `dev` FF `main`。
- DB 单测 `newMigratedTestDB`；本批有 migration 000013（确认从空库顺序 up + 补 down）。
- **跨域回改纪律**：在 booking/waitlist/session/commercial 写路径注入 emit 后，**复跑 13c–15 全 DB 单测证零回归**（emit best-effort 不得改变业务事务结果）。

## 测试账号（brand21，多角色）
owner `18816820405 / admin123`（收全品牌）；店长/前台/instructor 只读账号验 **data_scope 收紧**（只收本店事件）+ 角色定向（见 [[brand21-test-accounts]]）。门店 讯美广场 loc1。

---
**第一步只做 grill**：透数据模型/recipient targeting/事件集成三难点 + migration，AskUserQuestion 拍板，再写契约。学员微信订阅（§7.5）留 FR。
