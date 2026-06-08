# Batch 05 — Staff CRUD + InstructorProfile + 预置角色 + SubscriptionGuard 重构

状态：**已 approve** ✅ — 进入 plan 阶段
approve 时间：2026-06-06
更新时间：2026-06-06
契约 lifecycle：draft → approved → **plan-in-progress** → tdd → reviewed → done

---

## 业务背景

Batch 4 完成了向导骨架 + 第 1-2 步（品牌资料 + Location）。第 3 步「员工 / 授课员工」要求 `active brand_user ≥ 1 AND instructor_profile ≥ 1`，目前只有 owner 一条 brand_user 在场。本批让向导第 3 步真实可用，同时**清掉 Batch 4 review 转出去的 SubscriptionGuard 重构债**（Staff/Learner quota 都靠它复用）。

权限模型本批落"中等"档：预置 role_templates + brand 注册自动复制 + 角色分配 API。**不**做 RBAC enforcement —— middleware 拦截留 Batch 6。

```text
Batch 4: Brand 初始化向导 + Location
  ↓
Batch 5（本批）:
  - SubscriptionGuard 抽出，Location 改造复用
  - Staff CRUD（新 /staff 接口，与 Batch 1 /users 共存）+ staff_seats quota
  - InstructorProfile 1:1 嵌在 Staff 详情
  - 预置 4 个域权限 + 8 个角色模板（seed migration）
  - Brand 注册流程改造 + owner backfill（自动分配"品牌负责人"）
  - 角色分配 + Location 任职关联 API
  - 向导第 3 步真实闭环
  ↓
Batch 6: RBAC enforcement（middleware + data_scope 落地 + 品牌自定义角色 CRUD）
```

---

## Grill 阶段决定（详见会话历史，10 个根节点已 approve）

1. 范围档位：**中等**（不含 RBAC enforcement / data_scope 拦截）
2. SubscriptionGuard：**本批顺手抽**，Location 改造复用
3. Permission seed 范围：**只当前域**（brand / location / staff / instructor，约 20 个 code）
4. /staff 接口：**新建** `/api/v1/brand/staff`，老 `/users` 标 deprecated 保留 owner 流程
5. Owner backfill：**migration 老数据 + 改造新注册流程，双轨**
6. Owner 删除：**禁止**（必须至少 1 个 active owner）
7. data_scope：本批落 `role_default` + `assigned_locations`，`own_*` 留 Booking batch
8. 角色管理菜单：**藏起来**，第一版不暴露给品牌
9. InstructorProfile：**内嵌**员工详情页（"晋升为教练"开关 + 字段表单）
10. Soft delete：**查询时 JOIN 过滤** `brand_users.deleted_at IS NULL`，不级联

---

## 数据库变更

### Migration 000005 — Brand owner 标记 + permission/role seed

```sql
-- 000005_staff_roles_seed.up.sql

-- 1) brand_users 加 is_owner 字段（若未存在）
ALTER TABLE brand_users ADD COLUMN IF NOT EXISTS is_owner BOOLEAN NOT NULL DEFAULT FALSE;
CREATE INDEX IF NOT EXISTS idx_brand_users_brand_owner ON brand_users(brand_id) WHERE is_owner = TRUE;

-- 2) Permissions seed —— 仅本批涉及的 4 个域，约 20 个 code
INSERT INTO permissions (code, domain, action, name, description) VALUES
    ('brand.profile.view',        'brand',      'view',   '查看品牌资料',   '查看 Brand 的基本信息'),
    ('brand.profile.edit',        'brand',      'edit',   '编辑品牌资料',   '修改 Brand 资料字段'),
    ('location.view',             'location',   'view',   '查看门店',       '查看 Location 列表 / 详情'),
    ('location.create',           'location',   'create', '新增门店',       '创建 Location'),
    ('location.edit',             'location',   'edit',   '编辑门店',       '修改 Location 信息'),
    ('location.delete',           'location',   'delete', '删除门店',       '软删 Location'),
    ('location.toggle_status',    'location',   'edit',   '启用/停用门店',  '切换 Location 启用状态'),
    ('staff.view',                'staff',      'view',   '查看员工',       '查看 Staff 列表 / 详情'),
    ('staff.create',              'staff',      'create', '新增员工',       '创建 Staff'),
    ('staff.edit',                'staff',      'edit',   '编辑员工',       '修改 Staff 基本信息'),
    ('staff.delete',              'staff',      'delete', '删除员工',       '软删 Staff（owner 不可删）'),
    ('staff.assign_role',         'staff',      'edit',   '分配角色',       '给 Staff 分配 / 移除角色'),
    ('staff.assign_location',     'staff',      'edit',   '分配门店任职',   '给 Staff 分配 / 移除 Location'),
    ('instructor.view',           'instructor', 'view',   '查看教练档案',   '查看 InstructorProfile'),
    ('instructor.edit',           'instructor', 'edit',   '维护教练档案',   '维护 InstructorProfile（创建/编辑/启用停用）')
ON CONFLICT (code) DO NOTHING;

-- 3) Role templates seed —— 8 个蓝图列出的标准角色
INSERT INTO role_templates (code, name, scope_type, description) VALUES
    ('brand_owner',        '品牌负责人',     'brand',    '品牌全部权限（owner 默认）'),
    ('brand_admin',        '品牌管理员',     'brand',    '除商业 / 订阅设置外的品牌管理权限'),
    ('course_operator',    '课程运营',       'brand',    '课程模板和场次管理（本批先占位，权限待 Batch 6 课程域补全）'),
    ('finance_aftercare',  '财务/售后',     'brand',    '订单 / 退款 / 学员售后（占位）'),
    ('location_manager',   '店长',           'location', 'Location 范围内的全部运营权限'),
    ('location_reception', '前台',           'location', 'Location 接待 / 签到（占位）'),
    ('location_instructor','Instructor',     'location', '授课与个人场次维护'),
    ('location_assistant', '门店协助',       'location', '辅助操作（占位）')
ON CONFLICT (code) DO NOTHING;

-- 4) Role template ↔ Permission 关联（brand_owner 全权；其余 8 个仅赋当前 4 域内合理子集）
WITH p AS (SELECT id, code FROM permissions)
INSERT INTO role_template_permissions (template_id, permission_id)
SELECT t.id, p.id
FROM role_templates t
CROSS JOIN p
WHERE
    -- brand_owner：本批 4 域全部
    (t.code = 'brand_owner' AND p.code IN (
        'brand.profile.view','brand.profile.edit',
        'location.view','location.create','location.edit','location.delete','location.toggle_status',
        'staff.view','staff.create','staff.edit','staff.delete','staff.assign_role','staff.assign_location',
        'instructor.view','instructor.edit'
    ))
    -- brand_admin：除 staff.delete 外的 staff/location/brand/instructor 全部
 OR (t.code = 'brand_admin' AND p.code IN (
        'brand.profile.view','brand.profile.edit',
        'location.view','location.create','location.edit','location.toggle_status',
        'staff.view','staff.create','staff.edit','staff.assign_role','staff.assign_location',
        'instructor.view','instructor.edit'
    ))
    -- location_manager：本店 location/staff/instructor 查看 + 编辑
 OR (t.code = 'location_manager' AND p.code IN (
        'location.view','location.edit','location.toggle_status',
        'staff.view','staff.assign_location','instructor.view','instructor.edit'
    ))
    -- location_instructor：只看自己 + 维护自己档案
 OR (t.code = 'location_instructor' AND p.code IN (
        'staff.view','instructor.view','instructor.edit'
    ))
    -- course_operator / finance_aftercare / location_reception / location_assistant：占位，本批不赋权
ON CONFLICT (template_id, permission_id) DO NOTHING;

-- 5) Backfill：给所有现存 brand 创建 brand_roles 并把 owner 的 brand_user 关联到"品牌负责人"
-- 详见 service 层 Batch 5 起跑前的 backfill 脚本（手动跑一次，幂等）。
```

```sql
-- 000005_staff_roles_seed.down.sql
DELETE FROM role_template_permissions WHERE template_id IN (
    SELECT id FROM role_templates WHERE code IN (
        'brand_owner','brand_admin','course_operator','finance_aftercare',
        'location_manager','location_reception','location_instructor','location_assistant'
    )
);
DELETE FROM role_templates WHERE code IN (
    'brand_owner','brand_admin','course_operator','finance_aftercare',
    'location_manager','location_reception','location_instructor','location_assistant'
);
DELETE FROM permissions WHERE code LIKE 'brand.%' OR code LIKE 'location.%' OR code LIKE 'staff.%' OR code LIKE 'instructor.%';
ALTER TABLE brand_users DROP COLUMN IF EXISTS is_owner;
```

**注意**：owner backfill 的"复制 role_templates → brand_roles + role_template_permissions → brand_role_permissions + owner brand_user_role_assignments"流程**用代码实现**（onboarding `Service.BackfillOwnerRoles`），不放 SQL migration ——因为它依赖动态 brand_id 列表 + 多表事务，SQL 写出来反而难维护。Batch 5 起跑前用一次性 admin 工具调用，幂等。

---

## API 接口契约

### 后端：SubscriptionGuard（共享）

`internal/application/commercial/subscription_guard.go`（新）：

```go
type ResourceKind string

const (
    ResourceLocation ResourceKind = "location"
    ResourceStaff    ResourceKind = "staff"
    ResourceLearner  ResourceKind = "learner"
)

// CheckAndCount 在外部传入的 tx 里执行：
//   1. SELECT FOR UPDATE active 且未过期的 subscription
//   2. COUNT 当前 brand 下该 kind 的资源
//   3. 超 max → 返 QUOTA_EXCEEDED + Details{current, max}；未超 → 返 (count, max, nil)
//
// 调用方负责自己的 INSERT；guard 只负责"锁 + 数 + 判"。
func (g *SubscriptionGuard) CheckAndCount(
    ctx context.Context, tx *gorm.DB, brandID int64, kind ResourceKind,
) (current, max int64, err error)
```

`Location` 域改造为复用 SubscriptionGuard；同时 `subscriptionGuardCounter` 接口注入按 kind 路由的 COUNT 子查询（Location 走 `locations WHERE deleted_at IS NULL`，Staff 走 `brand_users WHERE deleted_at IS NULL AND is_owner=false OR ... `——见下方 Staff 域细节）。

### 后端：Staff 域

新 domain `internal/domain/staff/` + application `internal/application/staff/`。

| 方法 | 路径 | 鉴权 | 请求字段 | 响应字段 |
|---|---|---|---|---|
| GET    | `/api/v1/brand/staff`                                           | brand JWT | `?page&page_size&status&with_instructor` | `{items: [...], total, page, page_size}`，可选 join instructor_profile 摘要 |
| GET    | `/api/v1/brand/staff/:id`                                       | brand JWT | — | Staff 详情（含 instructor_profile / role_assignments / location_assignments） |
| POST   | `/api/v1/brand/staff`                                           | brand JWT | `{phone, name, initial_password, role_codes[], location_assignments[]}` | 新建 Staff |
| PATCH  | `/api/v1/brand/staff/:id`                                       | brand JWT | `{name?}`（phone 不允许改） | 更新后的 Staff |
| PATCH  | `/api/v1/brand/staff/:id/status`                                | brand JWT | `{status: 'active'\|'inactive'}` | 状态切换；owner 不允许置 inactive |
| DELETE | `/api/v1/brand/staff/:id`                                       | brand JWT | — | 软删；owner 拒绝 |
| PUT    | `/api/v1/brand/staff/:id/role-assignments`                      | brand JWT | `{assignments: [{role_code, location_id?, data_scope}]}` | 全量替换角色任职关系 |
| PUT    | `/api/v1/brand/staff/:id/location-assignments`                  | brand JWT | `{assignments: [{location_id, assignment_type, is_primary}]}` | 全量替换 Location 任职关系 |
| GET    | `/api/v1/brand/staff/:id/instructor`                            | brand JWT | — | Instructor profile（404 表示该 staff 未启用为教练） |
| PUT    | `/api/v1/brand/staff/:id/instructor`                            | brand JWT | `{display_name, avatar_url?, bio?, specialties?, certificates?, is_visible_to_learners, is_schedulable, status}` | Upsert instructor profile（含 1:1 校验） |
| DELETE | `/api/v1/brand/staff/:id/instructor`                            | brand JWT | — | 注销 instructor profile（不影响 staff 本身） |
| GET    | `/api/v1/brand/roles`                                           | brand JWT | — | brand 的角色列表（含预置 + 后续自定义；本批 read-only） |

**关键错误码**：
- `STAFF_PHONE_DUPLICATED` 409 — 手机号已被同 brand 的 active staff 占用
- `OWNER_PROTECTED` 409 — 尝试删除 / 停用 / 降级唯一 active owner
- `ROLE_NOT_FOUND` 404 — role_code 不存在或非本 brand 可见
- `LOCATION_ASSIGNMENT_INVALID` 400 — location_id 不属于本 brand 或已软删
- `INSTRUCTOR_PROFILE_NOT_FOUND` 404
- `SUBSCRIPTION_RESTRICTED` 403 / `QUOTA_EXCEEDED` 409 — 与 Batch 4 同
- `INVALID_PARAM` 400 — 通用参数校验

**OperationLog 写入点**：
- `staff_created` / `staff_deleted` / `staff_status_changed`
- `staff_role_assignments_changed` / `staff_location_assignments_changed`
- `instructor_profile_upserted` / `instructor_profile_deleted`

⚠️ Batch 4 FR 提到"OperationLog 合并 audit pkg"——本批顺手抽 `internal/audit/audit.go`，统一 actor type 枚举（`brand_user` / `platform_admin` / `system`）+ metadata 序列化。Location 已有的 `writeLocationOperationLog` 改造为复用。

### 后端：Brand 注册流程改造

`internal/application/commercial/service.go` 的 `CreatePublicSignupOrder`：
- 创建 owner brand_user 时同步：
  - 设置 `is_owner = true`
  - 调 `staffRoleAllocator.AssignDefaultOwnerRoles(ctx, brandID, brandUserID)` 自动创建 brand_roles（如果该 brand 还没复制过预置角色）+ INSERT `brand_user_role_assignments` 为"品牌负责人"+ `data_scope=all_brand`

`internal/application/staff/role_allocator.go`（新）：
- `AssignDefaultOwnerRoles(ctx, brandID, brandUserID)` — 幂等：
  1. 查 brand_roles，没有 brand_owner → 从 role_templates 复制全套 8 个 brand_roles 行（含 brand_role_permissions）
  2. INSERT brand_user_role_assignments(brand_user_id, role_id=brand_owner, location_id=NULL, data_scope='all_brand') ON CONFLICT DO NOTHING
- 这个方法也是 backfill 工具调的入口（admin 端可触发对所有现存 brand 跑一遍）

### 后端：Backfill admin 接口（一次性）

`POST /api/v1/admin/system/backfill-owner-roles` — admin JWT，遍历所有 `brand_users.is_owner=true` 调 `AssignDefaultOwnerRoles`。响应包含 `{processed, skipped, failed}` 统计。

---

## 前端页面模块（apps/brand）

| 页面 / 模块 | 类型 | 关键字段 / 操作 |
|---|---|---|
| `/(protected)/staff` | 页面 | DataTable（姓名 / 手机号 / 角色 / Location 任职 / 状态 / 教练标识）+ 新增按钮 + 搜索 + 状态过滤 |
| `/(protected)/staff/[id]` | 页面 | 三段：基础信息 / 角色 + Location 任职 / 教练档案（折叠卡） |
| `StaffCreateDialog` | 组件 | RHF + Zod：phone / name / initial_password / role 多选 / location_assignments 配置 |
| `StaffRoleAssignmentEditor` | 组件 | 多行表单：role_code（Select）+ location（Select / null）+ data_scope（Radio: role_default / assigned_locations） |
| `StaffLocationAssignmentEditor` | 组件 | 多行表单：location（Select）+ assignment_type（Select：member/manager/instructor/assistant）+ is_primary 单选 |
| `InstructorProfileSection` | 组件 | "晋升为教练"开关 / 完整字段表单 / 启用停用 |
| `ConfirmDialog` | 复用 Batch 4 | owner 删除被拒、状态切换确认 |
| `/(protected)/onboarding/staff` | **改造** | 第 3 步占位 → 真实页：嵌入 staff 列表 + 创建 Dialog；至少 1 个 active staff + 1 个 instructor_profile 才能继续 |

工作台首页菜单加 `员工管理`（不加 `角色管理`，per 决定 8）。

---

## 后端四层实现范围

### Domain（新增）
- `internal/domain/staff/staff.go` + `repository.go`
- `internal/domain/role/role.go`（轻量：BrandRole / Permission 类型 + 查询接口）
- `internal/domain/instructor/instructor.go`

### Application
- `internal/application/staff/service.go`：编排 CRUD + quota + 角色 / Location 任职管理 + instructor upsert
- `internal/application/staff/role_allocator.go`：owner 自动分配
- `internal/application/commercial/subscription_guard.go`（新公共组件，本批的最大重构）
- `internal/application/commercial/service.go`：注册流程改造

### Infrastructure
- `internal/infrastructure/persistence/staff_repository.go` + `staff_models.go`
- `internal/infrastructure/persistence/role_repository.go` + `role_models.go`
- `internal/infrastructure/persistence/instructor_repository.go` + `instructor_models.go`
- `internal/infrastructure/persistence/location_repository.go`：去掉内联 quota，改用 SubscriptionGuard
- `internal/audit/audit.go`（新 pkg）：统一 OperationLog 写入入口

### Interfaces
- `internal/interfaces/brand/staff_handler.go`（新，含 11 个 endpoint）
- `internal/interfaces/admin/system_handler.go`：加 backfill 接口
- 旧 `brand/handler.go` 的 `/users` 接口：注释标 deprecated，行为不变

### Wire
- 三个新 service + 一个 SubscriptionGuard + audit pkg 加进 wire.go；同时 `go generate` 重生

---

## 测试场景概览

详细测试场景待 approve 后写入 `pds/batches/batch-05-staff-instructor-roles-tests.md`。预期覆盖：

### Happy Path
- 登录 owner → 进员工管理 → 添加 staff（phone, name, password + 角色"店长"+ Location 任职 Location1 manager primary）→ staff 出现在列表 → 进入详情 → "晋升为教练" → 填资料 → 列表显示教练标识 → 向导第 3 步显示 completed

### Edge Cases
- E1 添加 staff 撞 max_staff_seats → 409 QUOTA_EXCEEDED + Details
- E2 phone 重复 → 409 STAFF_PHONE_DUPLICATED
- E3 删除唯一 owner → 409 OWNER_PROTECTED
- E4 停用唯一 owner → 409 OWNER_PROTECTED
- E5 PUT role-assignments 含未知 role_code → 404 ROLE_NOT_FOUND
- E6 PUT location-assignments 含跨 brand 的 location_id → 400 LOCATION_ASSIGNMENT_INVALID
- E7 重复 PUT instructor profile（同 brand_user_id）→ 实际是 update，1:1 不冲突
- E8 staff 注销 instructor profile 后查教练列表 → 不在
- E9 backfill 接口跑两次 → 第二次 skipped=N（幂等）
- E10 Subscription frozen → 403 SUBSCRIPTION_RESTRICTED
- E11 owner 在删除前先转移 ownership 给另一个 active staff（本批**不**做转移功能，留 Batch 6）→ E3 拒绝逻辑生效
- E12 并发创建 staff 撞 max → 一成一败（同 Batch 4 E24 测试模式）

---

## Plan 阶段：subagent task 切分

待 spec approve 后，**plan 阶段**输出在会话中给你过目，再 spawn subagent。预切：

**后端**（预计 12 个 task commit）：
1. migration 000005 + embed
2. `internal/audit/` pkg 抽出，Location 改造复用
3. `commercial/subscription_guard.go` 抽出（含单元测试覆盖 quota 边界）
4. Location 改造为复用 SubscriptionGuard（含 location_repository.go 简化）
5. domain staff / role / instructor 类型 + repository interface + 单测
6. role_repository（查询 brand_roles / permissions）
7. instructor_repository（CRUD + 1:1 校验）
8. staff_repository（CRUD + 软删 + soft-delete join 过滤）
9. application staff service：CRUD + quota + owner 保护 + 单测覆盖 E1-E12
10. application role_allocator + 注册流程改造（含 backfill 接口实现）
11. interfaces staff_handler（11 endpoints）+ admin system_handler backfill
12. Wire 重生 + 错误码扩 + 静态验证

**前端**（预计 7 个 task commit）：
1. types + api clients（staff / roles / instructor）+ 错误码常量
2. `/staff` 列表页（DataTable + 搜索 + 状态过滤）
3. StaffCreateDialog（RHF + Zod + 角色 / Location 多选）
4. `/staff/[id]` 详情页布局 + 基础信息编辑
5. StaffRoleAssignmentEditor + StaffLocationAssignmentEditor
6. InstructorProfileSection（折叠卡 + 字段表单 + 开关）
7. `/onboarding/staff` 改造：嵌入 staff 列表 + CTA

每个 task commit 按 `feat(batch-5-be): ...` / `feat(batch-5-fe): ...` 模板。

---

## 验收清单

**后端**：
- [ ] `go build ./...` + `go vet ./...` 通过
- [ ] `go generate ./...` 干净（no diff）
- [ ] 单测覆盖 Staff service + SubscriptionGuard + role_allocator + instructor upsert
- [ ] migration 000005 up/down 顺序可执行 + dirty 防御
- [ ] Location 改造后 Batch 4 测试场景（H/E1-E18）仍 100% 通过

**前端**：
- [ ] `pnpm --filter @mini-schedule/brand exec tsc --noEmit` 通过
- [ ] `pnpm --filter @mini-schedule/brand build` 通过
- [ ] 员工管理列表 / 详情 / 创建 Dialog / 任职编辑器 / Instructor 折叠卡均能渲染

**业务验收**：
- [ ] backfill 接口对现存 brand id=21 跑通，owner brand_user 自动获得"品牌负责人"角色
- [ ] 新注册品牌的 owner 自动有角色（注册流程改造生效）
- [ ] 创建第 N 个 staff 命中 quota → Details 显示 current/max
- [ ] 删 / 停用唯一 owner → 拒绝
- [ ] staff 详情页晋升为教练 → 向导第 3 步在 GetCounts 实时聚合后翻 completed

---

## 等用户 approve

逐条回 OK / 修改：

1. 11 个 staff endpoints 接口形状 OK？特别是 PUT 全量替换 role-assignments / location-assignments 的语义（替代 POST/DELETE 单条管理）
2. role_template 预置 8 个 + permission 4 域 20 个的 seed 范围 OK？
3. backfill 接口挂 admin 侧 `/api/v1/admin/system/backfill-owner-roles` OK？还是放别处？
4. `/staff` 路径独立菜单，不藏在 onboarding 下 OK？还是合并到向导第 3 步内？
5. audit pkg 抽出（Batch 4 FR）同步本批做 OK？还是分批？
6. 测试场景 E1-E12 覆盖够用 OK？是否要补 owner 转移 / data_scope 单测？

逐条 OK 后我会：
- 写测试场景文件 `batch-05-staff-instructor-roles-tests.md`
- 输出**plan 阶段**精细化 task 切分（含每个 commit 的红绿步骤）给你过目
- 并行 spawn 后端 + 前端 subagent，TDD 逐 task commit
- 静态验证 → `/code-review` → 发邮件请你做业务验收
