# Batch 7 启动预备 — 品牌自定义角色 / 调整权限 CRUD

> 本文件是 **grill 设计树预备**，不是契约。新 session 应先按 `pds/CLAUDE.md` 步骤 2 跟用户 grill 完下面的开放决策，再写 `batch-07-custom-roles.md` 契约并发邮件等 approve。
> 作用：把 RBAC 现状和待决策点前置整理好，让 grill 直接切要害，不重复摸代码。

---

## 1. 目标 & 范围

品牌后台让 owner / 有权限的管理员**自建角色、调整角色权限、停用/删除角色**，并把角色分配给员工（分配 UI 在 Batch 5 已做）。这是 Batch 5/6 两次列入的 RBAC 收尾主线。

只做 **brand 端**（与 Batch 6 一致）。admin 端不动。

---

## 2. RBAC 现状（已建好，作为 grill 的事实基础）

### 表结构（migration 000003 + 000005）

- **`brand_roles`**：`id, brand_id, template_id(可空,ON DELETE SET NULL), code(VARCHAR 80), name, scope_type('brand'|'location'), is_system(BOOL default TRUE), status('active'|'inactive'), description`
  - 唯一索引 `idx_brand_roles_brand_code (brand_id, code)` → **同 brand 内 code 唯一**
  - `is_system` 区分预置(TRUE)与自定义(FALSE) —— **自定义角色 CRUD 的关键开关**
- **`brand_role_permissions`**：`brand_id, role_id(ON DELETE CASCADE), permission_id(ON DELETE CASCADE)`，唯一 (role, permission)
- **`brand_user_role_assignments`**：`brand_user_id, role_id(ON DELETE CASCADE), location_id(可空), data_scope('role_default'|'all_brand'|'assigned_locations'|'own_sessions'|'own_records'), status`
  - ⚠️ `role_id` 是 **ON DELETE CASCADE** —— 硬删角色会连带删掉所有任职记录，危险，删除策略必须 grill。
- **`permissions`**：14 条细粒度 code（见下），`code, domain, action, name, description`

### 14 个细粒度 permission code（按 domain 分组，前端勾选树的数据源）

| domain | codes |
|---|---|
| brand | brand.profile.view / brand.profile.edit |
| location | location.create / location.edit / location.delete / location.toggle_status / location.view |
| staff | staff.view / staff.create / staff.edit / staff.delete / staff.assign_role / staff.assign_location |
| instructor | instructor.view / instructor.edit |

（注：000003 还有一套"粗粒度"占位 permission 与之并存；Batch 7 只用细粒度这套，前端 `PERMISSIONS` 常量即此 14 个。）

### 8 个预置 role_templates → 每 brand 复制成 8 个 is_system=TRUE 的 brand_roles
brand_owner / brand_admin / location_manager / instructor(location_instructor) / receptionist(location_reception) / finance_support(finance_aftercare) 等。复制逻辑在 `application/staff/role_allocator.go: EnsureBrandRolesSeeded`。

### 已有的读接口
- `GET /roles` 列表 —— 已存在（`staff_handler.go:listRoles` → `svc.ListRoles`）。
- 缺 `GET /roles/:code` 单条详情 + `GET /permissions` 全量列表（Batch 6 T08 延后，本批补，角色编辑器需要）。见 `backend/.learnings/FEATURE_REQUESTS.md`。

### 权限解析链路（Batch 6）
service 层 `require(code)` 校验；`Checker.Resolve`：ctx-cache → Redis L1 **60s TTL** → DB；owner fast-path；data_scope `role_default` → all_brand / assigned_locations。**改角色权限后缓存失效是本批必须处理的点**。

---

## 3. grill 设计树 —— 待用户决策（新 session 逐条问）

### A. 数据模型边界
- **A1 系统角色(is_system=TRUE)可改吗?** 典型方案:权限只读、不可改名、不可删,只能"复制为自定义角色"再改。还是允许调系统角色权限? → 决定写路径是否拦 is_system。
- **A2 自定义角色 code 怎么生成?** 用户填(校验唯一+格式) / 系统生成(custom_xxx / 雪花)。前端要不要暴露 code 还是只给 name?
- **A3 scope_type 创建时选,创建后能否改?** 改 brand↔location 会影响存量任职的 data_scope 推导语义。
- **A4 删除角色策略?**（FK 是 CASCADE，硬删危险）选项：① 有任职引用时禁止删，提示先移除任职 ② 软删(status=inactive)保留历史 ③ 级联删任职(需二次确认+审计)。建议 ① 或 ②。

### B. 权限编辑
- **B1 权限提升防护(安全关键)**:管理员能否创建/编辑出"比自己权限更大"的角色?Batch 5 review B4 已堵过"静默清空 owner 角色"的提权漏洞。本批必须明确:分配的 permission 集合 ⊆ actor 自己的有效权限?owner 例外?
- **B2 权限隐含落库策略**:UI 勾 edit 是否自动连带 view?后端 Expand 在内存做(Batch 6),落库存"原始勾选"还是"展开后"?建议落库存原始,解析时 Expand。
- **B3 角色 data_scope 默认值**:角色本身定 scope_type,但实际 data_scope 在 assignment 上(role_default / all_brand / assigned_locations)。自定义角色的 role_default 怎么映射?(brand scope→all_brand, location scope→assigned_locations,与 Batch 6 一致)

### C. 跨域依赖 / 缓存
- **C1 改角色权限后缓存失效**:持有该角色的**多个** brand_user 的 Redis L1(60s)要主动批量 invalidate,还是接受最多 60s 延迟自然过期?(Batch 6 Q10 对单用户选了自然过期;改角色影响面更大,需重新定。) 建议:写成功后按 role→users 反查 + 批量 DEL key,或保守接受 60s。
- **C2 GET /permissions + GET /roles/:code** 本批一并补(Batch 6 延后项)。

### D. 失败模式
- D1 删/停用角色时并发有人正被分配该角色（SELECT FOR UPDATE on role 行?）。
- D2 brand_owner 系统角色全程保护：不可删、不可去掉 brand.* 关键权限、不可降级（接 B4 漏洞防线）。
- D3 唯一冲突(brand_id, code) → 友好错误码（ROLE_CODE_DUPLICATED?）。

### E. 验收闭环（端到端业务流）
owner 登录 → 角色管理页 → 新建"前台兼职"(location scope, 勾 staff.view + location.view) → 在某员工详情页分配该角色 + location → 该员工登录看到对应菜单/按钮 → owner 编辑角色去掉 location.view → 员工刷新后该入口消失（验证缓存失效策略）→ 尝试删除仍有任职的角色（按 A4 策略验证）→ 验证系统角色按 A1 策略不可删/改。

---

## 4. 预估接口面（待 grill 后定稿，仅供 grill 时对照）

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | /api/v1/brand/permissions | 全量 permission（按 domain 分组）— 补 Batch 6 延后 |
| GET | /api/v1/brand/roles/:code | 单角色详情 + 权限明细 — 补 Batch 6 延后 |
| POST | /api/v1/brand/roles | 新建自定义角色（name, scope_type, description, permission_codes[]） |
| PUT | /api/v1/brand/roles/:code | 改名/描述 + 全量替换权限（permission_codes[]） |
| PATCH | /api/v1/brand/roles/:code/status | 停用/启用 |
| DELETE | /api/v1/brand/roles/:code | 删除（按 A4 策略） |

权限 gate：上述写接口需要一个新 permission（如 `role.manage` / `staff.assign_role` 复用？grill B 时定）。

## 5. 预估前端模块（待定稿）

| 模块 | 类型 | 关键点 |
|---|---|---|
| /roles 角色管理 | 页面 | 列表区分系统角色(badge)/自定义角色；新建按钮按权限 disabled+Hint |
| 角色编辑器 | Dialog/页面 | name + scope_type + description + permission 勾选树(按 domain 分组，隐含联动) |
| 角色删除/停用 | 弹窗 | ConfirmDialog；有任职引用时按 A4 提示 |
| 工作台菜单 | — | 加"角色管理"入口（Batch 5 当时刻意没加） |

复用：Batch 6 的 `Hint`(disabled tooltip)、`usePermissions`、fail-closed；员工详情页角色分配入口已存在。

---

## 6. 必带的历史防线（review/learnings 已记，勿回退）
- 权限提升漏洞（Batch 5 B4）：不能通过角色编辑给自己/他人提权或动 owner 角色。
- 跨用户缓存泄漏铁律（Batch 6 ERRORS）：本批不新增 user-scoped 前端 query 即可，但若加角色列表 query 注意会话边界已统一 `queryClient.clear()`。
- data_scope 越权返 404 不 403（Batch 6）。
- 前端会迭代的数组字段(permission_codes 等)**不要 omitempty**（Batch 5 坑）。
- handler 收 `[]string` 而非 string（Batch 5 坑）。
