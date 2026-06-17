# Batch 12b：RecurringSchedule 循环排课

状态：契约已 approve（会话内确认，2026-06-16；生成上限拍板 26 周 + 200 节）

## 0. 范围与决策（grill 定论，12a 已预定 + 本批细化）

闭环主题：**品牌按「每周重复 + 选周几 + 起止」一次性批量生成 N 节 class_session（逐节冲突检查、冲突跳过返清单）→ 循环排课列表/详情可见已生成场次 → 非级联取消（停模板，不动已生成场次）**。

> Batch 12 拆 12a（资源管理，已完成）+ **12b（本批，循环排课）**。复用 12a 的资源选择器、`SESSION_RESOURCE_CONFLICT`、容量优先级。表 `recurring_schedules` + `recurring_schedule_weekdays` 已在 000003 建好，**本批不动表结构**。

| # | 决策 | 结论 |
|---|---|---|
| D1 | 部分失败策略 | 外层单 tx：先插 `recurring_schedules` + `recurring_schedule_weekdays`，再逐 occurrence 用 **GORM 嵌套 tx（SAVEPOINT）** 插 class_session；撞 EXCLUDE(23P01) → 回滚该 savepoint、记入 skipped、继续；非冲突错误整批 abort。批级不变量（course published / 门店 active / 课程在门店可用 / 教练可排课 / 资源 active+同门店）**循环前查一次**，循环内只可能撞 EXCLUDE |
| D2 | 0 成功生成 | 全部 occurrence 冲突 → **整批 abort、回滚 recurring_schedules 插入**、返 409 `RECURRING_ALL_CONFLICT` + skipped 清单（不落空壳） |
| D3 | 取消语义 | recurring cancel 仅 `status active→cancelled`（停模板、解除门店 guard），**不级联取消已生成场次**（blueprint §10.2「不做整批取消」）。单场次仍在 /schedule 逐个取消 |
| D4 | 权限 | **复用 `session.view`（list/detail）/ `session.create`（生成）/ `session.cancel`（cancel）**，不新增 recurring.* 权限码 → **本批无权限 migration** |
| D5 | 数据权限 | 接 data_scope：生成/列表/详情/取消按 recurring.location_id 走 `scopeFilterIDs`/`guardLocationInScope`（镜像 classsession） |
| D6 | 周几约定 | `recurring_schedule_weekdays.weekday` = Go `time.Weekday`（**0=周日 … 6=周六**）。前端展示「周一…周日」并做映射 |
| D7 | 结束条件 | `end_date` 与 `repeat_weeks` **二选一（XOR，必填其一）**。`repeat_weeks` 给定时 effective_end = start_date + repeat_weeks*7 − 1 天 |
| D8 | 生成上限 | 最长 **26 周** 跨度；生成 occurrence 硬上限 **200 节**；超限 → 400 `SESSION_TIME_INVALID`（含提示）。空 weekday / start_date<今天 / 区间内无匹配日 → 400 |
| D9 | 容量 | recurring.capacity 入参未给（<=0）时优先级 **绑定资源容量 > course.default_capacity**；存进 recurring_schedules.capacity 并传给每节场次 |
| D10 | 时区 | DATE+TIME→timestamptz 按 **Asia/Shanghai** 生成（v1 中国单时区，与单场次「浏览器本地→UTC」一致）。per-brand TZ 转 FR |
| D11 | 门店删除 guard | `CountActiveReferences` 追加 active `recurring_schedules`（status='active'）。落实 Batch 9/11 挂的 FR recurring 部分（12a 已纳资源，本批补 recurring） |
| D12 | 额度门 | 同 11/12a，不挂 SubscriptionGuard |

## 1. 契约

### 1.1 数据库

无 migration（表已建、权限复用 session.*）。

### 1.2 后端 API

前缀 `/api/v1/brand`，JWT brand 中间件提供 brandID/userID，错误走 `response.Error` + `AppError`。

**RecurringSchedule**

| 方法 | 路径 | 权限门 | 请求字段 | 响应 |
|---|---|---|---|---|
| POST | /recurring-schedules | session.create | course_id, location_id, instructor_profile_id, location_resource_id?, weekdays[](0-6), start_date(YYYY-MM-DD), end_date?(YYYY-MM-DD), repeat_weeks?, start_time(HH:mm), duration_min, capacity? | `{recurring_schedule, created_count, skipped_count, created[]{…session}, skipped[]{date, start_time, reason}}` (201) |
| GET | /recurring-schedules | session.view | status?(active/cancelled/completed), location_id?, page?, page_size? | items[]{id, location_id, location_name, course_id, course_title, instructor_profile_id, instructor_name, location_resource_id, resource_name, weekdays[], start_date, end_date, repeat_weeks, start_time, duration_min, capacity, status, session_count, created_at} + total |
| GET | /recurring-schedules/:id | session.view | — | recurring_schedule（同上单条）+ `sessions[]`（该模板已生成场次，复用 ClassSession 投影，按 starts_at 升序） |
| PATCH | /recurring-schedules/:id/cancel | session.cancel | — | recurring_schedule（status=cancelled）|

校验与错误：
- weekdays 非空且 ∈ 0..6 去重；start_time `HH:mm`；duration_min>0；end_date/repeat_weeks 恰一；start_date ≥ 今天(Asia/Shanghai)；跨度 ≤26 周 & 生成 ≤200 节 → 否则 400 `SESSION_TIME_INVALID`。
- 批级校验（循环前，一次）：course 属本 brand+published（否则 `COURSE_NOT_ACTIVE`/`COURSE_NOT_FOUND`）；location active（`COURSE_LOCATION_UNAVAILABLE`/`LOCATION_NOT_FOUND`）；course 在 location 可用（`COURSE_LOCATION_UNAVAILABLE`）；instructor active+schedulable（`INSTRUCTOR_NOT_SCHEDULABLE`）；resource（若绑）active+同 location+未软删（`RESOURCE_NOT_AVAILABLE`/`RESOURCE_NOT_FOUND`）。
- 逐 occurrence：教练时段重叠→skip reason `instructor_conflict`；资源时段重叠→skip reason `resource_conflict`（按 EXCLUDE 约束名分流，复用 12a 的 `exclusionConstraint`）。
- 全部 skip（created_count==0）→ 整批 abort，409 `RECURRING_ALL_CONFLICT`，body.data 带 skipped 清单。
- cancel：仅 status='active' 可取消，否则 409 `RECURRING_CANCEL_NOT_ALLOWED`；不动已生成场次。
- list/detail/生成/cancel 全过 data_scope（assigned_locations 守卫 recurring.location_id）。
- 商业关键写（生成/cancel）写 OperationLog（target_type `recurring_schedule`）。

### 1.3 新增错误码（pkg/errors）

| 码 | HTTP | 含义 |
|---|---|---|
| `RECURRING_NOT_FOUND` | 404 | 循环排课不存在或越权 |
| `RECURRING_ALL_CONFLICT` | 409 | 全部 occurrence 冲突，未生成任何场次（body 带 skipped 清单） |
| `RECURRING_CANCEL_NOT_ALLOWED` | 409 | 仅 active 状态可取消 |

（参数校验复用 `SESSION_TIME_INVALID`/`INVALID_PARAM`；课程/门店/教练/资源校验复用 11/12a 错误码。）

### 1.4 后端落地清单（新域 recurringschedule）

- `internal/domain/recurringschedule/recurringschedule.go`：Schedule 实体（含 weekdays []int + 反范式名 + session_count）+ Status 常量 + GenerateInput + ListFilter + SkippedOccurrence + Repository 接口。
- `internal/application/recurringschedule/service.go`：`require` + `checker==nil` bypass + data_scope；入参校验（weekday/结束条件/上限/时区/start_date）；occurrence 日期生成（Asia/Shanghai）。
- `internal/infrastructure/persistence/recurring_schedule_models.go` + `recurring_schedule_repository.go`：外层 tx 插 recurring+weekdays → 批级校验 → 逐 occurrence SAVEPOINT 插 class_session（复用 `exclusionConstraint`/`sessionConflictError`、资源校验、容量优先级）→ created/skipped；List 反范式 JOIN + weekdays 聚合 + session_count；GetByID + sessions；Cancel（非级联）+ audit。
- `internal/interfaces/brand/recurring_schedule_handler.go`：4 endpoint，注册进 `handler.go`。
- 改 `location_repository.go` `CountActiveReferences`：+active recurring_schedules。
- wire：api-brand 重新生成。api-app/admin 不动。

### 1.5 前端页面模块

| 页面/模块 | 类型 | 关键字段/操作 |
|---|---|---|
| /schedule 页 Tabs | 改造 | 顶部加「单场次 / 循环排课」Tabs；单场次保持现状 |
| 循环排课列表 | Tab 面板 | 表（课程/门店/资源/教练/周几/起止/时段/已生成数/状态）+ 门店·状态筛选 + 「循环排课」按钮（门 session.create）+ 行「详情」「取消」（门 session.cancel）|
| 循环排课弹窗 | 弹窗 | 门店→已发布课程→可排课教练→资源(可选,级联,同单场次)→周几多选→开始日期→结束方式(结束日期 / 重复周数 二选一)→开始时间→时长→容量(资源/课程默认填充)；提交后展示结果：「成功 N 节 / 跳过 M 节」+ 跳过清单（日期·时间·原因 教练冲突/资源冲突）|
| 循环排课详情 | 弹窗/抽屉 | 模板信息 + 已生成场次列表（时间/状态）|
| 取消确认 | ConfirmDialog | 说明「仅停用该循环模板，已生成场次不会被取消，如需取消请到单场次逐个操作」|

前端落地：`packages/types` 增 RecurringSchedule 类型；`packages/api/src/recurring-schedules.ts`（client+hooks，含 package exports 登记）；`errors.ts` + 前端无需新权限码（复用 SESSION_*）；新增 3 错误码常量。

### 1.6 前端实现约束

- 复用 /web/apps/brand 现有组件（Tabs/DataTable/Select/Dialog/ConfirmDialog/Hint/usePermissions），不引入新 UI 库。
- 镜像 schedule/courses 页结构与 data-testid 命名（`recurring-create-button`/`recurring-row`/`recurring-field-*`/`recurring-submit`/`recurring-skipped-list`）。
- 周几多选用 checkbox 组，提交映射成 0-6（time.Weekday）。
- 「打开默认值」用 ref 一次性守卫（Batch 11 教训）。
- 时间→后端：start_date/start_time/duration 直接传，后端按 Asia/Shanghai 组装（不在前端拼时区）。

## 2. 验收闭环（端到端）

1. owner 登录 → /schedule「循环排课」Tab → 「循环排课」：门店讯美广场 → 已发布课程 → 教练张三 →（可选资源）→ 勾周一+周三 → 开始日期(下周一) → 重复 4 周 → 09:00 / 60 分钟 → 提交。
2. 结果显示「成功生成 8 节，跳过 0 节」；列表出现该循环排课，已生成数=8；/schedule 单场次 Tab 可见这 8 节。
3. 再建一个与已有场次时段冲突的循环（同教练同时段）→ 结果显示部分跳过 + 跳过清单（reason 教练冲突）。
4. 全部冲突的循环 → 提示「全部时段冲突，未生成」（RECURRING_ALL_CONFLICT），列表不新增。
5. 绑资源的循环与已占资源时段冲突 → 跳过清单 reason 资源冲突。
6. 取消某循环排课 → status=已取消；已生成的 8 节场次仍在（未被取消）。
7. 只读 13900139777：循环排课按钮 + 取消全 disabled。
8. （门店 guard）有 active 循环排课时删其门店 → LOCATION_IN_USE。

e2e 由用户另开 session 跑，主线程给自包含 prompt（账号/端口 8081/重启铁律/`@mini-schedule/brand` filter/rm .next）。
