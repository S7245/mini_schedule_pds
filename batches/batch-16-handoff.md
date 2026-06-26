# Batch 16 交接 prompt —— 基础运营看板 / 报表（§15）

> 新 session 直接读本文件按此执行。**「3 批收尾」第 2 批（顺序 15→16→17，本批可独立于 15）。**
> **第一步只做 grill 设计树 + 拆批/范围决策，不写代码。** 这是**纯读聚合**批（数据全在库），风险低但**范围/指标定义**要 grill 清。

## 先读（按序）
1. `pds/CLAUDE.md`（每轮流程；会话内 approve/验收不发邮件，13a–14b 惯例）。读 `pds/PROGRESS.md` 确认本批编号 + 全部已完成域（报表的数据源）。
2. `COURSE_BOOKING_BUSINESS_BLUEPRINT.md`：**§15 基础报表**（**品牌看板** 11 指标 + **平台看板** 9 指标 + 「第一版不做」清单——别越界做收入分析/对账/绩效/留存/LTV/导出）、**§14.2 品牌后台菜单「报表」**、**§14.1 平台后台菜单**、§3 角色（谁能看报表）。
3. 三库 `.learnings`（onboarding 的 `CountsByStep` 一次性多表 COUNT 范式 = 报表聚合可照搬；13b–13e 的 booking/attendance/entitlement/waitlist 落库语义）。
4. 核对数据源代码：`internal/infrastructure/persistence/` 下 `booking_repository.go`（bookings/attendance 终态）、`class_session_repository.go`（场次/容量/booked_count）、`entitlement_repository.go`（hold/consume）、`waitlist_repository.go`、`commercial/`（平台看板：brand/subscription/order/quota）。`internal/application/onboarding/service.go`（`CountsByStep` 多表 COUNT 范式）。
5. 核对权限：`grep -n "report" migrations/000003_*.up.sql` 看有无 `report.*` 权限码（大概率无 → 决定新增 or 复用）。

## §15 指标清单（source of truth，grill 时逐条核对可算性）
**品牌看板**：预约数、到课数、取消数、爽约数、上座率、热门课程、Location 场次/预约分布、Instructor 授课场次数、权益锁定/消耗次数、待处理爽约数、候补人数。
**平台看板**：品牌总数、活跃品牌数、付费品牌数、即将到期品牌数、本月订单收入、套餐分布、Location 用量、员工席位用量、学员用量。
**第一版不做**（别做）：收入分析、支付对账、退款报表、教练绩效薪酬、课程转化率、学员留存、渠道来源、LTV、经营驾驶舱、导出中心。

## 现状（待核实）
- **零报表**：无报表端点、无看板页。数据全在库（13a–14b 已落齐）。
- **两端**：品牌看板属 brand 后台（:8081 / 前端 :3002 `@mini-schedule/brand`）；平台看板属 admin 后台（:8083 / 前端 admin）。
- 大概率**零 migration**（纯读）；除非新增 `report.view` 权限码（则从 000013，镜像 13a/13b 的权限 seed+backfill）。
- brand21 有大量历史数据（13a–14b e2e 残留）可直接出数。

## 核心难点（grill 必须透）
### A. 聚合效率（不许 N+1、不许分页伪装统计）
每个看板一次 HTTP 应一次性出全部指标：**多个 COUNT/GROUP BY 聚合查询**（镜像 onboarding `CountsByStep`），按 time-range + brand（+ 可选 location）过滤。热门课程/Location 分布/Instructor 场次 = GROUP BY 取 TopN。**CLAUDE.md 铁律：不把分页数据伪装为全局统计**——统计走专用聚合 SQL，不是 List 出来前端数。
### B. 范围切片（品牌 vs 平台双端）
品牌看板（brand）与平台看板（admin）是**两端两套数据源/权限**。grill 决定：本批做单端还是双端？倾向**先做品牌看板**（业务方最常看 + 数据源就在 13x 域），平台看板可同批或拆 16b。
### C. RBAC + data_scope
谁能看报表？新增 `report.view` 权限码（migration + 角色 seed，镜像 13a）vs 复用既有（如 owner/admin 才有的某码）。**data_scope**：店长/前台只能看本店指标（assigned_locations 过滤），owner/admin 看全品牌——镜像 13c booking 的 `scopeFilterIDs`。
### D. 指标精确定义（grill 拍口径）
「上座率」= 到课数/总容量 还是 到课/预约？「热门课程」按预约数还是到课数？时间范围默认（今日/本周/本月/自定义）？这些口径 grill 时定清写进契约，免返工。

## 拆批 / 范围决策点（grill 后 AskUserQuestion 拍板，≤4 问）
1. **范围**：(a) 仅品牌看板（**推荐**，业务价值高 + 数据源就位）/ (b) 品牌 + 平台双看板一批 / (c) 拆 16a 品牌 + 16b 平台。
2. **指标集**：(a) §15 品牌看板 11 项全量 / (b) 先做高频子集（预约/到课/取消/爽约/上座率/待处理爽约/候补，热门课程+分布留 16b）。grill 给推荐。
3. **权限**：(a) 新增 `report.view` 权限码（migration 000013 + 角色 seed/backfill，镜像 13a，**推荐**——报表是独立菜单应有独立门）/ (b) 复用 owner/admin 既有码（零 migration）。+ data_scope（店长本店）是否本批。
4. **时间范围 + 过滤**：默认时间窗（今日/本周/本月/自定义区间）+ 是否 per-location 过滤（品牌看板）。

## 流程铁律（13a–14b 惯例）
grill → 契约 `pds/batches/batch-16-*.md` → **会话内 approve** → 测试场景 → **主线程逐 task TDD commit**（聚合 SQL 用 DB 单测断言计数；先红→绿→单 task commit）→ `go build/test ./...` + `pnpm --filter @mini-schedule/brand build`（+admin 若做平台端）→ `/code-review`（2 并行）→ **业务验收**（会话内；造已知数据→断言看板数字；可主线程自测 curl + psql 对账 + 浏览器看页）→ 更新 PROGRESS + 三库 `.learnings` → 三仓库 push → backend/web `dev` FF `main`。
- DB 单测 `newMigratedTestDB`；聚合正确性靠「造 N 条已知态数据 → 断言看板返回的计数/比率」。
- 前端：brand `/reports`（或 `/dashboard` 概览）页，复用既有 DataTable/Card/筛选；不引新图表库前先 grill（卡片数字 + 简单条形即可，§15 不做复杂图表）。

## 测试账号（brand21）
owner `18816820405 / admin123`（全品牌看板）；若做 data_scope，用店长/前台只读账号验本店收紧（见 [[brand21-test-accounts]]）。门店 讯美广场 loc1。

---
**第一步只做 grill**：逐条核对 §15 指标可算性 + 聚合范式 + 范围/权限/口径决策点，AskUserQuestion 拍板，再写契约。
