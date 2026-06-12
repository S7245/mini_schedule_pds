# Batch 7 契约 — 品牌自定义角色 / 调整权限 CRUD

> 状态：**已完成 ✅**（契约 2026-06-12 approve；同日端到端验收通过：E1–E8 + E4-编辑变体 + Happy #7/#8/#10 全绿，C1 在 Redis 层证实）。预备文件见 `batch-07-custom-roles-prep.md`，进度账本见 `../PROGRESS.md` Batch 7 区块。
>
> 验收备注：① B1 最终采用**增量校验**（option B，非原契约的全集严格）；② `location.view` 前端无可见门（`/locations` 页未建，Batch 4 FR 遗留），Happy #7/#8 的 location 入口在 UI 层无体现，C1 仅 API+Redis 层验证 → 转 FR。
> 范围：仅 **brand 端**。admin 端不动。RBAC 现状已在 prep 文件，并经本批 Explore 复核确认。

---

## 0. grill 设计树决策（已与用户确认 2026-06-11）

| 编号 | 决策点 | 定论 |
|---|---|---|
| **A1** | 系统角色(is_system=TRUE)可改性 | **完全只读**：权限只读、不可改名、不可改 scope、不可删；只能「复制为自定义角色」再改。写路径统一拦 `is_system=TRUE`。 |
| **A2** | 自定义角色 code 生成 | **系统生成**，前端只暴露 name。后端生成 `custom_<短随机/序号>`，保证 (brand_id, code) 唯一。 |
| **A3** | scope_type 可改性 | **创建时选定，创建后锁定不可改**（避免 brand↔location 改动破坏存量任职 data_scope 语义）。 |
| **A4** | 删除策略 | **有任职引用时禁止删**，返回友好错误 `ROLE_IN_USE`，提示先到员工详情移除该角色任职；无引用时方可硬删。 |
| **B1** | 权限提升防护（安全关键） | 创建/编辑角色时，`permission_codes[]` 必须 **⊆ actor 自己的有效权限集**；**owner 例外**（fast-path 全权）。越权返回 `ROLE_PERMISSION_EXCEEDS_ACTOR`。 |
| **B2** | 权限隐含落库 | **落库存原始勾选**（不展开），解析时由 Batch 6 `Expand()` 在内存做隐含（edit→view 等）。 |
| **B3** | role_default 映射 | 与 Batch 6 一致：brand scope → `all_brand`，location scope → `assigned_locations`。本批不改解析逻辑。 |
| **C1** | 改角色权限后缓存失效 | **主动批量 invalidate**：写成功后按 role→users 反查所有持有该角色的 brand_user，批量 `DEL rbac:perms:<userID>`。 |
| **D2** | owner 系统角色全程保护 | brand_owner 系统角色不可删、不可改、不可降级（接 Batch 5 B4 提权防线）。 |
| **D3** | code 唯一冲突 | 系统生成已避免；保留 `ROLE_CODE_DUPLICATED` 兜底。 |
| **gate** | 写接口权限门 | **新增细粒度 permission `role.manage`**（domain=`role`），seed 给 brand_owner / brand_admin 模板，并 backfill 存量 brand。 |

---

## 1. 契约

### API 接口

base path：`/api/v1/brand`，沿用现有 staff_handler 中间件链（JWT + GetBrandID/GetUserID）。

| 方法 | 路径 | 权限 gate | 请求字段 | 响应字段 |
|---|---|---|---|---|
| GET | `/permissions` | `role.manage` | — | `[{ domain, permissions: [{ code, action, name, description }] }]`（按 domain 分组的全量细粒度权限）|
| GET | `/roles/:code` | `staff.view` | — | `BrandRole`（含 permissions 明细 + is_system + scope_type + status）— 复用已有 `svc.GetRole`，仅补 handler 路由 |
| POST | `/roles` | `role.manage` | `name`, `scope_type`('brand'\|'location'), `description?`, `permission_codes[]` | 新建的 `BrandRole`（code 由后端生成） |
| PUT | `/roles/:code` | `role.manage` | `name`, `description?`, `permission_codes[]`（全量替换）| 更新后的 `BrandRole` |
| PATCH | `/roles/:code/status` | `role.manage` | `status`('active'\|'inactive') | 更新后的 `BrandRole` |
| DELETE | `/roles/:code` | `role.manage` | — | `204`（无任职引用时硬删） |

**写接口统一不可对 `is_system=TRUE` 角色生效**（A1/D2）：命中返回 `ROLE_IS_SYSTEM`（HTTP 409）。

**字段约束**
- `permission_codes[]`：JSON 数组，handler 收 `[]string`，**不加 `omitempty`**（Batch 5 坑）。允许空数组（= 无权限角色）。
- `scope_type` 仅 POST 接受；PUT **忽略/拒绝** scope_type 变更（A3）。
- `name`：必填，1–40 字符，brand 内不强制唯一（code 才唯一）。

**新增错误码（`pkg/errors/error.go`）**

| Code | HTTP | 触发 |
|---|---|---|
| `ROLE_IS_SYSTEM` | 409 | 试图改/删系统角色 |
| `ROLE_IN_USE` | 409 | 删除时仍有 active 任职引用（A4） |
| `ROLE_PERMISSION_EXCEEDS_ACTOR` | 403 | 勾选权限超出 actor 有效权限（B1，非 owner） |
| `ROLE_CODE_DUPLICATED` | 409 | (brand_id, code) 冲突兜底（D3） |
| `ROLE_NOT_FOUND` | 404 | 已存在，复用 |
| `OWNER_PROTECTED` | 409 | 已存在，复用（D2 brand_owner）|

**缓存失效（C1）**：`UpdateRole` / `PATCH status` / `DELETE` 成功后，repo 反查 `brand_user_role_assignments` 中引用该 role_id 的全部 active `brand_user_id`，逐一 `DEL rbac:perms:<userID>`（key 格式见 `rbac/checker.go`）。在事务提交后做（避免回滚后误删缓存导致一次多余 DB 回源，可接受）。

**权限提升校验（B1）**：service 层在 POST/PUT 落库前，若 actor 非 owner，`Resolve(actor)` 取其有效权限集，校验 `permission_codes ⊆ effective`，否则 `ROLE_PERMISSION_EXCEEDS_ACTOR`。owner（is_owner=TRUE）跳过。

### 数据库迁移（`000006_role_manage_permission`）

up：
1. `INSERT INTO permissions (code, domain, action, name, description) VALUES ('role.manage','role','manage','角色管理','创建/编辑/删除自定义角色') ON CONFLICT (code) DO NOTHING;`
2. 映射到模板：`INSERT INTO role_template_permissions (template_id, permission_id)` 选 `role_templates.code IN ('brand_owner','brand_admin')` × `permissions.code='role.manage'`，`ON CONFLICT DO NOTHING`。
3. **Backfill 存量 brand**：`INSERT INTO brand_role_permissions (brand_id, role_id, permission_id)` —— 对所有 `brand_roles.code IN ('brand_owner','brand_admin')` 注入 `role.manage`，`ON CONFLICT DO NOTHING`。（owner 走 fast-path 本不需要，但 brand_admin 必须 backfill，否则存量 brand 的 admin 看不到角色管理。）

down：删除上述映射与 permission 行（可回滚范围内补 down）。

> 注：`is_system` 复制逻辑、`EnsureBrandRolesSeeded` 不改；新建自定义角色走新 `CreateRole`，显式置 `is_system=FALSE`。

---

## 2. 前端页面模块

| 页面/模块 | 类型 | 关键字段/操作 |
|---|---|---|
| `/roles` 角色管理页 | 页面 | 角色列表表格：name、scope_type badge、系统/自定义 badge(`is_system`)、status、权限数；「新建角色」按钮按 `role.manage` disabled + `Hint`。系统角色行操作仅「查看 / 复制为自定义」，自定义角色行「编辑 / 停用启用 / 删除」。 |
| 角色编辑器 | Dialog | name + scope_type(仅新建可选，编辑时只读) + description + **权限勾选树**（按 domain 分组，数据源 `GET /permissions`）。隐含联动仅 UI 提示（落库存原始）。超出 actor 权限的项 disable + Hint（前端预防 B1，后端兜底）。 |
| 复制为自定义 | 操作 | 系统角色 → 预填其 name(加「副本」)/scope/permissions 打开编辑器走 POST。 |
| 删除/停用确认 | 弹窗 | 复用 `ConfirmDialog`；删除命中 `ROLE_IN_USE` 时 toast 提示「该角色仍有员工任职，请先在员工详情移除」。 |
| 工作台菜单 | 导航 | `config/nav.tsx` 加「角色管理」入口；`layout.tsx` `NAV_HREF_PERMISSIONS['/roles'] = 'role.manage'`。 |

### 前端契约补充
- `packages/types`：`BrandRole`/`BrandRolePermission` 已存在；新增 `CreateRoleInput`/`UpdateRoleInput`、`PermissionGroup` 类型。
- `packages/api/src/roles.ts`：在现有 `listBrandRoles` 旁补 `getBrandRole(code)`、`createBrandRole`、`updateBrandRole`、`patchRoleStatus`、`deleteBrandRole`、`listPermissions` + 对应 react-query hooks；mutation `onSuccess` invalidate `['brand-roles']`。
- `apps/brand/lib/permissions.tsx`：`PERMISSIONS` 加 `ROLE_MANAGE: 'role.manage'`。

### 前端实现约束
- 复用 `/web/apps/brand` 现有 shadcn 组件（Dialog/Table/Select/Card/Hint/ConfirmDialog）、`usePermissions`/`Hint` fail-closed 模式，不引入新 UI 库。
- 不新增 user-scoped query（守 Batch 6 跨用户缓存泄漏铁律）；角色列表 query 是 brand-scoped，会话边界已有 `queryClient.clear()`。
- 迭代数组字段 `permission_codes` 序列化**不加 omitempty**。

---

## 3. 验收闭环（端到端业务流）

owner 登录 → 工作台见「角色管理」入口 → 新建「前台兼职」(location scope, 勾 `staff.view` + `location.view`) → 列表出现自定义 badge → 到某员工详情分配该角色 + 某 location → 该员工登录看到对应菜单/按钮 → owner 编辑角色去掉 `location.view` → 员工**刷新后**该入口立即消失（验证 C1 主动失效，非等 60s）→ 尝试删除仍有任职的角色 → 提示 `ROLE_IN_USE` 拒绝 → 移除员工该任职后删除成功 → 验证系统角色 brand_owner 行无「编辑/删除」、仅「复制为自定义」（验证 A1/D2）→ 非 owner 管理员尝试新建超出自身权限的角色 → 被 `ROLE_PERMISSION_EXCEEDS_ACTOR` 拒绝（验证 B1）。

---

## 4. 任务拆分（subagent TDD 逐 task commit 参考）

**后端**
- T01 migration 000006 + backfill（go test 起库顺序 up）
- T02 `pkg/errors` 新错误码
- T03 repo：CreateRole / UpdateRole(替换权限) / DeleteRole / role→users 反查 / ListPermissions
- T04 service：CreateRole/UpdateRole/PatchStatus/DeleteRole（is_system 拦截、B1 提权校验、A4 引用检查）+ 缓存批量失效
- T05 handler：POST/PUT/PATCH status/DELETE /roles + GET /roles/:code + GET /permissions（路由 + 权限 gate）

**前端**
- T06 types + api(roles.ts) + hooks + PERMISSIONS.ROLE_MANAGE
- T07 角色管理页 `/roles`（列表 + badge + 权限 gate 按钮）
- T08 角色编辑器 Dialog（权限勾选树 + scope 只读规则 + 复制为自定义）
- T09 删除/停用确认 + 错误码 toast 映射
- T10 nav 入口 + 权限 gate
- T11（延后项回收）端到端 Playwright 回归（Batch 6 未补的 T10）
