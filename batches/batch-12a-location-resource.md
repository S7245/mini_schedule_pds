# Batch 12a：Location Resource 资源管理

状态：契约待 approve（会话内确认）

## 0. 范围与决策（grill 定论）

闭环主题：**品牌在门店下建可排课资源（教室/场地/线上/设备）→ 排单场次时可选绑资源 → 同一资源同一时段不能被两个有效场次占用（DB EXCLUDE 落地）→ 停用资源不能新排课 → 资源容量作为场次容量默认值 → 删除资源带引用保护 → 门店删除 guard 纳入 active 资源**。

> Batch 12 拆为 **12a（本批，资源管理）** + **12b（循环排课）**。12b 复用本批的资源选择器与 `SESSION_RESOURCE_CONFLICT`。

| # | 决策 | 结论 |
|---|---|---|
| D1 | 范围切分 | 本批：LocationResource CRUD（含软删带引用保护）+ 单场次排课**可选绑资源** + 资源时段冲突业务错误 + 资源容量默认值 + 门店删除 guard 纳入 active 资源。**推迟 12b**：RecurringSchedule 循环排课 |
| D2 | 资源端点形态 | **扁平 `/location-resources`**（list 用 `?location_id=` 过滤，create 在 body 带 location_id），镜像 location/course/class-session 现有扁平 handler，便于排课弹窗按门店级联拉取 |
| D3 | 权限码 | 新增 `location_resource.view/create/edit/delete` 4 码（domain=`location_resource`）+ role_template 映射 + 存量 brand backfill（镜像 000007）。`delete` Expand 隐含 `view+edit`，故拿 delete 的角色无需再显式 seed view/edit，但映射里仍全列以求显式 |
| D4 | 数据权限 | 资源 list/detail/写接口接 Batch 6 ScopeResolver：all_brand / assigned_locations（按 `location_id` 过滤/守卫），镜像 classsession.Service 的 `scopeFilterIDs`/`guardLocationInScope` |
| D5 | 删除策略 | **软删**（`location_resources.deleted_at`）。被 `class_sessions`(status scheduled/in_progress) 或 `recurring_schedules`(status active) 引用时拒删 → `RESOURCE_IN_USE`(409)。（12b 落地前 recurring 引用恒 0，提前写好不返工） |
| D6 | 资源/教练冲突分错误码 | 两条 EXCLUDE 都报 23P01，按 `pgErr.ConstraintName` 区分：`class_sessions_resource_no_overlap`→新 `SESSION_RESOURCE_CONFLICT`(409)；其余→`SESSION_INSTRUCTOR_CONFLICT`(409)。`isExclusionViolation` 重构为返回 `(bool, constraintName)` |
| D7 | 容量默认值 | 单场次 create 容量优先级：**显式 capacity > 绑定资源的 capacity > course.default_capacity**（blueprint §5.3）。未绑资源时维持现状（input > course.default_capacity） |
| D8 | 门店删除 guard | `CountActiveReferences` 纳入 active `location_resources`（deleted_at IS NULL AND status='active'）。落实 Batch 9/11 挂的 FR 的资源部分；recurring 部分留 12b |
| D9 | 额度门 | 同 Batch 11，资源不挂 SubscriptionGuard（blueprint §4.1 第一版不限制） |

## 1. 契约

### 1.1 数据库（migration 000008）

`location_resources` / `recurring_schedules` / `recurring_schedule_weekdays` 三表已在 000003 建好，**本批不动表结构**，仅补权限 seed。

```sql
-- 1) 4 条细粒度 permission（新 domain location_resource）
INSERT INTO permissions(code,domain,action,name,description) VALUES
  ('location_resource.view',   'location_resource','view',   '查看门店资源','查看 LocationResource'),
  ('location_resource.create', 'location_resource','create', '新增门店资源','创建 LocationResource'),
  ('location_resource.edit',   'location_resource','edit',   '编辑门店资源','编辑/启用停用 LocationResource'),
  ('location_resource.delete', 'location_resource','delete', '删除门店资源','软删 LocationResource')
ON CONFLICT (code) DO NOTHING;

-- 2) role_template 映射（镜像 000007）
--    brand_owner / brand_admin：全部 4 码
--    location_manager：全部 4 码（资源属其门店）
--    course_operator / instructor / receptionist：location_resource.view（排课需看资源）
-- 3) 存量 brand backfill：brand_role_permissions JOIN role_template_permissions（复用 step 2 映射）
```

> `delete` 的 Expand 自动隐含 `view+edit`（Batch 5 约定：delete→view+edit），故 location_manager 即便只 seed delete 也会得到 view/edit；映射仍全列 4 码以保证 PermissionSet 表显式一致。

### 1.2 后端 API

前缀 `/api/v1/brand`，JWT brand 中间件提供 brandID/userID，错误走 `response.Error` + `AppError`。

**LocationResource**

| 方法 | 路径 | 权限门 | 请求字段 | 响应 |
|---|---|---|---|---|
| GET | /location-resources | location_resource.view | `location_id`(必填), `status?`(active/inactive), `page?`, `page_size?` | items[]{id,location_id,location_name,name,type,capacity,status,remark,created_at} + total |
| GET | /location-resources/:id | location_resource.view | — | resource（同上单条 + location_name 反范式） |
| POST | /location-resources | location_resource.create | location_id, name, type, capacity?, remark? | resource |
| PATCH | /location-resources/:id | location_resource.edit | name?, type?, capacity?, status?, remark? | resource |
| DELETE | /location-resources/:id | location_resource.delete | — | 204 |

校验与错误：
- `location_id` 必填且属本 brand + active；不在 data_scope → `RESOURCE_NOT_FOUND`/`LOCATION_NOT_FOUND`(404)。
- `type` ∈ {classroom,venue,online,equipment,other}，否则 `INVALID_PARAM`(400)。
- `capacity` 省略/<=0 → 默认 1（DB DEFAULT），`>0` CHECK。
- 同门店重名（未软删）→ `RESOURCE_NAME_DUPLICATED`(409)（DB `unique(location_id,name) where deleted_at is null` 兜底，repo 判 23505）。
- PATCH `status` 仅 active/inactive。
- DELETE 引用保护：存在 scheduled/in_progress `class_sessions` 或 active `recurring_schedules` 引用 → `RESOURCE_IN_USE`(409)，否则软删（写 deleted_at）。
- list/detail/写全过 data_scope（assigned_locations 仅本人门店）。
- 商业关键写（create/update/delete）写 OperationLog（audit）。

**单场次排课扩展（class-sessions）**

| 方法 | 路径 | 变更 |
|---|---|---|
| POST | /class-sessions | body 增可选 `location_resource_id`(int64)。绑定时校验：资源属本 brand + 同 location_id + status='active' + 未软删，否则 `RESOURCE_NOT_AVAILABLE`(409)/`RESOURCE_NOT_FOUND`(404)。容量默认值改 D7 优先级。撞资源 EXCLUDE → `SESSION_RESOURCE_CONFLICT`(409) |
| GET | /class-sessions, /class-sessions/:id | 响应增 `location_resource_id` + 反范式 `resource_name`（LEFT JOIN location_resources） |

### 1.3 新增错误码（pkg/errors）

| 码 | HTTP | 含义 |
|---|---|---|
| `RESOURCE_NOT_FOUND` | 404 | 资源不存在或越权 |
| `RESOURCE_NAME_DUPLICATED` | 409 | 同门店资源重名 |
| `RESOURCE_IN_USE` | 409 | 删除资源时仍被 scheduled/in_progress 场次或 active 循环排课引用 |
| `RESOURCE_NOT_AVAILABLE` | 409 | 排课绑定的资源已停用/软删/跨门店 |
| `SESSION_RESOURCE_CONFLICT` | 409 | 同一资源同一时段重叠（DB EXCLUDE `class_sessions_resource_no_overlap`，23P01） |

### 1.4 后端落地清单（新域 locationresource，镜像 location）

- `internal/domain/locationresource/locationresource.go`：Resource 实体 + Status/Type 常量 + IsValid* + Repository 接口（List/GetByID/Create/Update/Delete/CountActiveReferences-on-resource）。
- `internal/application/locationresource/service.go`：`require(code)` + `checker==nil` bypass + `scopeFilterIDs`/`guardLocationInScope`（镜像 classsession）。
- `internal/infrastructure/persistence/location_resource_repository.go`：tx + audit；List 反范式 JOIN location_name；isUniqueViolation→RESOURCE_NAME_DUPLICATED；Delete 引用 COUNT 守卫。
- `internal/interfaces/brand/location_resource_handler.go`：5 endpoint，注册进 `handler.go`（mirror classSession 块）。
- 改 `class_session_repository.go`：`isExclusionViolation`→返约束名；Create 接 location_resource_id（校验+容量默认+冲突分流）；baseQuery LEFT JOIN resource 名。
- 改 `classsession` domain/service/handler：CreateInput/body 增 LocationResourceID。
- 改 `location_repository.go` `CountActiveReferences`：+active location_resources。
- wire：api-brand 重新生成（注入新 handler）。api-app / api-admin 不动。
- migration `000008_location_resource_permissions.up/down.sql`。

### 1.5 前端页面模块

| 页面/模块 | 类型 | 关键字段/操作 |
|---|---|---|
| `/resources` 资源管理 | 页面 | 门店下拉筛选（必选其一，默认第一个 active 门店）+ 状态筛选 + 资源表（名称/类型/容量/状态/备注）+ 分页；新增按钮（门 `location_resource.create`）；行操作 编辑/停用切换/删除（门对应权限，disabled+Hint）|
| 资源表单弹窗 | 弹窗 | RHF+zod：门店（create 时选，来自 active 门店；edit 锁定）、名称、类型(select)、容量(number)、备注；create/edit 用 `initial` 区分；提交映射 RESOURCE_NAME_DUPLICATED→「该门店已有同名资源」|
| 资源删除确认 | ConfirmDialog | 删除映射 RESOURCE_IN_USE→「该资源仍被未结束场次或循环排课占用，请先取消后再删除」，弹窗保持打开 |
| 资源停用切换 | 行内 toggle | active⇄inactive（门 `location_resource.edit`）|
| 排课弹窗扩展 | 弹窗 | 在「门店」之后增「资源（可选）」级联下拉：选门店后拉 `/location-resources?location_id=&status=active`；含「不绑定资源」空选项；选中资源后容量输入默认填资源容量（用户可改）；提交带 `location_resource_id`；错误映射增 SESSION_RESOURCE_CONFLICT→「该资源在此时段已被占用」、RESOURCE_NOT_AVAILABLE→「该资源已停用，请改选」|
| 场次表/详情 | 既有页 | /schedule 表增「资源」列（resource_name，空显「—」）|
| 导航 | sidebar/nav | 加「资源管理」`/resources`（门 `location_resource.view`，接 NAV_HREF_PERMISSIONS）|

前端落地：`packages/api/src/location-resources.ts`（client + hooks，mutation 失效 list）；`packages/types` 增 Resource 类型；`packages/api/src/errors.ts` + 前端 PERMISSIONS 常量增 5 错误码 / 4 权限码；`apps/brand` 新页 + 弹窗 + nav + 排课弹窗/表扩展。

### 1.6 前端实现约束

- 复用 `/web/apps/brand` 现有组件（DataTable/Select/Dialog/ConfirmDialog/Hint/usePermissions），不引入新 UI 库。
- 镜像 `/course-categories` + `/locations` 页结构与 data-testid 命名（`resource-create-button`/`resource-row`/`resource-field-*`/`resource-submit` 等）供 e2e。
- 「打开时默认值」用 ref 一次性守卫，不 keying on 派生 length（Batch 11 教训）。
- 时间输入→RFC3339 不变（本批不动单场次时间逻辑，仅加资源字段）。

## 2. 验收闭环（端到端）

1. owner 18816820405/admin123 登录 → /resources 选门店「讯美广场」→ 建资源「1 号教室」(classroom, cap 10)。
2. 重名建资源 → RESOURCE_NAME_DUPLICATED。
3. /schedule 排课：门店讯美广场 → 课程(已发布) → 教练张三 → 选资源「1 号教室」→ 容量自动填 10 → 建场次 A 成功。
4. 再排场次 B：同资源、时间重叠、**换教练**（避开教练冲突）→ SESSION_RESOURCE_CONFLICT 被拦。
5. 停用「1 号教室」→ 再排课选它 → RESOURCE_NOT_AVAILABLE（或下拉不出现）。
6. 删除被场次 A 引用的资源 → RESOURCE_IN_USE；取消场次 A 后删成功（204）。
7. 只读 13900139777/admin123：/resources 写按钮全 disabled。
8. （门店 guard）建 active 资源后删其门店 → LOCATION_IN_USE（active 资源计入）。

e2e 由用户另开 session 跑，主线程给自包含 prompt（账号/端口 8081/重启铁律/`@mini-schedule/brand` filter/rm .next）。
