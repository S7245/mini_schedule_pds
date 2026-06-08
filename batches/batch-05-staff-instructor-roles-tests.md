# Batch 05 测试场景 — Staff + InstructorProfile + 角色 seed + SubscriptionGuard 重构

执行方式：
- 单元测试：`go test ./internal/application/{staff,commercial}/... ./internal/audit/... ./internal/domain/{staff,role,instructor}/...`
- 集成测试（含事务 + 并发 quota）：`go test ./internal/infrastructure/persistence/... -tags=integration`
- 端到端：browser-automation skill + 手动 curl

前置条件：
- backend dev 含 Batch 5 全部 commit
- 数据库 migration 跑到 000005
- 已有测试品牌 id=21，owner brand_user 18816820405（Batch 3 验收数据）
- 已跑过 admin 端 backfill 接口，owner 已分配品牌负责人角色

---

## Happy Path — 端到端

| # | 步骤 | 预期结果 |
|---|---|---|
| H1 | 登录 owner → 工作台菜单可见"员工管理" | 菜单项渲染（角色管理不展示，per 决定 8） |
| H2 | 点员工管理 → /staff 列表 | 显示 1 行：owner 自己（手机号 18816820405，角色"品牌负责人"，状态 active，无教练标识） |
| H3 | 点"新增员工" → 填 phone=`13900139001`, name=`张三`, password=`test1234`, 角色多选"店长", Location 任职 Location1 `manager` `is_primary=true` → 提交 | 201；列表新增一行 |
| H4 | 进 /staff/2 详情页 | 三段：基础信息（手机号、姓名、状态、is_owner=false） / 角色任职（店长 @ Location1, data_scope=role_default） / 教练档案折叠卡（"晋升为教练"按钮） |
| H5 | 点"晋升为教练" → 折叠卡展开 → 填 display_name=`张教练`, bio, specialties, is_visible_to_learners=true, is_schedulable=true → 保存 | PUT /staff/2/instructor 200；列表回到 /staff 后该行出现教练标识 |
| H6 | 进 onboarding → 看第 3 步状态 | step `staff` 显示 completed（GetCounts 算 brand_users active≥1 AND instructor_profiles≥1） |
| H7 | 进 onboarding 第 3 步页面 | 显示"已完成，可继续下一步" + 跳转按钮（不强制再操作） |

---

## Edge Cases — Staff CRUD / Quota

| # | 场景 | 预期 |
|---|---|---|
| E1 | 已建满 max_staff_seats（plan id=2 默认 20）后再 POST | 409 `QUOTA_EXCEEDED` + Response.Data `{current: 20, max: 20}` + 前端 "席位已达套餐上限 20/20" |
| E2 | POST 手机号 `18816820405`（已被 owner 占用） | 409 `STAFF_PHONE_DUPLICATED` + inline 错误 |
| E3 | POST 手机号被另一个 brand 已使用 | 409 `STAFF_PHONE_DUPLICATED`（brand_users.phone 唯一索引是跨品牌的，Batch 1 注册同款逻辑） |
| E4 | initial_password 不符合复杂度（< 8 位 / 仅字母 / 仅数字） | 400 `INVALID_PARAM` + 前端 RHF 校验 |
| E5 | POST 不带 role_codes / location_assignments | 201 创建成功（皆可选），列表显示无角色无任职 |
| E6 | DELETE 唯一 owner staff_id=1 | 409 `OWNER_PROTECTED` |
| E7 | PATCH 唯一 owner status=inactive | 409 `OWNER_PROTECTED` |
| E8 | DELETE 非 owner staff | 204；列表过滤掉；OperationLog `staff_deleted` |
| E9 | DELETE 跨 brand 的 staff_id | 404 `STAFF_NOT_FOUND`（跨 brand 隔离） |
| E10 | 并发 POST staff 撞 max（2 个请求并行，max=20、current=19） | 一成一败（一个 201、一个 409 QUOTA_EXCEEDED） |
| E11 | Subscription frozen 时 POST staff | 403 `SUBSCRIPTION_RESTRICTED` |
| E12 | Subscription `expires_at` 已过、`status='active'` 没刷 | 同 E11 |

---

## Edge Cases — 角色 / Location 任职

| # | 场景 | 预期 |
|---|---|---|
| E13 | PUT role-assignments 含 role_code=`不存在` | 404 `ROLE_NOT_FOUND` |
| E14 | PUT role-assignments 含 role_code=`brand_owner`（非 owner staff 尝试拿 owner 权限） | 400 `INVALID_PARAM`（owner 角色不可手动分配，只能 backfill / 注册自动） |
| E15 | PUT location-assignments 含 location_id 不属于本 brand | 400 `LOCATION_ASSIGNMENT_INVALID` |
| E16 | PUT location-assignments 含已软删的 location_id | 400 `LOCATION_ASSIGNMENT_INVALID` |
| E17 | PUT location-assignments 多个 is_primary=true | 400 `INVALID_PARAM`（最多 1 个 primary） |
| E18 | PUT role-assignments 含 location 级角色但 location_id=NULL | 400 `INVALID_PARAM` |
| E19 | PUT role-assignments 含 brand 级角色但 location_id 非 NULL | 400 `INVALID_PARAM` |
| E20 | PUT role-assignments 含 inactive brand_role | 400 `INVALID_PARAM` |
| E21 | PUT role-assignments 空数组 | 200 替换为空（清空角色） |

---

## Edge Cases — InstructorProfile

| # | 场景 | 预期 |
|---|---|---|
| E22 | PUT instructor 重复（同 brand_user_id） | 200 update（1:1 upsert）；OperationLog `instructor_profile_upserted` |
| E23 | PUT instructor display_name 为空 | 400 `INVALID_PARAM` |
| E24 | DELETE instructor 后 GET 教练列表 | 不在；GetCounts 的 `staff` step status 重新计算（可能跌回 not_started） |
| E25 | DELETE instructor 不影响 staff 本身 | staff 详情仍可见，"晋升为教练"按钮恢复 |
| E26 | 跨 brand 操作 staff 的 instructor | 404 `STAFF_NOT_FOUND` |

---

## Edge Cases — Backfill / 注册流程改造

| # | 场景 | 预期 |
|---|---|---|
| E27 | 跑 POST /admin/system/backfill-owner-roles 第一次 | `{processed: 1, skipped: 0, failed: 0}`（brand_id=21 处理完） |
| E28 | 第二次跑 backfill | `{processed: 0, skipped: 1, failed: 0}`（幂等） |
| E29 | 全新注册流程：跑 Batch 1 / 2 / 3 注册新 brand → 完成支付 → 登录 | owner 自动有"品牌负责人"角色（无需手动跑 backfill） |
| E30 | backfill 接口未带 admin JWT | 401 / 403 |

---

## Edge Cases — Audit pkg / OperationLog

| # | 场景 | 预期 |
|---|---|---|
| E31 | POST staff 成功后查 operation_logs | 新增一行 action=`staff_created`, actor_type=`brand_user`, target_type=`staff`, metadata 含创建后的 staff 摘要 |
| E32 | PUT role-assignments 后查 logs | action=`staff_role_assignments_changed`, metadata 含 before / after 数组 |
| E33 | Location 创建（Batch 4 既有） | action 仍为 `location_created`（audit pkg 改造后行为不变） |
| E34 | 跨 brand 的 actor_id 与 brand_id 不一致 | 不应发生（service 层校验 brand context） |

---

## Edge Cases — SubscriptionGuard 重构回归

确保 Batch 4 的 Location quota 行为不变：

| # | 场景 | 预期 |
|---|---|---|
| E35 | Batch 4 H5 重跑：POST location 第 3 个 | 201 |
| E36 | Batch 4 E12 重跑：POST 第 4 个 location | 409 QUOTA_EXCEEDED + Details `{current: 3, max: 3}` |
| E37 | Batch 4 E13 重跑：Subscription frozen 时 POST location | 403 SUBSCRIPTION_RESTRICTED |
| E38 | Batch 4 E14 重跑：同名 active 已存在 | 409 LOCATION_NAME_DUPLICATED（与 quota 无关）|

---

## 执行方式

**单元测试**：
- `internal/application/staff/service_test.go`：覆盖 H3, H5, E1–E10, E13–E25
- `internal/application/commercial/subscription_guard_test.go`：覆盖 quota 边界（多 ResourceKind 走同条规则）
- `internal/application/staff/role_allocator_test.go`：覆盖 backfill 幂等 + 注册流程改造 + E27/E28/E29
- `internal/audit/audit_test.go`：覆盖 metadata 序列化 + actor type 枚举 + E31–E34

**集成测试**（GORM 事务 + 真实 PG）：
- `internal/infrastructure/persistence/staff_repository_integration_test.go`：E10 并发 staff quota
- 复用 Batch 4 既有 location_repository_integration_test.go：E35–E38 回归

**端到端**（browser-automation Playwright）：
- 完整 Happy Path H1–H7
- /staff 列表渲染 + 创建 Dialog 字段校验（E4 / E14 / E17）
- 详情页"晋升为教练"折叠卡（H5 + E23）

**手动 curl**：

```bash
# 拿 owner token
TOKEN=$(curl -s -X POST http://localhost:8081/api/v1/brand/login \
  -H "Content-Type: application/json" \
  -d '{"phone":"18816820405","password":"test1234"}' | jq -r .data.token)

# 跑 backfill（如未跑过）—— 这里假设 admin 接口
ADMIN_TOKEN=$(...)  # 通过 admin login 拿
curl -s -X POST http://localhost:8083/api/v1/admin/system/backfill-owner-roles \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq

# 看 owner 角色
curl -s http://localhost:8081/api/v1/brand/staff/1 \
  -H "Authorization: Bearer $TOKEN" | jq .data.role_assignments

# 添加员工
curl -s -X POST http://localhost:8081/api/v1/brand/staff \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "13900139001",
    "name": "张三",
    "initial_password": "test1234",
    "role_codes": ["location_manager"],
    "location_assignments": [
      {"location_id": 1, "assignment_type": "manager", "is_primary": true}
    ]
  }' | jq

# 晋升新员工为教练（拿到 staff id 后）
STAFF_ID=2
curl -s -X PUT http://localhost:8081/api/v1/brand/staff/$STAFF_ID/instructor \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "display_name": "张教练",
    "bio": "10 年瑜伽经验",
    "is_visible_to_learners": true,
    "is_schedulable": true,
    "status": "active"
  }' | jq

# 查向导
curl -s http://localhost:8081/api/v1/brand/onboarding/status \
  -H "Authorization: Bearer $TOKEN" | jq '.data.steps[] | select(.step_key == "staff")'
# 期望：status=completed, count >= 1, target=1
```
