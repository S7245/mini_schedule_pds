# Batch 05 Plan — task 切分 + 依赖图 + subagent 提示词

阶段：**plan**（spec 已 approve，TDD 启动前）
本文是给主线程 + subagent 看的执行手册。

---

## 后端 task DAG（13 个 task commit）

```text
T01 migration 000005 + embed ──┐
                                │
T02 internal/audit/ pkg ─────┐  ├──┐
                              │  │  │
T03 SubscriptionGuard 抽出 ──┘  │  │
                                 │  │
T04 Location 改造复用 SG ───────┴──┤  ← T02+T03 都依赖到，T04 验证两者落地
                                    │
T05 domain staff / role / instructor 类型 ──┐
                                              │
T06 role_repository（读 brand_roles）─────────┤
                                              │
T07 instructor_repository（CRUD + 1:1）──────┤
                                              │
T08 staff_repository（CRUD + soft-delete join 过滤）──┐
                                                      │
T09 application staff service ────────────────────────┤
                                                      │
T10 role_allocator + 注册流程改造 ────────────────────┤
                                                      │
T11 interfaces staff_handler + admin backfill ──────┤
                                                      │
T12 Wire 重生 + 错误码扩 ──────────────────────────┤
                                                      │
T13 静态验证 + 集成测试跑通 ────────────────────────┘
```

并发机会：T01 / T02 / T03 / T05 / T06 / T07 之间相互独立，可并行 spawn。但单 subagent 串行 commit 也够快，本批不再拆并发子代理。

---

## 后端 task 细节（红绿步骤）

### T01 — migration 000005 + embed
- **红**：写 `migrations/000005_staff_roles_seed.up.sql` 包含 `ALTER TABLE brand_users ADD COLUMN IF NOT EXISTS is_owner` + 4 域 ~15 个 permission INSERT + 8 个 role_templates INSERT + role_template_permissions WITH CTE 关联
- **绿**：`make migrate-up` 跑通；查 `SELECT count(*) FROM permissions WHERE domain IN (...)` 验证数量
- **down**：DELETE 三表对应行 + DROP column
- **commit**：`feat(batch-5-be): migration 000005 — seed permissions + role_templates`

### T02 — internal/audit/ pkg 抽出
- **红**：单元测试 `audit_test.go` 期待 `audit.Write(tx, audit.Event{...})` 接受统一参数 + 写 OperationLogModel + metadata 自动序列化 + actor_type 枚举校验
- **绿**：
  - `internal/audit/audit.go`：`type Event struct { BrandID *int64; Actor Actor; Action string; Target Target; Reason string; Before, After any }`
  - `Actor` / `Target` / `ActorType` enum（`actor_brand_user` / `actor_platform_admin` / `actor_system`）
  - `Write(tx *gorm.DB, e Event) error` —— marshal before/after 到 metadata，调 `tx.Create(&persistence.OperationLogModel{...})`
- **注意**：audit pkg 不应反向依赖 persistence；OperationLogModel 通过 interface 注入或者把 model struct 挪到 `internal/audit/model.go`。**取舍**：留 model 在 persistence 包，audit pkg 接受 `*gorm.DB`，用 `tx.Table("operation_logs").Create(map[string]interface{}{...})` 避免循环依赖
- **commit**：`refactor(batch-5-be): extract internal/audit pkg for unified OperationLog writes`

### T03 — SubscriptionGuard 抽出
- **红**：单元测试 `subscription_guard_test.go` 期待：
  - `Guard.CheckAndCount(tx, brandID, ResourceLocation)` 返 (current=2, max=3, nil) 当未达上限
  - 返 (3, 3, *AppError{QUOTA_EXCEEDED with Details}) 当达上限
  - 返 (0, 0, *AppError{SUBSCRIPTION_RESTRICTED}) 当 subscription 不可用
  - 不同 ResourceKind 走不同 COUNT 查询和 max 字段
- **绿**：`internal/application/commercial/subscription_guard.go`
  ```go
  type ResourceKind string
  const (ResourceLocation, ResourceStaff, ResourceLearner ResourceKind = ...)
  type Guard struct{}
  func (g *Guard) CheckAndCount(ctx, tx, brandID, kind) (current, max int64, err error)
  ```
  内部按 kind 分发 COUNT SQL（locations / brand_users / brand_learner_profiles）+ 各自的 max 字段（MaxLocations / MaxStaffSeats / MaxLearners）
- **commit**：`feat(batch-5-be): extract SubscriptionGuard with parameterised ResourceKind`

### T04 — Location 改造为复用 SubscriptionGuard
- **红**：跑 Batch 4 既有 location_repository 单测 / 集成测试 + E35–E38 期望仍通过
- **绿**：location_repository.Create 内联 SELECT FOR UPDATE + COUNT 替换为 `guard.CheckAndCount(tx, brandID, ResourceLocation)`
- **commit**：`refactor(batch-5-be): location_repository reuses SubscriptionGuard`

### T05 — domain staff / role / instructor 类型
- **绿**：
  - `internal/domain/staff/staff.go`：Staff entity + Status + 输入输出类型 + Repository interface
  - `internal/domain/role/role.go`：BrandRole + Permission 类型 + Repository（查询） 接口
  - `internal/domain/instructor/instructor.go`：InstructorProfile 实体 + 状态 + 输入输出类型 + Repository interface
- **commit**：`feat(batch-5-be): domain staff/role/instructor types and repository interfaces`

### T06 — role_repository
- 查询 brand_roles（含 permissions JOIN）/ role_templates（用于 backfill 时复制）
- 提供：`ListBrandRoles(ctx, brandID)`、`GetBrandRoleByCode(ctx, brandID, code)`、`ListRoleTemplatesWithPermissions(ctx)`
- **commit**：`feat(batch-5-be): role_repository (brand_roles / role_templates read paths)`

### T07 — instructor_repository
- CRUD + 1:1 校验（同 brand_user_id 已有时 update 不冲突）
- `GetByBrandUserID` / `Upsert` / `Delete`
- **commit**：`feat(batch-5-be): instructor_repository CRUD with 1:1 upsert`

### T08 — staff_repository
- CRUD + 软删（`deleted_at IS NULL` 过滤）
- 关联查询：`GetWithAssignments(ctx, brandID, id)` 一次拉 brand_user + role_assignments + location_assignments + instructor_profile
- `ListByBrand(ctx, brandID, filter, page, pageSize)` 支持 status / has_instructor 过滤
- `ReplaceRoleAssignments(ctx, tx, brandUserID, assignments)` 全量替换
- `ReplaceLocationAssignments(ctx, tx, brandUserID, assignments)` 全量替换
- **commit**：`feat(batch-5-be): staff_repository CRUD + assignment replacement`

### T09 — application staff service
- CRUD（POST 内事务：staff INSERT + role_assignments INSERT + location_assignments INSERT + audit.Write + SubscriptionGuard）
- Owner 保护：`Delete` / `UpdateStatus(inactive)` 时校验 `is_owner==true && active_owner_count(brand)==1` 拒
- 角色校验：`ReplaceRoleAssignments` 时校验 role_code 存在 + active + scope_type 匹配 location_id 配置
- 注入 `audit.Write` 到所有 mutation
- **单测**：覆盖 H3 / E1-E10 / E13-E21
- **commit**：`feat(batch-5-be): staff service with quota + owner protection + role/location assignment`

### T10 — role_allocator + 注册流程改造
- `internal/application/staff/role_allocator.go`：
  - `EnsureBrandRolesSeeded(ctx, tx, brandID)`：如果 brand 还没 brand_roles，从 role_templates 复制 8 个 + 复制 role_template_permissions 到 brand_role_permissions。幂等
  - `AssignDefaultOwnerRoles(ctx, tx, brandID, brandUserID)`：调上面 + INSERT brand_user_role_assignments (role=brand_owner, scope=all_brand) ON CONFLICT DO NOTHING
- 修改 `commercial.Service.CreatePublicSignupOrder` (Batch 1 入口)：创建 owner brand_user 后调 `AssignDefaultOwnerRoles`，所有动作在同事务
- admin backfill 接口 `BackfillOwnerRoles(ctx)`：遍历 `brand_users WHERE is_owner=true`，逐 brand 调 `AssignDefaultOwnerRoles`，返回 `{processed, skipped, failed}`
- **单测**：E27-E29 + 幂等性
- **commit**：`feat(batch-5-be): role_allocator + brand registration ownership backfill`

### T11 — interfaces staff_handler + admin backfill handler
- `internal/interfaces/brand/staff_handler.go`（11 endpoints）
- `internal/interfaces/admin/system_handler.go` 加 `POST /system/backfill-owner-roles`
- handler 不写业务，调 service；统一 `response.Success` / `response.Error`
- 旧 `/api/v1/brand/users` 接口加 `// Deprecated: use /api/v1/brand/staff (Batch 5+)` 注释，行为不变
- **commit**：`feat(batch-5-be): staff_handler (11 endpoints) + admin backfill route`

### T12 — Wire 重生 + 错误码扩
- 新错误码：`STAFF_PHONE_DUPLICATED` / `STAFF_NOT_FOUND` / `OWNER_PROTECTED` / `ROLE_NOT_FOUND` / `LOCATION_ASSIGNMENT_INVALID` / `INSTRUCTOR_PROFILE_NOT_FOUND`
- `go generate ./...` 重生三个 wire_gen.go
- **commit**：`chore(batch-5-be): error codes + wire regen for staff/role/instructor providers`

### T13 — 静态验证 + 集成测试
- 不是单独 commit，是 task 完成的 gate：`go build / vet / test ./internal/{application,domain,audit,interfaces,infrastructure}/...`
- 失败回退到对应 task 修

---

## 前端 task DAG（7 个 task commit）

```text
F01 types + api clients ──┐
                           │
F02 /staff 列表页 ──────────┤
                           │
F03 StaffCreateDialog ───┤
                           │
F04 /staff/[id] 详情页 ────┤
                           │
F05 StaffRoleAssignmentEditor + LocationAssignmentEditor ──┤
                                                            │
F06 InstructorProfileSection 折叠卡 ────────────────────────┤
                                                            │
F07 /onboarding/staff 真实化 + 工作台菜单加员工管理 ─────────┘
```

每个 F0X commit 前跑 `pnpm --filter @mini-schedule/brand exec tsc --noEmit`。

### F01 — types + api clients
- `apps/brand/types/index.ts`：加 `Staff`, `StaffStatus`, `RoleAssignment`, `LocationAssignment`, `InstructorProfile`, `BrandRole`
- `apps/brand/lib/api/staff.ts` / `roles.ts` / `instructor.ts`：fetch 函数 + React Query hook
- 错误码常量 STAFF_*, OWNER_PROTECTED, ROLE_NOT_FOUND, LOCATION_ASSIGNMENT_INVALID, INSTRUCTOR_PROFILE_NOT_FOUND, QUOTA_EXCEEDED, SUBSCRIPTION_RESTRICTED
- **commit**：`feat(batch-5-fe): staff/role/instructor types + api clients`

### F02 — /staff 列表页
- 新路由 `(protected)/staff/page.tsx`
- 列：姓名 / 手机号 / 角色（chips）/ 主 Location / 状态 / 教练标识 / 操作
- 顶部：搜索框 + 状态过滤 + 新增按钮
- 复用 admin-system DataTable / FilterBar 模式（per web/.learnings Pre-Batch-4 baseline）
- **commit**：`feat(batch-5-fe): /staff list page with search + status filter`

### F03 — StaffCreateDialog
- RHF + Zod：phone（手机号正则），name（1-50 字），initial_password（8+ 字母+数字），role_codes 多选（从 useBrandRoles 拿，排除 brand_owner），location_assignments 多行
- 提交 → POST /staff，409 QUOTA_EXCEEDED 显示 toast + Details 里 current/max
- **commit**：`feat(batch-5-fe): StaffCreateDialog with role + location assignment editors`

### F04 — /staff/[id] 详情页
- 三段：基础信息（姓名 inline 编辑）/ 角色任职 / Location 任职 / Instructor 折叠卡
- 删除按钮（owner 禁用 + tooltip 解释）
- 状态切换 toggle（复用 Batch 4 LocationStatusToggle 模式）
- **commit**：`feat(batch-5-fe): /staff/[id] detail page layout + basic edit`

### F05 — Role/Location AssignmentEditor 组件
- 二者结构相似：list-of-rows + add/delete row + per-row 表单
- RoleAssignmentEditor 行：role_code（Select）/ location_id（仅 location-scope 角色才显示）/ data_scope（Radio）
- LocationAssignmentEditor 行：location_id（Select）/ assignment_type（Select）/ is_primary（Radio，全列唯一）
- PUT 全量替换语义
- **commit**：`feat(batch-5-fe): StaffRoleAssignmentEditor + StaffLocationAssignmentEditor`

### F06 — InstructorProfileSection
- 折叠卡（Card + Collapsible）
- 未启用时：CTA"晋升为教练" → 展开表单
- 已启用时：摘要 + "编辑"按钮展开 + "注销教练资格"链接按钮（红色，需 ConfirmDialog）
- **commit**：`feat(batch-5-fe): InstructorProfileSection embedded in staff detail`

### F07 — onboarding 第 3 步 + 工作台菜单
- `(protected)/onboarding/staff/page.tsx`：检测 status==='completed' 时显示完成态；未完成时嵌入 StaffList + StaffCreateDialog（复用，不复制）
- 工作台 layout 菜单加 `员工管理` 项（不加角色管理）
- **commit**：`feat(batch-5-fe): onboarding step 3 real flow + staff menu`

---

## subagent 提示词大纲（spawn 前最终化）

### 后端 subagent
- 必读：本 plan 文件 + 契约 + 测试场景 + backend/.learnings/{LEARNINGS,ERRORS,FEATURE_REQUESTS}.md（特别注意"JSONB BeforeCreate"清债已部分完成；audit pkg 抽出在 T02、SubscriptionGuard 抽出在 T03，**不要**留 inline 副本）
- 13 个 task 按 DAG 顺序逐 commit
- 每个 task commit 前：`go build / vet / test` 对应包；红绿步骤遵守
- 完成后报告：commit SHA 列表 / 覆盖的测试场景编号 / 留下的 TODO

### 前端 subagent
- 必读：本 plan + 契约 + 测试场景 + web/.learnings/{LEARNINGS,ERRORS,FEATURE_REQUESTS}.md（特别看 Batch 4 onboarding/locations patterns + Batch 5 reuse checklist）
- 7 个 task 按 DAG 顺序逐 commit
- 每个 task commit 前：`pnpm --filter @mini-schedule/brand exec tsc --noEmit`
- 完成后报告：commit SHA + 截图（如能）+ 留下的 TODO

---

## 风险点 / 预案

1. **role_template_permissions 的 INSERT CTE 写法**：Postgres 支持 `INSERT ... SELECT ... WHERE ...`，但 ON CONFLICT 子句必须放 SELECT 之后。提前在迁移本地手测过；T01 跑 `make migrate-up` 失败则回滚。
2. **audit pkg 与 persistence 包循环依赖**：方案是 audit pkg 用 raw `tx.Table("operation_logs").Create(map)` 不引 persistence。T02 写时如发现需要更多结构化，fallback 把 OperationLogModel 移到 audit 包。
3. **SubscriptionGuard 重构破坏 Batch 4 测试**：T04 验证 E35-E38 必须 100% 通过；若 fail 是 SG 抽出时漏 grace_ends_at/expires_at 双窗口。
4. **角色 backfill 跨事务的幂等性**：T10 单测 E28（第二次跑）必须 skipped；用 `ON CONFLICT DO NOTHING` 兜底。
5. **brand_users.phone 唯一索引跨 brand**：意味着同一手机号不能在两个 brand 都注册。如不符合产品预期需改索引，但本批不动（蓝图未明说）。E3 测试场景说明此约束。
6. **Wire 重生失败**：T12 跑 `go generate` 时若依赖图复杂报错，回到 plan 重审；预案是手改 wire_gen.go 但**禁止留 TODO 注释**，必须立刻 commit 一份"as-if-generated"版本。

---

## Stage checkpoint：plan 阶段产出确认（待用户过目）

- [ ] 13 个后端 task + DAG OK？
- [ ] 7 个前端 task + DAG OK？
- [ ] 红绿步骤够清楚？
- [ ] 风险点 + 预案覆盖够全？

确认后我会：
- spawn 后端 + 前端 subagent，按 DAG 逐 task TDD commit
- 完成后跑静态验证 + `/code-review` 自检本批 diff
- 发邮件请你做业务验收
