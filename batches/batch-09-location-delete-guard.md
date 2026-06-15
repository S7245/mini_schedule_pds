# Batch 9 契约 — 门店删除引用保护（LOCATION_IN_USE）

> 状态：**契约已 approve**（2026-06-12，会话内确认）。
> 背景：Batch 8 验收转 FR。门店删除现状是无条件软删（UPDATE deleted_at），**软删不触发 FK 行为**，留下指向已删门店的悬空引用。镜像 Batch 7 A4（ROLE_IN_USE）补引用保护。

---

## 0. 事实基础（survey 已确认）

- `location_repository.SoftDelete`（`location_repository.go:199`）软删 + 写 audit，**无引用检查**。
- 12 张表带 location_id，但**当前 build 阶段只有两张真实有数据**：
  - `staff_location_assignments`（员工↔门店任职，Batch 5；status active/inactive，但 `ReplaceLocationAssignments` 硬删旧行重插，**实际只存在 active 行**）
  - `brand_user_role_assignments.location_id`（门店级角色任职，Batch 5/7；同样硬删重插，只存 active）
  - class_sessions / recurring_schedules（FK RESTRICT）等课程/场次表 schema 已建但**无 CRUD、无数据**（未来批次）。
- Batch 7 A4 模式可镜像：service 层 `Count...` > 0 → `apperr.ErrXxxInUse` 409。
- 错误码 `LOCATION_IN_USE` 后端/前端均**不存在**，需新增。

## 0.1 grill 决策（已与用户确认 2026-06-12）

| 决策 | 定论 |
|---|---|
| 哪些引用阻止删除 | **active 员工任职（staff_location_assignments）+ active 门店级角色任职（brand_user_role_assignments.location_id）**。两者都是当前真实会悬空的 active 引用。 |
| 是否前向检查 class_sessions/recurring_schedules | **否**（现无 CRUD/无数据，属空检查，违反"先做抽象再找落点"）。等课程/场次批次落地时再纳入本 guard。 |
| active vs all 状态 | **active**（两表都硬删重插，实际只有 active 行；filter active 语义直白）。 |

---

## 1. 契约

### API 接口
**无新增路由**，改既有 `DELETE /api/v1/brand/locations/:id`（gate location.delete）行为：

| 方法 | 路径 | 变化 |
|---|---|---|
| DELETE | `/api/v1/brand/locations/:id` | 软删前先查引用：有 active 员工任职或门店级角色任职 → **409 `LOCATION_IN_USE`**（不再软删）；无引用 → 同现状软删 204 |

### 后端改动

1. **错误码**（`pkg/errors/error.go`）：新增 `ErrLocationInUse ErrorCode = "LOCATION_IN_USE"`（HTTP 409）。
2. **repo**（`location_repository.go`）：新增
   ```go
   // CountActiveReferences 统计阻止删除的 active 引用：员工任职 + 门店级角色任职。
   func (r *locationRepository) CountActiveReferences(ctx, brandID, locationID int64) (int64, error)
   ```
   = `COUNT staff_location_assignments WHERE location_id=? AND brand_id=? AND status='active'`
   + `COUNT brand_user_role_assignments WHERE location_id=? AND brand_id=? AND status='active'`（两次 COUNT 求和，或 UNION ALL；带 brand_id 防跨租户）。接口加到 `location.Repository`。
3. **service**（`location/service.go` `Delete`）：在 `guardLocationInScope` 之后、`SoftDelete` 之前插入：
   ```go
   n, err := s.repo.CountActiveReferences(ctx, brandID, id)
   if err != nil { return err }
   if n > 0 { return apperr.NewAppError(apperr.ErrLocationInUse, "该门店仍有员工任职或角色绑定，请先移除后再删除", 409) }
   ```
   require(location.delete) + scope guard 不变。

### 前端改动

1. `packages/api/src/errors.ts`：`ErrorCodes` 加 `LOCATION_IN_USE: 'LOCATION_IN_USE'`。
2. `apps/brand/app/(protected)/locations/page.tsx` `confirmDelete`：加 `case ErrorCodes.LOCATION_IN_USE` → `toast.error('该门店仍有员工任职或角色绑定，请先移除后再删除')`；**确认弹窗保持打开**（重试无意义直到移除引用，与 ROLE_IN_USE 一致，不调 `setDeleteTarget(null)`）。

### 约束
- 无新增路由/迁移；不动 location 其它行为（list/get/create/update/status 不变）。
- 后端遵守 GO_BACKEND_LANGUAGE_DESIGN（service 校验、repo 查询、传 ctx）。
- 前端复用既有 confirmDelete 结构，不改组件。

---

## 2. 测试场景

### 后端单测（DB-backed，mirror Batch 7 role-in-use；无 PG 自动 skip）
- BE-1：门店有 1 条 active staff_location_assignments → `service.Delete` 返 `ErrLocationInUse`，门店未被软删。
- BE-2：门店有 1 条 active brand_user_role_assignments(location_id=该店) → 同样 `ErrLocationInUse`。
- BE-3：门店无任何引用 → `Delete` 成功（软删）。
- BE-4（repo）：`CountActiveReferences` 对混合引用返正确合计；跨 brand 的同 id 引用不计入（brand_id 隔离）。

### e2e（`web/e2e/batch-09-location-delete-guard.spec.ts`，真实栈，沿用 Batch 8 约定）
- 前置：owner 登录。用 `page.request` 直接调后端 API 做 setup（建唯一名门店 → 建一个唯一手机号员工并分配该门店）。
- G1：UI 上删该门店 → toast「该门店仍有员工任职或角色绑定，请先移除后再删除」，确认弹窗保持打开、行仍在。
- G2：移除该员工的门店任职（API 或 UI）后，再删门店 → 成功消失。
- teardown：清理创建的员工 + 门店（软删/移除）。

---

## 3. 验收闭环

owner 进 `/locations` → 选一个有员工任职的门店点删除 → **409 + toast 提示先移除任职，弹窗不关、门店还在** → 到员工详情移除该门店任职 → 回来再删 → 成功。后端 `go test ./...` 绿；e2e `batch-09-*` 绿。

---

## 4. 任务拆分（TDD 逐 task commit）

**后端**
- T01 `pkg/errors` 加 `LOCATION_IN_USE`
- T02 repo `CountActiveReferences` + 接口声明 + DB-backed 单测（BE-4）
- T03 service `Delete` 插引用 guard + 单测（BE-1/2/3）

**前端**
- T04 errors.ts 加 `LOCATION_IN_USE` + page.tsx confirmDelete 加 case（弹窗保持打开）

**e2e**
- T05 `batch-09-location-delete-guard.spec.ts`（G1/G2，API setup + UI 断言 + teardown）

**转 FR（不做）**
- 课程/场次批次落地后，把 class_sessions/recurring_schedules 纳入 `CountActiveReferences`。
