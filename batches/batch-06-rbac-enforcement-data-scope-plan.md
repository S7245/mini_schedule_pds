# Batch 06 Plan — task DAG + 红绿步骤 + 风险预案

阶段：**plan**（spec 已 approve，TDD 启动前）
执行手册给主线程 + subagent 看。

---

## 后端 task DAG（10 个 task commit）

```text
T01 domain/rbac types ────┐
                           │
T02 repository interface──┤
                           │
T03 rbac_repository ───────┴── 数据访问层就绪
                                │
T04 application/rbac/checker ───┤  ← 注入 cache + repository，单测覆盖 implies/合并
                                │
T05 scope_resolver ──────────────┤  ← 单测覆盖 data_scope 推导矩阵
                                │
T06a staff service 回填 ──────┐ │
T06b location service 回填 ────┼─┤  ← 这三个并联：每个 service 加 checker 参数 + 头部 RequirePermission
T06c brandprofile + onboarding┘ │
                                │
T07 service 层 data_scope 收紧 ──┤  ← staff_repository.List + GetWithAssignments、location_repository.List + Get 加 scope WHERE
                                │
T08 interfaces：me_handler + roles 扩展 ─┤
                                          │
T09 错误码 + Wire 重生 ─────────────────┤
                                          │
T10 静态验证 gate ─────────────────────┘
```

---

## 后端 task 细节

### T01 — domain/rbac types

**绿**：`internal/domain/rbac/permission.go`：
```go
type PermissionSet map[string]struct{}
func (s PermissionSet) Has(code string) bool
func (s PermissionSet) HasAll(codes ...string) bool
// Expand 内存计算 implies：
//   edit/create → +view
//   delete → +view, +edit
// 不修改入参，返新集合
func Expand(raw []string) PermissionSet

type DataScopeKind string
const (DataScopeAllBrand, DataScopeAssignedLocations, DataScopeNone DataScopeKind = ...)

type DataScope struct {
    Kind        DataScopeKind
    LocationIDs []int64  // 仅 AssignedLocations 时填
}
// Merge 多个 assignment 的 scope（并集；任一 all_brand 整体 all_brand）
func MergeScopes(scopes []DataScope) DataScope
```

**单测**：覆盖 Expand 4 个隐含规则；MergeScopes 边界（empty、单 brand、多 location、混合）。

**commit**：`feat(batch-6-be): domain rbac types — PermissionSet + DataScope`

### T02 — domain/rbac repository interface

```go
type Repository interface {
    // LoadEffectiveRaw 一次 SQL JOIN 三表拿 raw permission codes + 各 role assignment 的 (scope, location_id)
    LoadEffectiveRaw(ctx context.Context, brandID, brandUserID int64) (rawCodes []string, scope DataScope, isOwner bool, err error)
}
```

**commit**：`feat(batch-6-be): rbac domain repository interface`

### T03 — rbac_repository 实现

`internal/infrastructure/persistence/rbac_repository.go`：单条 SQL：

```sql
SELECT bu.is_owner,
       bura.role_id, br.scope_type, bura.location_id, bura.data_scope,
       p.code
FROM brand_users bu
LEFT JOIN brand_user_role_assignments bura
       ON bura.brand_user_id = bu.id
      AND bura.status = 'active'
LEFT JOIN brand_roles br ON br.id = bura.role_id AND br.status = 'active'
LEFT JOIN brand_role_permissions brp ON brp.role_id = bura.role_id
LEFT JOIN permissions p ON p.id = brp.permission_id AND p.status = 'active'
WHERE bu.id = ? AND bu.brand_id = ? AND bu.deleted_at IS NULL
```

逻辑：
- is_owner=true → 返 isOwner=true，rawCodes 给空（Checker 上层用所有 code）
- 多行聚合：codes set 化；location_ids 收集成 set；推导 scope（per spec）
- 没有任何 active assignment 且非 owner → DataScopeNone

**集成测试**：用真实 PG，张三 brand_user_id=18 验证拿到 location_manager 的权限码 + scope=AssignedLocations + location_ids=[1]。

**commit**：`feat(batch-6-be): rbac_repository — load effective raw + scope derivation`

### T04 — application/rbac/checker

`internal/application/rbac/checker.go`：
```go
type Checker struct {
    repo  rbac.Repository
    cache cacheStore // 接口，可注入 Redis 真实实现或 in-mem fake
    allPerms []string  // 启动时从 permissions 表 load 一次，用于 owner 兜底
}

func (c *Checker) Resolve(ctx, brandID, brandUserID) (*PermissionSet, *DataScope, error) {
    // 1) cache GET rbac:perms:{brandUserID}
    // 2) MISS → repo.LoadEffectiveRaw
    //    if isOwner → permissions = Expand(c.allPerms), scope = AllBrand
    //    else → permissions = Expand(rawCodes), scope = scope
    // 3) cache SET TTL 60s（JSON 编码 [codes, scope]）
    // 4) 返
}

func (c *Checker) Require(ctx, brandID, brandUserID, code) error {
    perms, _, err := c.Resolve(...)
    if err != nil { return err }
    if !perms.Has(code) {
        return apperr.NewAppError(ErrPermissionDenied, "权限不足", 403).
            WithDetails(map[string]any{"required": code, "missing": []string{code}})
    }
    return nil
}

// RequireScope 校验 target 是否在 scope 内（按 location_id）
func (c *Checker) RequireScope(ctx, brandID, brandUserID, kind, targetLocationIDs []int64) error
```

**单测**：mock repo + in-mem cache，覆盖 H1 / H4 / E1 / E19 / E22-E26 / E28。

**风险**：Redis 不可用时 fallback — cacheStore 接口的 SET 失败仅 log warning，GET 失败回 nil 触发重查。

**commit**：`feat(batch-6-be): rbac.Checker with Redis cache + owner fast-path`

### T05 — application/rbac/scope_resolver

`internal/application/rbac/scope_resolver.go`：
```go
// ApplyToQuery 把 DataScope 转 GORM Where 子句。
// 调用方传 column name（如 "staff_location_assignments.location_id"），
// 返回 *gorm.DB（chained where）或 error。
// AllBrand → no-op
// AssignedLocations → WHERE col IN (?)
// None → WHERE 1=0（永远空）
func ApplyToQuery(q *gorm.DB, col string, scope DataScope) *gorm.DB
```

**单测**：mock gorm builder（用 `clause` interface），三种 kind 各一例。

**commit**：`feat(batch-6-be): rbac.ScopeResolver — DataScope to GORM where`

### T06a — staff service 回填 RequirePermission

每个 method 头部一行：
```go
func (s *Service) Create(ctx, in CreateInput) (*Staff, error) {
    if err := s.checker.Require(ctx, in.BrandID, in.ActorID, "staff.create"); err != nil {
        return nil, err
    }
    // ... 既有逻辑
}
```

涵盖 method（per spec 表）：List / GetByID / GetWithAssignments / Create / Update / UpdateStatus / Delete / ReplaceRoleAssignments / ReplaceLocationAssignments / GetInstructor / UpsertInstructor / DeleteInstructor / ListRoles。

**单测**：fake checker（permitted=true / Require 返 PermissionDenied 两路径）；mock checker 验证调用了正确 code。

**风险**：现有单测对 `NewService` 签名 + fake repo 都要改 — 加 mock checker 参数。

**commit**：`feat(batch-6-be): staff service — RequirePermission gates on every method`

### T06b — location service 回填

同 T06a 模式：List / GetByID / Create / Update / UpdateStatus / Delete。

**commit**：`feat(batch-6-be): location service — RequirePermission gates`

### T06c — brandprofile + onboarding service 回填

- brandprofile: Get（brand.profile.view）、Update（brand.profile.edit）
- onboarding: GetOnboardingStatus（**不需要权限** — 自服务）、SkipStep / Complete（brand.profile.edit）

**commit**：`feat(batch-6-be): brandprofile + onboarding services — RequirePermission gates`

### T07 — data_scope 收紧

修改 repository 层：
- `staff_repository.List` / `GetWithAssignments` 内查询前接受 scope filter 参数；List WHERE 子句 INNER JOIN staff_location_assignments 限制 location_id ∈ scope.location_ids；GetWithAssignments 拉完后检查任职 location 是否有交集，无则返 STAFF_NOT_FOUND
- `location_repository.List` / `GetByID` 直接按 location_id 过滤
- service 层在调 repo 前 `s.checker.Resolve` 拿 scope，传给 repo

**注意**：scope 检查要在数据 mutation 前完成（PATCH/DELETE 也要 GetByID 验证），不要漏。

**单测**：mock checker + scope 推导多场景；覆盖 E9-E15。

**风险**：可能破 Batch 5 测试（owner 之外的查询路径）— **必须**保留 Batch 5 全部测试通过。

**commit**：`feat(batch-6-be): repositories — data_scope filter on List / GetByID paths`

### T08 — me_handler + roles 扩展

新文件 `internal/interfaces/brand/me_handler.go`：
- GET /me/permissions → 调 checker.Resolve，序列化 PermissionSet 为 string[] + DataScope

扩 `roles_handler.go`：
- GET /roles/:code → 单角色详情（含 permissions 列表）
- GET /permissions → 列全部 permission（需 staff.assign_role）

**commit**：`feat(batch-6-be): /me/permissions + roles detail + permissions list endpoints`

### T09 — 错误码 + Wire 重生

- 加 `ErrPermissionDenied ErrorCode = "PERMISSION_DENIED"`（http.StatusForbidden）
- Wire：rbac.Checker 注入到 staff / location / brandprofile / onboarding service 的构造函数
- `~/go/bin/wire` 三个 cmd 都跑

**commit**：`chore(batch-6-be): error code + wire regen for rbac.Checker`

### T10 — 静态验证 gate

不单独 commit。最终：
- `go build ./...` + `go vet ./...` 干净
- `go test ./internal/{application,domain,infrastructure}/...` 全过
- Batch 5 既有测试（staff / location / onboarding / brandprofile）全过（关键回归）

---

## 前端 task DAG（5 个 task commit）

```text
F01 usePermissions hook + Context provider + 常量 ────┐
                                                       │
F02 工作台菜单按权限隐藏 ────────────────────────────┤
                                                       │
F03 详情页按钮 disabled + tooltip ───────────────────┤
                                                       │
F04 全局 403 兜底（client.ts onError 分支）─────────┤
                                                       │
F05 联调 H1-H4 happy path ──────────────────────────┘
```

### F01 — usePermissions hook

新文件 `apps/brand/lib/permissions.tsx`：
- `<PermissionsProvider>` Context wrapper，包在 (protected)/layout 内
- `usePermissions()` hook 返 `{ permissions: string[], dataScope: {...}, has: (code) => boolean, isLoading }`
- React Query 60s staleTime，refetch on window focus
- `PERMISSIONS` 常量对象（typing 安全）

**commit**：`feat(batch-6-fe): usePermissions hook + Context provider + constants`

### F02 — 工作台菜单按权限隐藏

修改 layout.tsx：每个菜单项加 `hidden={!has('staff.view')}` 类似条件。

- 员工管理：staff.view
- 门店管理（如有独立菜单）：location.view
- 品牌资料：brand.profile.view
- 工作台首页：无条件显示

**commit**：`feat(batch-6-fe): menu items hidden by permission`

### F03 — 按钮 disabled + tooltip

- /staff 列表"新增员工"：disabled if !has('staff.create')
- /staff/[id] 详情页"删除"：复用 Batch 5 的 disabled 逻辑，加 `|| !has('staff.delete')`
- StaffRoleAssignmentEditor 编辑：disabled if !has('staff.assign_role')
- /onboarding 跳过 / 完成按钮：disabled if !has('brand.profile.edit')
- LocationStatusToggle：disabled if !has('location.edit')

统一 tooltip 文案："权限不足，请联系管理员"。

**commit**：`feat(batch-6-fe): action buttons disabled by permission with tooltip`

### F04 — 全局 403 兜底

修改 `packages/api/src/client.ts`：
- onError 检测 PERMISSION_DENIED → sonner toast "权限不足: {required}"
- 不重定向、不 logout（用户仍登录，仅个别 action 受限）

**commit**：`feat(batch-6-fe): global PERMISSION_DENIED toast handler`

### F05 — 联调

不单独 commit，仅 final verify：
- 张三登录看菜单 + 各按钮 disabled 状态
- owner 登录对比

---

## subagent 提示词大纲

### 后端
- 必读：本 plan + 契约 + tests + backend/.learnings/{LEARNINGS,ERRORS,FEATURE_REQUESTS}.md
- 10 个 task 按 DAG 顺序逐 commit；T06 是大 task 但拆 a/b/c 三 commit
- 关键约束（per .learnings）：
  - Owner 必走 fast-path（is_owner=true 直接给所有 permissions）— 否则 owner 漏配角色会被自己锁死
  - Redis 不可用时 fallback DB + log warning，**不**阻塞功能
  - service 层回填**必须**保 Batch 5 测试全过（每改一个 service 就跑该 service 的 *_test.go）
  - Wire 重生必须干净（无 TODO 注释，per Batch 3 ERRORS）
- 完成报告：commit SHA + 覆盖测试场景编号 + 留 TODO + Wire 状态

### 前端
- 必读：本 plan + 契约 + tests + web/.learnings/{LEARNINGS,ERRORS,FEATURE_REQUESTS}.md
- 5 个 task 顺序 commit；每个 task 前 `pnpm --filter @mini-schedule/brand exec tsc --noEmit` 干净
- 关键约束：
  - usePermissions 失败时（如登录但 /me/permissions 返 500）默认给空 PermissionSet → 所有按钮 disabled（fail-closed）
  - 菜单隐藏 + 按钮 disabled 的策略要一致：菜单隐藏意味着这一类操作全无；按钮 disabled 意味着部分操作可见但不可执行
  - 403 toast 不要 spam — 短时间内同 code 去重

---

## 风险点 + 预案

1. **service 层回填破 Batch 5 测试**：T06 改 service 构造函数会破现有测试 setUp。预案：每个 task 完成立刻跑该 service 的全部 *_test.go；T10 最后再跑一遍全套。
2. **Owner 兜底逻辑**：is_owner=true 路径必须独立于 role_assignments，否则配置事故时 owner 自锁。T04 单测 E28 / E29 强制覆盖。
3. **Redis fallback**：cacheStore 接口必须真的允许 Redis 不可用时回 DB；T04 加 mock cacheStore 测 E27。
4. **scope 推导 corner case**：多个 location-scope role 任职不同 location → 并集；任一 brand-scope → 整体 all_brand。T01 + T05 单测覆盖。
5. **/me/permissions 性能**：每次请求都会调 RequirePermission（多次），实测 Resolve 必须 in-context-cache（同请求内复用），避免 1 个 HTTP 请求触发 5 次 Redis GET。T04 实现要在 ctx 里挂 once.Do 模式。
6. **前端 fail-closed**：usePermissions 加载失败默认所有按钮 disabled，不要 default 给空数组 + 假装权限正常 — 前端表现要保守。

---

## Stage checkpoint：plan 阶段产出确认

待用户过目本 plan 后回 OK。OK 后我会 spawn 后端 + 前端 subagent。
