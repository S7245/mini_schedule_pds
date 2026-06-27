# Batch 17b 交接 prompt —— 平台运营看板（§15 平台看板，补全 admin dashboard）

> 新 session 直接读本文件按此执行。**「后端/品牌端收尾」延伸批：15 场次自动化 → 16 订阅自动化 → 17a 品牌看板 → **17b 平台看板** → 18 通知。**
> **第一步只做 grill 设计树 + 范围/口径决策，不写代码。** 这是 **纯读聚合 + 扩现有端点** 批（比 17a 更轻），风险低但**用量口径 / grace 计数 / 结构**要 grill 清。
> 上一批 17a（品牌看板）已完成并合 main，本批是其平台端对偶。**会话内 approve / 验收，不发邮件**（13a–17a 惯例）。

## 先读（按序）
1. `pds/CLAUDE.md`（每轮流程）+ `pds/PROGRESS.md` 的 **Batch 17 段**（17a 品牌看板，本批要镜像的范式 + FR 里写明的「17b 平台看板」缺口）。
2. **17a 实现 = 要镜像的聚合范式**（已合 main）：
   - `backend/internal/infrastructure/persistence/report_repository.go`（`BrandOverviewCounts`：N 条聚合 SQL，`COUNT`/`GROUP BY`/`FILTER` 一次出全指标；GROUP BY 取分布/TopN 的写法）。
   - `backend/internal/application/report/service.go`（`resolveWindow` 月/周/日 UTC 边界——本批「本月收入」复用月边界逻辑）。
   - `backend/.learnings/LEARNINGS.md` 的 Batch 17 段（聚合范式 / 口径分组 / GORM Raw Scan 进 slice）。
3. **要扩的现有代码**（本批核心——**扩 GetPlatformSummary，不新建域**）：
   - `backend/internal/domain/commercial/repository.go:169` `PlatformSummary` struct（加新字段）。
   - `backend/internal/infrastructure/persistence/commercial_repository.go:703` `GetPlatformSummary`（10 个 COUNT 已在；加 5 个新聚合查询）。
   - `backend/internal/interfaces/admin/commercial_handler.go:144`（`GET /platform/summary` 已注册，返回 struct，**无需新端点**）。
   - `web/apps/admin/app/(protected)/dashboard/page.tsx`（已渲染 `summary?.brand_total` 等卡片，加新卡片）+ `web/packages/api/src/admin.ts:114` `usePlatformSummary` + `web/packages/types`（PlatformSummary 类型加字段）。
4. `COURSE_BOOKING_BUSINESS_BLUEPRINT.md` **§15 平台看板 9 指标** + **「第一版不做」清单**（别越界做收入分析/对账/退款/LTV/驾驶舱）。

## §15 平台看板 9 指标（source of truth）+ 现状核对
`GetPlatformSummary` 现返回 10 字段（`commercial_repository.go:703-749`），覆盖情况：

| §15 指标 | 现状 | 来源 |
|---|---|---|
| 品牌总数 | ✅ `BrandTotal` | `COUNT(brands)` |
| 活跃品牌数 | ✅ `ActiveBrandTotal` | `brands status='active'` |
| 付费品牌数 | ⚠️ `ActiveSubscriptionTotal` | `brand_subscriptions status='active'`（**Batch 16 后 grace_period 不计**——见决策 3）|
| 即将到期品牌数 | ✅ `ExpiringIn7DaysTotal` | active 且 expires_at∈[now,+7d] |
| 本月订单收入 | ❌ **缺**（仅 `TodayPaidAmount` 今日）| 须 `SUM(amount) paid orders WHERE paid_at>=月初` |
| 套餐分布 | ❌ **缺** | `GROUP BY plan_id`（active(+grace?) 订阅）JOIN saas_plans 取名 → 列表 |
| Location 用量 | ❌ **缺** | `COUNT(locations WHERE deleted_at IS NULL)` 平台总量 |
| 员工席位用量 | ❌ **缺** | `COUNT(brand_users WHERE deleted_at IS NULL)` |
| 学员用量 | ❌ **缺** | `COUNT(brand_learner_profiles WHERE deleted_at IS NULL)` |

> 现 struct 另有 `PendingBrandTotal / RestrictedOrFrozenTotal / TodayOrderTotal / TodayPaidAmount / ExceptionOrderTotal / FailedCallbackTotal`（运营健康指标，超出 §15 但已在 dashboard，**不动**）。本批 = **补 5 个缺口指标**（本月收入 + 套餐分布 + 3 个用量）。

## 现状（已核实，不必重查）
- **大概率零 migration**：所有指标读现有表（saas_plan_orders / brand_subscriptions / saas_plans / locations / brand_users / brand_learner_profiles），dev DB 保持 v12。
- **零新权限码**：admin API 由 `middleware.JWTAuth(h.jwtSvc, "admin")` 整体门控（`admin/handler.go:81`），**无 brand 端那种 per-permission RBAC**——平台管理员是单一特权角色。`/platform/summary` 已在 admin auth 组内。
- **零新端点 / 零新错误码**：扩 `GetPlatformSummary` 既有 struct + 既有路由 + 既有 dashboard 页（纯读）。
- **两端**：本批是 **admin 端**（后端 api-admin :8083 / 前端 admin app）。17a 是 brand 端，互不影响。
- 结构对偶提醒：17a 为品牌看板新建了 `domain/report`；本批**默认扩 `commercial.GetPlatformSummary`**（dashboard 已 wired，churn 最小），是否改走 report 域见决策 4。

## 核心难点（grill 必须透）
### A. 用量口径 = 平台总量 vs 用量÷配额利用率
「Location/员工席位/学员 用量」两种读法：(a) **平台总资源量**（`COUNT(locations/brand_users/brand_learner_profiles)` 全平台，"平台规模"）；(b) **利用率**（`SUM(已用) / SUM(订阅 max_*)`，"配额吃了多少"）。(a) 简单直观、(b) 更运营但要 join 订阅 max。grill 拍：倾向 **(a) 总量**，利用率留 FR/stretch。
### B. 套餐分布口径
按 **active(+grace?) 订阅 GROUP BY plan_id**（当前付费分布，推荐）vs 按所有品牌（含 pending/expired）。输出 `[{plan_id, name, brand_count}]`。软删 plan 仍按名显示。
### C. grace_period 计数（Batch 16 遗留 FR）
Batch 16 后 `grace_period` 订阅**不入任何 summary 桶**（`ActiveSubscriptionTotal` 仅 active、`RestrictedOrFrozenTotal` 仅 restricted/frozen）。本批顺手修否？grace 仍在运营窗（17a guard 视同可用）。grill 拍：付费品牌数 / 套餐分布是否含 grace（倾向**含 active+grace**，与 17a guard 口径一致）。
### D. 时间窗口
平台看板是**快照**（非 17a 那种可选区间）。「本月收入」= 当前自然月（月初→now，按 `paid_at`，复用 17a `resolveWindow` 的月边界 / 现 `TodayPaidAmount` 的 day 边界换月）。其余用量/分布 = 实时快照。UTC 边界（per-brand TZ 不适用，平台级）。

## 决策点（grill 后 AskUserQuestion 拍板，≤4 问）
1. **范围**：(a) 补全 §15 平台看板 5 缺口（本月收入 + 套餐分布 + 3 用量，**推荐**）/ (b) 子集（先标量、套餐分布留后）。
2. **用量口径**：(a) 平台总量 count（**推荐**）/ (b) 用量÷配额利用率。
3. **grace 计数**：(a) 付费品牌数 + 套餐分布含 **active+grace**（**推荐**，对齐 17a guard）/ (b) 维持仅 active（grace 留 FR）。
4. **结构**：(a) 扩 `commercial.GetPlatformSummary`（**推荐**，dashboard 已 wired、churn 最小）/ (b) 新建 `report.PlatformOverview`（与 17a 域对称，但要新端点 + 改 dashboard 数据源）。

## 流程铁律（13a–17a 惯例）
grill → 契约 `pds/batches/batch-17b-platform-dashboard.md` → **会话内 approve** → 测试场景（口径表即断言）→ **主线程逐 task TDD commit**（聚合 SQL 用 DB 单测：造已知 brands/subs/plans/locations/staff/learners → 断言 5 新指标 + 套餐分布 GROUP BY；先红→绿→单 task commit）→ `go build/test ./...`（零回归，复跑 commercial / GetPlatformSummary 既有测试）+ `pnpm --filter @mini-schedule/admin build` → `/code-review`（high）→ **业务验收**（会话内：platform admin curl `GET /api/v1/admin/platform/summary` + psql 对账每个新指标 + 浏览器看 admin dashboard 新卡片）→ 更新 PROGRESS + 三库 `.learnings` → 三仓库 push + backend/web `dev` FF `main`（pds 直接 main）。
- **默认分支 push（backend/web `dev:main` FF、pds `main`）须先 AskUserQuestion 取用户显式授权**（auto-mode classifier 拦默认分支直推，13a–17a 实测；`dev` 非默认可直推）。
- DB 单测 `newMigratedTestDB`；admin 端测试账号 = 平台管理员（查 admin 登录 seed / `cmd/api-admin`；api-admin 须 `CONFIG_PATH=configs/config-admin.yaml`，测试用 `PORT=18083` 避开可能 stale 的 :8083）。
- 前端：扩 `apps/admin/.../dashboard/page.tsx` 既有卡片网格 + `admin.ts` 类型；复用既有卡片组件，不引图表库（§15 不做复杂图表，套餐分布用简单表/条）。

## 留 FR（写入三库 `.learnings/FEATURE_REQUESTS.md`）
- 用量÷配额利用率（若决策 2 选总量）；本月收入趋势/环比；§15「第一版不做」（收入分析/对账/退款/绩效/转化/留存/渠道/LTV/驾驶舱/导出）。
- 平台看板可选时间区间（现快照 + 本月固定）。

---
**第一步只做 grill**：核对 §15 平台看板缺口 5 指标可算性 + 4 决策点（范围/用量口径/grace/结构），AskUserQuestion 拍板，再写契约 `pds/batches/batch-17b-platform-dashboard.md`。grill/拍板/契约完成前不写实现代码。
