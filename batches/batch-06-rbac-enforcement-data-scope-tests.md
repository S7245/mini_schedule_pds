# Batch 06 测试场景 — RBAC enforcement + data_scope 落地

执行方式：
- 单元测试：`go test ./internal/application/rbac/... ./internal/domain/rbac/... ./internal/application/{staff,location,onboarding,brandprofile}/...`
- 集成测试：`go test ./internal/infrastructure/persistence/rbac_repository_test.go ./internal/application/rbac/checker_integration_test.go` 含真实 Redis
- 端到端：curl + 浏览器手测 + browser-automation Playwright

前置条件：
- backend dev 含 Batch 6 全部 commit
- Redis 在 localhost:6379 运行
- brand_id=21、owner brand_user_id=16（手机号 18816820405 / 密码 test1234）
- 测试号 brand_user_id=18（张三 / 13900139001 / admin123）— location_manager @ Location1，19 个 effective permissions

---

## Happy Path

| # | 步骤 | 预期结果 |
|---|---|---|
| H1 | 张三登录 → GET /me/permissions | 200；`{ permissions: [...19个], data_scope: {kind:'assigned_locations', location_ids:[1]} }` |
| H2 | 张三 GET /staff | 200；只返回任职在 Location1 的 staff（含他自己；owner 如未任职 L1 不显示） |
| H3 | 张三 PATCH /staff/18/instructor 编自己教练档案 | 200；instructor.edit 通过 + 自己在 Location1 通过 data_scope |
| H4 | owner（16）登录 → GET /me/permissions | 200；`{ permissions: [所有 brand_owner 全权限], data_scope: {kind:'all_brand'} }`；location_ids 不出现 |
| H5 | owner GET /staff | 200；返全 brand 内所有 active staff |
| H6 | owner POST /staff | 201；staff.create 通过 |

---

## Edge Cases — 权限不足（PERMISSION_DENIED）

| # | 场景 | 预期 |
|---|---|---|
| E1 | 张三 POST /staff（无 staff.create） | 403 `PERMISSION_DENIED` + Details `{required:"staff.create", missing:["staff.create"]}` |
| E2 | 张三 PATCH /brand/profile（无 brand.profile.edit） | 403 PERMISSION_DENIED |
| E3 | 张三 POST /locations（无 location.create） | 403 |
| E4 | 张三 DELETE /staff/18 自删（无 staff.delete） | 403 |
| E5 | 张三 PATCH /staff/18/role-assignments（无 staff.assign_role） | 403 |
| E6 | 张三 DELETE /locations/1（无 location.delete） | 403 |
| E7 | 张三 POST /onboarding/complete（无 brand.profile.edit） | 403 |
| E8 | 张三 PATCH /onboarding/steps/staff/skip（无 brand.profile.edit） | 403 |

---

## Edge Cases — data_scope 收紧

| # | 场景 | 预期 |
|---|---|---|
| E9 | 张三 GET /staff/16（owner，假设 owner 未任职 Location1） | 404 `STAFF_NOT_FOUND`（data_scope 让他看不到 owner） |
| E10 | 张三 GET /staff/18（自己） | 200 |
| E11 | 张三 GET /staff 列表 | items 都至少有 1 个 Location1 任职关系 |
| E12 | 张三 PATCH /staff/{别人}/location-assignments（有 staff.assign_location 但目标不在 Location1） | 404 STAFF_NOT_FOUND |
| E13 | 张三 GET /locations | 只返 Location1（assigned_locations 收紧） |
| E14 | 张三 GET /locations/1 | 200 |
| E15 | 张三 GET /locations/{Location2 id}（如有 Location2） | 404 LOCATION_NOT_FOUND |

---

## Edge Cases — 自服务接口（无需权限）

| # | 场景 | 预期 |
|---|---|---|
| E16 | 张三 GET /me/permissions | 200（自服务，永远允许） |
| E17 | 张三 GET /onboarding/status | 200（读状态，所有员工可见） |
| E18 | 张三 GET /brand/profile | 200（brand.profile.view 张三 implied 有，因为他有 location.view? 不，view 是独立的；张三**没有** brand.profile.view → 403）— 修正：profile.view 是 brand 域，张三确实没有 → **403 expected** |

> ⚠️ E18 修正后表现 403。这是合理的：店长看品牌资料是合理需求，但 seed 阶段没给他。验收时如果产品认为店长应该能看 → 调 Batch 5 migration 给 location_manager 加 brand.profile.view；否则保持 403。**本批以"按 seed 数据严格执行"为准**。

---

## Edge Cases — Permission 隐含

| # | 场景 | 预期 |
|---|---|---|
| E19 | 张三的 effective_permissions 中应有 instructor.view（来自 instructor.edit 隐含） | GET /me/permissions 含 instructor.view |
| E20 | 张三的 effective_permissions 中应有 location.view（直接） + 是否隐含 location.create? 否（create 不来自 edit） | location.create 不在 effective set |
| E21 | 自定义场景：手工给某测试用户分配 staff.delete（但不给 staff.view / staff.edit）| effective 应有 staff.delete + staff.edit + staff.view（delete 隐含两者） |

---

## Edge Cases — Redis 缓存

| # | 场景 | 预期 |
|---|---|---|
| E22 | 张三第一次 GET /me/permissions | DB 查询发生（日志 GORM trace）+ Redis SET key=`rbac:perms:18` TTL=60s |
| E23 | 立即第二次 GET /me/permissions | Redis HIT；DB 无 GORM trace |
| E24 | 同一请求里多次内部 RequirePermission 调用 | 第一次查 cache，后续走 in-memory（context 内复用） |
| E25 | 60s 后第三次 GET /me/permissions | Redis MISS + DB 重查 + SET |
| E26 | admin 把张三的 location_manager assignment status 改 inactive | 60s 内张三仍有原权限（accept stale）；60s+ 后 DB 重查发现 0 active assignment → 403 拒所有需权限的接口 |
| E27 | Redis 不可用（关掉 redis） | Checker fallback 到每请求 DB 查 + log warning；不阻塞功能 |

---

## Edge Cases — Owner 特权

| # | 场景 | 预期 |
|---|---|---|
| E28 | brand_user.is_owner=true 用户 effective_permissions | 含所有 permission code（不论 role_assignment 怎么配） |
| E29 | owner 没有任何 role_assignment（异常数据） | 仍按 owner 给全权（兜底） |
| E30 | owner GET /me/permissions | kind='all_brand'，location_ids 字段 omit |

---

## Edge Cases — 跨 brand 隔离

| # | 场景 | 预期 |
|---|---|---|
| E31 | 张三 GET /staff/{别 brand 的 staff_id} | 404 STAFF_NOT_FOUND（跨 brand 屏蔽 — 已是 Batch 5 行为，回归确认不破） |
| E32 | 张三 GET /locations/{别 brand 的 location_id} | 404 LOCATION_NOT_FOUND |

---

## Edge Cases — Service 层回填回归

| # | 场景 | 预期 |
|---|---|---|
| E33 | Batch 5 所有 staff 测试（含 owner 保护、quota、PUT 全量替换）以 owner 身份重跑 | 全过 |
| E34 | Batch 4 所有 onboarding 测试以 owner 身份重跑 | 全过 |
| E35 | Batch 4 所有 location 测试以 owner 身份重跑 | 全过 |

---

## 执行方式

**单元测试**：
- `internal/application/rbac/checker_test.go`：覆盖 H1/H4 + E1-E10 + E19-E21（mock repository + fake cache）
- `internal/application/rbac/scope_resolver_test.go`：覆盖 data_scope 推导矩阵 + 多 role assignment 合并
- 各 service 的 *_test.go：加 mock checker 测 RequirePermission 通过 / 拒绝两路径

**集成测试**：
- `internal/application/rbac/checker_integration_test.go`：用真实 Redis 测 E22-E27
- `internal/infrastructure/persistence/rbac_repository_test.go`：JOIN 三表的实际 SQL 行为

**端到端（curl + 浏览器）**：

```bash
# 张三登录
TOKEN=$(curl -s -X POST http://localhost:8081/api/v1/brand/login \
  -H "Content-Type: application/json" \
  -d '{"phone":"13900139001","password":"admin123"}' | jq -r .data.access_token)

# H1: 看自己权限
curl -s http://localhost:8081/api/v1/brand/me/permissions \
  -H "Authorization: Bearer $TOKEN" | jq

# H2: 看可见员工列表（限 Location1）
curl -s http://localhost:8081/api/v1/brand/staff \
  -H "Authorization: Bearer $TOKEN" | jq '.data.items[].id'

# E1: 尝试创建员工 → 403
curl -s -X POST http://localhost:8081/api/v1/brand/staff \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"phone":"13900139999","name":"测试","initial_password":"test1234"}' | jq
# 期望：{"code":"PERMISSION_DENIED","message":"...","data":{"required":"staff.create","missing":["staff.create"]}}

# E2: 尝试改品牌资料 → 403
curl -s -X PATCH http://localhost:8081/api/v1/brand/profile \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"description":"hack"}' | jq

# E5: 尝试改自己角色 → 403
curl -s -X PUT http://localhost:8081/api/v1/brand/staff/18/role-assignments \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"assignments":[]}' | jq

# E10: 看自己详情 → 200
curl -s http://localhost:8081/api/v1/brand/staff/18 \
  -H "Authorization: Bearer $TOKEN" | jq
```

**浏览器手测**：
- 张三登录后菜单：员工管理可见、门店管理隐藏（无 location.create / location.delete 不足以隐藏整个菜单，但创建按钮 disabled + tooltip）
- 员工详情页：删除按钮 disabled + tooltip "权限不足"
- 用 owner 登录对比：所有菜单 / 按钮都正常
