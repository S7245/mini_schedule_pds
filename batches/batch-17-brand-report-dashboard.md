# Batch 17：品牌基础运营看板（§15 品牌看板 11 指标，纯读聚合）

> 「后端/品牌端收尾」第 3 批（15 场次自动化 → 16 订阅自动化 → **17 报表** → 18 通知）。
> 交接：[batch-17-handoff.md](batch-17-handoff.md)。grill 设计树 + AskUserQuestion 拍板（4 决策点**全取推荐项**，2026-06-27 会话内）后写就。契约 approve 前不写代码。

## 1. 背景

§15 基础报表「第一版只做基础运营看板」。当前**零报表**（无端点、无看板页），但数据全在库（13a–14b 落齐）。本批做**品牌看板**（brand 后台 :8081 / 前端 :3002 `@mini-schedule/brand`），一次 HTTP 出全部 11 指标的聚合端点 + 看板页。平台看板（admin）留 **17b**（其 ~5/9 指标已由 admin `GetPlatformSummary` 覆盖）。

## 2. 拍板结果（决策点，全取推荐项）

| # | 决策 | 结论 |
|---|---|---|
| 1 | 范围 | **仅品牌看板**；平台看板（套餐分布 + Location/席位/学员用量 + 本月收入）拆 17b |
| 2 | 指标集 | **§15 品牌看板全 11 项**（全部可算，单聚合端点，TopN 仅多几条 GROUP BY） |
| 3 | 上座率口径 | **到课数 / 总容量**（座位占用率；分母 = SUM(已完成场次 capacity)） |
| 4 | data_scope | **本批做店长本店 scope**（镜像 13c `scopeFilterIDs`：owner/admin/course_operator 全品牌，location_manager 仅 assigned locations） |

## 3. 已核实事实（grill 勘察，不必重查）

- **零 migration**：`report.view_basic` 权限码已在 `000003` seed（→ brand_owner / brand_admin / course_operator / location_manager），无新表。dev DB 保持 v12。
- **零新权限码 / 零新错误码**（纯读，无业务校验失败态）。
- 列已核实：`bookings.{booked_at,cancelled_at,status,brand_id,class_session_id}`、`class_sessions.{starts_at,capacity,status,brand_id,location_id,instructor_profile_id}`、`attendance_records.attended_at`、`entitlement_holds.{held_at,status}`、`entitlement_consumptions.consumed_at`、`waitlist_entries.status`。
- **聚合范式 = onboarding `GetCounts`**（`onboarding_repository.go:175`）：一次 repo 调用跑 N 条 `COUNT`/`GROUP BY`，逐个 `Scan` 进一个 struct。指标 1–4 可用 `COUNT(*) FILTER (WHERE status=…)` 一趟出。**不把分页 List 伪装统计**（CLAUDE.md 铁律）。
- booking 状态机字符串：`booked / cancelled / attended / pending_no_show / no_show`（§22.6：系统从不自动 no_show，`pending_no_show` 是真实可查积压态）。waitlist 活跃态：`waiting / eligible_to_promote`（`promoted/cancelled/skipped` 不计候补）。

## 4. 指标口径（source of truth，逐条精确定义，TDD 按此断言）

**时间窗**：preset `today` / `this_week` / `this_month`（默认）/ `custom`（须传 `from`、`to`）。窗口边界 UTC 计算（per-brand 时区留既有 FR）。`[from, to)` 左闭右开。

**A 组 — 周期活动（锚定场次 `starts_at` 落在窗口；场次 `status IN (scheduled,in_progress,completed)`，按 booking `status` 分类）**

| # | 指标 | 定义 |
|---|---|---|
| 1 | 预约数 | 窗口内场次上 `status <> 'cancelled'` 的 booking 数（该周期课程的活跃预约） |
| 2 | 到课数 | 窗口内场次上 `status = 'attended'` 的 booking 数 |
| 3 | 取消数 | 窗口内场次上 `status = 'cancelled'` 的 booking 数 |
| 4 | 爽约数 | 窗口内场次上 `status = 'no_show'` 的 booking 数（已确认爽约，区别于待处理） |
| 5 | 上座率 | 到课数(仅 `completed` 场次) / `SUM(capacity)`(仅 `completed` 场次)；分母 0 → 返 `0`。surfaced `total_capacity` + `attended_in_completed` 供透明对账 |
| 6 | 热门课程 TopN | 窗口内场次按 course 聚合，`COUNT(booking status<>'cancelled')` 排序取 Top 5：`[{course_id, title, booking_count}]`（course 软删仍按 title 显示） |
| 7 | Location 分布 | 窗口内 per location：`session_count`(场次数) + `booking_count`(status<>'cancelled')：`[{location_id, name, session_count, booking_count}]` |
| 8 | Instructor 场次 | 窗口内 per instructor：`session_count`(场次数)：`[{instructor_profile_id, name, session_count}]` |

**B 组 — 周期事件（锚定事件自身时间戳）**

| # | 指标 | 定义 |
|---|---|---|
| 9a | 权益锁定次数 | `COUNT(entitlement_holds WHERE held_at ∈ 窗口)` |
| 9b | 权益消耗次数 | `COUNT(entitlement_consumptions WHERE consumed_at ∈ 窗口)` |

**C 组 — 实时积压（忽略时间窗，读"当前待处理多少"）**

| # | 指标 | 定义 |
|---|---|---|
| 10 | 待处理爽约数 | `COUNT(bookings status='pending_no_show')`（live 全量积压） |
| 11 | 候补人数 | `COUNT(waitlist_entries status IN ('waiting','eligible_to_promote'))`（live） |

**data_scope（决策 4）**：`scopeLocationIDs` nil = 全品牌（owner/admin/course_operator）；非 nil = 店长 assigned locations。收紧方式：A 组天然有 `class_sessions.location_id`；B/C 组 location-less 表经 `→ bookings → class_sessions.location_id` join 收紧（holds/consumptions 有 `booking_id`；waitlist/pending_no_show 经 booking/session）。nil 时不加 location 过滤。空 scope（店长无分配）→ 全 0。

> **每指标按各自锚点过滤，严禁单一全局 `created_at`**（否则把"本周取消的旧预约"算进本周预约——subagent 标注的 #1 陷阱）。

## 5. 契约

**API 接口**

| 方法 | 路径 | 请求字段 | 响应字段 |
|---|---|---|---|
| GET | `/api/v1/brand/reports/overview` | `range`(today/this_week/this_month/custom，默认 this_month)、`from`、`to`(custom 必传)、`location_id`(可选，须在 scope 内) | `range{preset,from,to}`、`bookings_total`、`attended_total`、`cancelled_total`、`no_show_total`、`occupancy_rate`、`total_capacity`、`attended_in_completed`、`entitlement_locked_total`、`entitlement_consumed_total`、`pending_no_show_total`、`waitlist_total`、`popular_courses[]`、`location_distribution[]`、`instructor_sessions[]` |

- **权限门**：`report.view_basic`（middleware，镜像既有 permission 中间件）。
- **data_scope**：复用 13c 解析（actor 的 assigned location IDs；owner/admin/course_operator → nil=全品牌）。
- `location_id` 传入时须 ∈ scope（越权返 403 或按不存在处理，镜像 13c）。
- 纯读、无事务、无写、无 audit（查看类操作不写 OperationLog）。

**新增/改动（层次）**

| 层 | 文件 | 说明 |
|---|---|---|
| app（新） | `internal/application/report/service.go` | `Service.GetBrandOverview(ctx, ReportQuery) (*BrandOverview, error)`；解析 preset→[from,to)、组装 scope，调 repo |
| domain/types（新） | `internal/application/report/` 内 | `ReportQuery{BrandID, ScopeLocationIDs []int64, LocationID *int64, From, To time.Time}` + `BrandOverview` 结果 struct |
| infra（新） | `internal/infrastructure/persistence/report_repository.go` | `BrandOverviewCounts(ctx, query) (*report.BrandOverview, error)`：N 条聚合 SQL（镜像 onboarding `GetCounts`），scope/window/location 过滤 |
| interfaces（新） | `internal/interfaces/http/report_handler.go` | 解析 query 参数 + 校验 → 调 service → JSON |
| 路由/DI（改） | api-brand 路由注册 + Wire | 挂 `GET /reports/overview` 于 brand 路由（report.view_basic 门）；Wire 加 report service+repo+handler |

**前端页面模块**

| 页面/模块 | 类型 | 关键字段/操作 |
|---|---|---|
| `/reports` 运营看板页 | 页面 | 时间窗切换（今日/本周/本月/自定义区间）+ 可选门店筛选；标量指标卡片（预约/到课/取消/爽约/上座率/待处理爽约/候补/权益锁定/消耗）；热门课程 Top5 列表、Location 分布表、Instructor 场次表（简单条形/表格，**不引图表库**） |
| 导航入口 | 导航 | `report.view_basic` 门控的「报表」菜单项（镜像 NAV_HREF_PERMISSIONS 模式） |

**前端实现约束**：复用 brand 既有 Card / DataTable / Select / 时间选择组件；不引新 UI/图表库（§15 不做复杂图表，卡片数字 + 简单条形/表格即可）；按 brand app 布局惯例。

## 6. 失败模式 / 边界

| 风险 | 处理 |
|---|---|
| 上座率分母 0（窗口无 completed 场次） | 返 `occupancy_rate=0`，不除零 |
| 空窗口 / 店长无分配 location | 全 0（合法空态） |
| course/location 软删 | 热门课程/分布按 title/name 仍显示（不 inner-filter deleted_at，历史仍计） |
| N+1 / 分页伪装统计 | 专用聚合 SQL（COUNT/GROUP BY/FILTER），非 List 出来前端数 |
| 性能 | v1 规模 seqscan 可忽略；booked_at/starts_at/instructor calendar 索引多已就位；TopN GROUP BY 小集 |
| 时区 | 窗口边界 UTC（per-brand TZ 留既有 FR） |

## 7. 测试（DB 单测，真实 PG `newMigratedTestDB`）

- 造已知数据：多 location + 多 course + 多 instructor、跨窗口（窗内/窗外 starts_at）、各 booking 终态（booked/attended/cancelled/no_show/pending_no_show）、holds(held_at 窗内外)、consumptions、waitlist（active/非 active）。
- 断言：11 指标精确计数 + 上座率比率（含分母 0）+ A 组按 starts_at 锚定（窗外场次不计）+ B 组按 held_at/consumed_at 锚定 + C 组 live（忽略窗）+ 热门课程/分布/instructor 的 GROUP BY 排序与 TopN。
- **data_scope**：店长 scope（仅 assigned location 计入）vs nil（全品牌）；location-less 表经 booking→session join 收紧正确；空 scope → 全 0。
- 时间窗 preset 边界（事件刚好窗内/窗外 1 秒）。
- handler：参数校验（custom 缺 from/to）、location_id 越权。

## 8. 验收方式（会话内主线程自测 = 业务验收）

1. 起 api-brand（`CONFIG_PATH=configs/config-brand.yaml`，**先重启确保最新二进制**）+ 前端 brand。
2. 用 brand21 既有历史数据（13a–14b e2e 残留）或造已知数据：curl `GET /brand/reports/overview?range=this_month` → psql 手工对账每个指标（`SELECT COUNT … WHERE …` 比对）。
3. data_scope：owner(18816820405) 看全品牌；店长只读账号看本店收紧（数字 ≤ owner）。
4. 浏览器看 `/reports` 页：卡片数字、时间窗切换、门店筛选、热门课程/分布/instructor 列表渲染。
5. 若造数据，验毕清空（dev DB 复原 v12）。

## 9. 流程

grill ✅ → AskUserQuestion 拍板 ✅ → **本契约（会话内 approve）** → 测试场景 `batch-17-brand-report-dashboard-tests.md`（可选，口径表已含断言）→ 主线程逐 task TDD commit（聚合 SQL 用 DB 单测断言计数；先红→绿→单 task commit）→ `go build/test ./...` + `pnpm --filter @mini-schedule/brand build` → `/code-review`（high）→ 业务验收（会话内）→ 更新 PROGRESS + 三库 `.learnings` → 三仓库 push + backend/web `dev` FF `main`（pds 直接 main）。

- 预计**零 migration**、**零新权限码**、**零新错误码**。
- **后端聚合为主 + brand 前端一页**批；平台看板留 17b。

## 10. 留 FR

- 平台看板（17b：套餐分布 + Location/席位/学员用量 + 本月订单收入，补全 admin `GetPlatformSummary`）。
- per-brand 时区窗口边界（合并既有 TZ FR）。
- §15「第一版不做」清单：收入分析/支付对账/退款/教练绩效/转化率/留存/渠道/LTV/驾驶舱/导出。
- Instructor「自己场次数据」报表（21.2，instructor 未 seed report.view_basic，未来开放）；爽约数按确认时间精确锚点（现用父场次 starts_at 代理，因 bookings 无 no_show_at 列）；取消数按 cancel_source 拆分（learner/staff/session_cancelled）；导出中心。
