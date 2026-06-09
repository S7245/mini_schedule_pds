# Batch 06 — RBAC enforcement + data_scope 落地 + 角色 read API

状态：**已 approve** ✅ — 进入 plan 阶段
approve 时间：2026-06-06
契约 lifecycle：draft → approved → **plan-in-progress** → tdd → reviewed → done
更新时间：2026-06-06

---

## 业务背景

Batch 5 把权限模型的"数据"全建好了（permissions / role_templates / brand_roles / brand_user_role_assignments），但**没"用"**：

- 任何 brand JWT 都能调所有 endpoints
- data_scope 字段在写不在用
- 非 owner 员工登录后菜单和数据范围与 owner 完全相同

本批让权限模型**实际生效**：service 层根据登录用户的权限拒绝越权请求；查询根据 data_scope 自动收紧；前端有独立 endpoint 拿当前用户能干什么。

自定义角色 CRUD 留 Batch 7。

```text
Batch 5: 数据建好但不生效
  ↓
Batch 6（本批）:
  - service 层显式 RequirePermission 校验
  - PermissionChecker（Redis 60s TTL 缓存 + edit 隐含 view）
  - data_scope 实际收紧 GET 查询（role_default / assigned_locations）
  - GET /me/permissions 暴露给前端
  - 预置 8 个角色 read API（POST/PATCH/DELETE 在 Batch 7）
  - 前端：菜单/按钮按权限隐藏 + 403 兜底
  ↓
Batch 7: 自定义角色 CRUD（品牌可基于 permission code 新建角色）
```

---

## Grill 阶段决定（14 个节点）

1. **范围档**：中等（不含自定义角色 CRUD，留 Batch 7）
2. **permission 装载**：Redis 60s TTL 缓存，key=`rbac:perms:{brand_user_id}`
3. **权限校验位置**：service 层显式 `RequirePermission(ctx, "staff.create")`
4. **未授权返**：403 for action（同 brand）/ 404 for cross-brand
5. **data_scope 实现**：注入 `ScopeResolver` 到 service，service 显式调 `resolver.Resolve(ctx, actorID)` 获取 location_ids 集合
6. **预置角色锁定**：8 个预置全锁；Batch 7 才允许品牌**新建**自定义角色
7. **permission 隐含**：内存计算时 `X.edit` 隐含 `X.view`、`X.delete` 隐含 `X.view` + `X.edit`、`X.create` 隐含 `X.view`
8. **当前用户权限暴露**：独立 `GET /api/v1/brand/me/permissions`
9. **own_sessions / own_records**：本批**不做**，留 Booking batch
10. **缓存失效**：TTL 自然过期（60s），无 pub/sub
11. **错误码**：新增 `PERMISSION_DENIED`（403），不复用 FORBIDDEN
12. **范围**：只做 brand 端，admin 端不动
13. **assigned_locations 收紧**：staff 列表只显示任职在我可见 location 的人
14. **测试基础**：用 brand_user_id=18（张三，location_manager @ Location1）

---

## 概念模型

### data_scope 推导规则

数据库 `brand_user_role_assignments.data_scope` 是用户在某角色上的**实际值**。但 `role_default` 是个 alias，实际生效值按 brand_role.scope_type 推导：

| 角色 scope_type | data_scope 写入值 | 实际生效值 |
|---|---|---|
| brand | `role_default` | **all_brand**（看全品牌）|
| brand | `all_brand` | all_brand |
| location | `role_default` | **assigned_locations**（看任职 location）|
| location | `assigned_locations` | assigned_locations |
| 任意 | `own_sessions` / `own_records` | **本批未实现** — service 层 fallback 到 assigned_locations 并日志告警 |

一个 staff 可能有多个 role_assignment（如同时 brand_admin + Location2 location_manager），合并规则：**取并集**。

- 如果任一 assignment 解析后是 `all_brand` → 整体 all_brand
- 否则收集所有 `assigned_locations` 的 location_id 并集
- 没有任何 assignment → 拒绝所有查询（除了 /me/* 和 /onboarding/* 等自服务接口）

### permission 集合计算

```
raw_permissions = SELECT p.code FROM brand_user_role_assignments bura
                   JOIN brand_role_permissions brp ON brp.role_id = bura.role_id
                   JOIN permissions p ON p.id = brp.permission_id
                  WHERE bura.brand_user_id = ? AND bura.status='active'

effective_permissions = raw_permissions ∪ implied(raw_permissions)

implied(X.edit)   → +X.view
implied(X.create) → +X.view
implied(X.delete) → +X.view, +X.edit
```

注意：implied 只在**内存**计算，不写回库。原因：将来扩 permission code 时不会污染历史数据。

---

## API 接口契约

### 后端：me / permissions

| 方法 | 路径 | 鉴权 | 请求 | 响应 |
|---|---|---|---|---|
| GET | `/api/v1/brand/me/permissions` | brand JWT | — | `{ permissions: string[], data_scope: { kind: 'all_brand' \| 'assigned_locations', location_ids: number[] } }` |

- `permissions` 是**effective** 集合（含隐含）
- `data_scope.kind` 是合并后的整体 scope
- `data_scope.location_ids` 仅当 kind='assigned_locations' 时给

### 后端：角色 read（Batch 7 加 CRUD）

| 方法 | 路径 | 鉴权 | 所需权限 | 响应 |
|---|---|---|---|---|
| GET | `/api/v1/brand/roles` | brand JWT | `staff.view`（已存在） | 列表：所有 active brand_roles（含 scope_type / is_system / permissions 数组） |
| GET | `/api/v1/brand/roles/:code` | brand JWT | `staff.view` | 单个角色详情（含 permission code 列表） |
| GET | `/api/v1/brand/permissions` | brand JWT | `staff.assign_role` | 全部 permissions 列表（code, domain, action, name），用于 Batch 7 自定义角色 UI 准备 |

Batch 5 已有 `/api/v1/brand/roles` 列表接口，本批扩**单个角色详情** + `/permissions` 全列表。

### 后端：现有接口加权限校验（service 层逐一回填）

不新增 endpoint，但所有现有 service method 头部插 `RequirePermission`：

| Service method | 所需权限 | data_scope 收紧？ |
|---|---|---|
| `staff.Service.List` | `staff.view` | 是（按 location_ids 过滤 staff_location_assignments）|
| `staff.Service.GetByID / GetWithAssignments` | `staff.view` | 是（target 必须任职在 location_ids 内，否则 404） |
| `staff.Service.Create` | `staff.create` | — |
| `staff.Service.Update` | `staff.edit` | 是（同 GetByID） |
| `staff.Service.UpdateStatus` | `staff.edit` | 是 |
| `staff.Service.Delete` | `staff.delete` | 是 |
| `staff.Service.ReplaceRoleAssignments` | `staff.assign_role` | 是 |
| `staff.Service.ReplaceLocationAssignments` | `staff.assign_location` | 是 |
| `staff.Service.GetInstructor / UpsertInstructor / DeleteInstructor` | `instructor.view` / `instructor.edit` | 是 |
| `staff.Service.ListRoles` | `staff.view` | 否（角色对全品牌通用）|
| `location.Service.List / Get` | `location.view` | 是（仅返 location_ids 内的）|
| `location.Service.Create` | `location.create` | — |
| `location.Service.Update / UpdateStatus` | `location.edit` | 是 |
| `location.Service.Delete` | `location.delete` | 是 |
| `brandprofile.Service.Get` | `brand.profile.view`（implied）| — |
| `brandprofile.Service.Update` | `brand.profile.edit` | — |
| `onboarding.Service.GetOnboardingStatus` | — | — | 自服务接口，所有员工都能访问 |
| `onboarding.Service.SkipStep` | `brand.profile.edit`（onboarding 是品牌资料管理的延伸） | — |
| `onboarding.Service.Complete` | `brand.profile.edit` | — |

### 错误码新增

```go
ErrPermissionDenied ErrorCode = "PERMISSION_DENIED"  // HTTP 403
```

返回包含 Details: `{ required: "staff.create", missing: ["staff.create"] }` 便于调试和未来权限申请页。

---

## 前端页面模块（apps/brand）

| 页面 / 模块 | 类型 | 改动 |
|---|---|---|
| 全局：登录后 fetch /me/permissions | hook | `usePermissions()` React Query hook + Context provider；60s staleTime |
| 工作台菜单 | layout | 按 `permissions.includes(...)` 隐藏菜单项（员工管理 / 门店 / 等）；owner 永远有所有菜单（permissions 集合覆盖）|
| 详情页按钮 | 各页面 | 创建/编辑/删除按钮按权限 disabled + tooltip "权限不足，请联系管理员"|
| 403 兜底 | 全局错误处理 | API 返 PERMISSION_DENIED → sonner toast + 高亮缺失权限码 |
| 角色管理菜单 | — | 仍**藏起来**（Batch 5 决定 8），等 Batch 7 上 CRUD 一起暴露 |

---

## 后端实现范围

### Domain（新增）
- `internal/domain/rbac/permission.go`：PermissionSet 类型（含 `Has(code)` / `HasAny(codes)` + edit 隐含 view 计算）+ DataScope 类型（kind + location_ids）

### Application
- `internal/application/rbac/checker.go`：**核心组件**
  - `Checker.Resolve(ctx, brandID, brandUserID) (*PermissionSet, *DataScope, error)`：查 DB + Redis 缓存 + 计算隐含 + 合并多 role_assignment
  - `Checker.Require(ctx, brandID, brandUserID, code) error`：返 `PERMISSION_DENIED` 或 nil
  - `Checker.RequireScope(ctx, brandID, brandUserID, kind, location_ids) error`：检查目标资源是否在 scope 内
- `internal/application/rbac/scope_resolver.go`：从 PermissionSet/DataScope 派生 SQL WHERE 子句构造器（service 用）
- 现有 service：构造函数加 `checker *rbac.Checker` 参数；每个 method 头部 `s.checker.Require(...)`

### Infrastructure
- `internal/infrastructure/persistence/rbac_repository.go`：
  - `LoadEffectiveRaw(ctx, brandID, brandUserID) ([]string, *DataScope, error)` — JOIN 三张表 + 合并
  - 不带缓存，缓存在 Checker 层用 cache pkg
- 用现有 `internal/infrastructure/cache` pkg 包 Redis，key=`rbac:perms:{brandUserID}`，TTL 60s

### Interfaces（新增）
- `internal/interfaces/brand/me_handler.go`：`GET /me/permissions`
- `internal/interfaces/brand/roles_handler.go`（扩展 Batch 5 既有的 roles read）：加 `/roles/:code` + `/permissions`
- 全局错误格式化：handler 不动，`response.Error` 已能处理 PERMISSION_DENIED + Details

### Wire 装配
- 新 service `rbac.NewChecker` + provider 注入到 staff / location / brandprofile / onboarding 的 service 构造函数
- Wire 三个 wire_gen.go 重生

---

## 数据库变更

**无 migration 改动**。所有所需表 Batch 5 已建：
- `permissions`（13 + 6 行 = 19 个 code）
- `brand_roles` / `brand_role_permissions` / `brand_user_role_assignments`

注意：测试环境（brand_id=21）现有数据已足够覆盖验收。

---

## 测试场景概览

详细测试场景待 approve 后写入 `pds/batches/batch-06-rbac-enforcement-data-scope-tests.md`。预期覆盖（用 brand_user_id=18 张三 作为非 owner 测试号）：

### Happy Path
- H1 张三登录 → GET /me/permissions → 19 个 permissions + data_scope=`assigned_locations` + location_ids=[1]
- H2 张三 GET /staff → 只返回任职在 Location1 的员工（含他自己 + Owner 如果 Owner 任职 Location1，否则只他自己）
- H3 张三 PATCH /staff/{自己}/instructor → 200（instructor.edit + 自己在 Location1）
- H4 owner（brand_user_id=16）登录 → GET /me/permissions → ~15 个 permissions（Batch 5 brand_owner 全权）+ data_scope=`all_brand`

### Edge Cases
- E1 张三 POST /staff（无 staff.create） → 403 `PERMISSION_DENIED` + Details `{required:"staff.create"}`
- E2 张三 PATCH /brand/profile（无 brand.profile.edit） → 403
- E3 张三 POST /locations（无 location.create） → 403
- E4 张三 DELETE /staff/{自己} → 403（无 staff.delete）
- E5 张三 GET /staff/{owner_id}（owner 不在 Location1） → 404 STAFF_NOT_FOUND（按 data_scope 收紧后看不到）
- E6 张三 PATCH /staff/{自己}/role-assignments → 403（无 staff.assign_role）
- E7 张三 PATCH /staff/{自己}/location-assignments → 200（有 staff.assign_location）— 注意：能自己改自己的任职
- E8 张三 GET /onboarding/status → 200（自服务接口）
- E9 张三 PATCH /onboarding/steps/staff/skip → 403（无 brand.profile.edit）
- E10 隐含权限：删除张三的 instructor.edit 权限关联后，instructor.view 仍应通过隐含获取 → 验证 implies 规则反向（删除高权限不会废低权限）

### Cache 行为
- E11 第一次 GET /me/permissions → DB 查询 + Redis SET
- E12 立即第二次 → Redis HIT，不查 DB
- E13 60s 后第三次 → Redis MISS + 重查
- E14 在 admin 把张三的 location_manager 改成 inactive → 60s 内张三仍有权限（接受 stale），60s+1s 后丢权限

### 跨域
- E15 张三 GET /staff/{别 brand 的 staff_id} → 404（已有 cross-brand 隔离）

### Permission inheritance
- E16 张三有 instructor.edit → effective set 中也有 instructor.view（隐含）
- E17 张三没有 staff.edit → effective set 中也没有 staff.view（不会反向隐含；但他有 staff.view 是直接的，不影响）

---

## Plan 阶段：subagent task 切分（待 approve 后输出）

待 spec approve 后，plan 阶段会输出独立 plan 文件，含：
- 8-10 个后端 task DAG（domain rbac → checker → scope resolver → 各 service 回填 → 新 handler → Wire → 静态验证）
- 4-5 个前端 task DAG（hook + Context → 菜单按权限隐藏 → 按钮 disabled + tooltip → 全局 403 兜底 → /me/permissions hook 联调）
- 每个 task 红绿步骤
- 风险点（最大风险：service 层回填要改 ~20 个 method，DAG 必须保证不破 Batch 5 现有测试）

---

## 等用户 approve

逐条回 OK / 修改：

1. service 层显式 `RequirePermission` 而非 middleware 风格，OK？将带来 ~20 处 service method 头部插一行的 boilerplate
2. data_scope.role_default 推导规则（brand-scope → all_brand / location-scope → assigned_locations）OK？
3. permission 隐含规则（edit 隐含 view 等）OK？
4. `GET /me/permissions` 响应里同时给 permissions 数组 + 整体 data_scope（kind + location_ids），OK？
5. PERMISSION_DENIED 错误码 Details 含 `{required, missing}`，OK？
6. Batch 5 已落地接口（staff 11 个 + location 6 个 + onboarding 3 个 + profile 2 个）都加 RequirePermission，OK？特别注意 onboarding 三个接口的权限映射（status 不要权限 / skip + complete 用 brand.profile.edit）
7. 前端 60s staleTime 的 /me/permissions hook + 菜单按权限隐藏 + 按钮 disabled，OK？
8. 测试用 brand_user_id=18 张三，按 location_manager 权限矩阵验收 E1-E17，OK？

逐条 OK 后我会进 plan 阶段，输出独立 plan 文件给你过目，再 spawn TDD subagent。
