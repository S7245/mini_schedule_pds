# Batch 10 契约 — 小债清扫（4 项）

> 状态：**契约已 approve**（2026-06-12，会话内确认）。
> 性质：技术债清扫批，4 个独立小项打包。来源：Batch 7/8/9 验收期转入的 FR。

---

## 0. 范围（用户 2026-06-12 确认四项全做）

| # | 项 | 类型 | 风险 |
|---|---|---|---|
| 1 | app/admin 水合竞态推广 | 前端 | 低（复用现成 useAuthHydrated） |
| 2 | 后端 RBAC 质量清理 | 后端 | 中（动 RBAC 代码，但无行为变化） |
| 3 | location list name 搜索 | 后端+前端 | 低（加 q 参数 + 搜索框） |
| 4 | 店长/只读权限门 e2e 用例 | e2e | 低（补回归覆盖） |

---

## 1. 各项契约

### 项 1：app/admin 水合竞态推广（前端）

Batch 8 给 brand 加了 SSR-safe `useAuthHydrated()`（`@mini-schedule/api/auth` 已导出）。app/admin 两端 protected 守卫有**同款竞态**（hard load 时 persist 未水合 → `isAuthenticated` 瞬时 false → 误跳 /login）。复用同 hook 修复：

- `apps/app/app/(protected)/layout.tsx`：现 `if (!isAuthenticated) return null` + 跳转 effect 无水合等待。加 `const authHydrated = useAuthHydrated()`，跳转 effect 开头 `if (!authHydrated) return`，`return null` 改 `if (!authHydrated || !isAuthenticated) return null`，effect deps 加 authHydrated。
- `apps/admin/components/layout/admin-guard.tsx`（admin 的守卫组件）：同样改造（gate redirect effect + `return null` 等 authHydrated）。

**无行为变化**（仅消除 hard-load 误跳），soft nav 不受影响。

### 项 2：后端 RBAC 质量清理（后端，无行为变化）

Batch 7 code-review 转移项，纯重构/优化：

- **GetRole 双查 → 单查**：`staff/service.go GetRole` 现先 `GetBrandRoleByCode` 再 `ListBrandRoles(brandID)` 全量扫出单条 permissions。改：给 `role.Repository` 加 `GetBrandRoleWithPermissions(ctx, brandID, code) (*BrandRole, error)`（impl 复用既有 unexported `getBrandRoleByIDWithPermissions` 逻辑，按 code→id 再取），`GetRole` 直接调它，省掉 O(N) 扫描。
- **角色缓存批量失效 pipeline**：`rbac.Checker` 加 `InvalidateMany(ctx, brandUserIDs []int64) error`（一次多 key Redis DEL / pipeline，而非逐 key 往返）；`staff/service.go invalidateRoleHolders` 改调 `InvalidateMany`。单用户 `Invalidate` 保留。
- **brand_owner/is_system 检查收敛**：`role.BrandRole` 加谓词 `IsOwnerRole() bool`（`Code == "brand_owner"`）。把散落的 `code == "brand_owner"` 字面量（`resolveRoleAssignments`、`ReplaceRoleAssignments`、`requireMutableCustomRole`）换成该谓词。**保持现有错误码优先级不变**：requireMutableCustomRole 仍是先 owner→`OWNER_PROTECTED` 再 is_system→`ROLE_IS_SYSTEM`。

约束：所有现有单测（Batch 7 role_crud_test / service_test / repo test）必须继续绿，不改断言（行为不变）。

### 项 3：location list name 搜索（后端+前端）

- **后端**：`location_handler.go list` 读 `q := c.Query("q")`；location 的 list filter struct 加 `Q string`；`location_repository.go`（list query 在 ~line 105）加 `if filter.Q != "" { q = q.Where("name ILIKE ?", "%"+filter.Q+"%") }`（Postgres ILIKE 大小写不敏感）。service 透传。data_scope/分页/status 不变。
- **前端**：`packages/api/src/locations.ts` `listLocations`/`useBrandLocations` 签名加可选 `q`；query key 含 q。`apps/brand/app/(protected)/locations/page.tsx` 顶部筛选区加搜索 `Input`（受控 state，输入即查或回车查；数据集小，可不防抖但 page 重置为 1）。空结果复用既有空状态。
- 单测：后端 repo q 过滤命中/不命中各一条（DB-backed，auto-skip 无 PG）。

### 项 4：只读权限门 e2e（e2e）

`web/e2e/batch-10-location-permission-gate.spec.ts`（真实栈，沿用约定）：
- 登录只读账号 **13900139777 / admin123**（brand 21，持 custom 前台兼职 = staff.view + location.view，**无 create/edit/delete**）。
- 进 `/locations`（有 location.view 故导航入口可见、列表可看）。
- 断言：「新建门店」按钮 disabled；任一行的「编辑」「删除」按钮 disabled（该账号 canCreate/canEdit/canDelete 全 false）。
- 只读断言，无写操作，无需 teardown。

> 注：location_manager（13900139001）有 edit 故「编辑/停用」会是 enabled —— 用只读账号 13900139777 才是干净的"全写禁用"断言。

---

## 2. 验收闭环
- 项1：app/admin 直达 deep-link 不再被弹去 /login（手动/或观察 e2e 稳定性）。
- 项2：`go test ./...` 全绿（行为不变，旧测试即回归）。
- 项3：/locations 搜索框输入门店名 → 列表过滤；后端 repo q 单测绿。
- 项4：`pnpm exec playwright test e2e/batch-10-location-permission-gate.spec.ts` 绿。

---

## 3. 任务拆分（TDD 逐 task commit）

**前端**
- T01 app/admin 水合竞态修复（app layout + AdminGuard 接 useAuthHydrated）
- T02 location 搜索：locations.ts 加 q + 页面搜索框

**后端**
- T03 RBAC：GetBrandRoleWithPermissions（GetRole 单查）
- T04 RBAC：Checker.InvalidateMany + invalidateRoleHolders 改用
- T05 RBAC：role.BrandRole.IsOwnerRole() 收敛 brand_owner 字面量
- T06 location list q 搜索（handler+filter+repo ILIKE）+ repo 单测

**e2e**
- T07 `batch-10-location-permission-gate.spec.ts`（只读账号写按钮全 disabled）

**约束**：各仓库只提交本批文件，不碰 web 树里并行未提交工作；backend/web 在 `dev`，pds 在 `main`。
