# Batch 04 测试场景 — 品牌初始化向导骨架 + Location 闭环

执行方式：
- 单元测试：`go test ./internal/application/onboarding/...` + `./internal/application/location/...`
- 集成测试（含事务 + quota 并发）：`go test ./internal/infrastructure/persistence/...`
- 端到端：browser-automation skill 跑 Playwright 脚本 + 手动 curl

前置条件：
- api-brand 在 localhost:8081 运行（含 Batch 4 新接口）
- brand 前端在 localhost:3002 运行（含 /onboarding 路由）
- 数据库迁移已跑到 000004（含 `brands.description` 字段）
- 测试用品牌 id=21 经过 Batch 1-3 流程：status=active、有一条 active subscription（plan id=2，max_locations=3）、用户手机号 18816820405 / 密码 `test1234`

mock / 测试约定：
- JWT 通过 `POST /api/v1/brand/login` 拿，前端守卫自动注入 Authorization 头
- 所有金额 / 限额数据沿用 Batch 1-3 已建数据

---

## Happy Path — 端到端

| # | 步骤 | 预期结果 |
|---|---|---|
| H1 | 访问 `/login`，输入 `18816820405 / test1234` 登录 | 跳转 `/onboarding`（首登检测到 onboarding_status ≠ completed） |
| H2 | onboarding 主页加载 | 显示 8 步进度条；当前 step=`brand_profile`；其他 step 状态为 `not_started`（除 brand_profile 因 description IS NULL 也是 `not_started`） |
| H3 | 进入第 1 步 `brand_profile`，填 `description="测试品牌简介"` + `industry_type="fitness"` + `logo_url="https://..."`，点"保存并继续" | PATCH `/api/v1/brand/profile` 返 200；前端跳到第 2 步；GET `/api/v1/brand/onboarding/status` 显示 step `brand_profile` status=`completed` |
| H4 | 第 2 步 `location` 页加载，列表为空 | 显示"暂无门店，请创建第一个"+ "新增"按钮；step 状态仍为 `not_started`（count=0 < target=1） |
| H5 | 点"新增"，填 `name="总店"` + `address="北京市朝阳区..."` + `phone="010-12345678"`，提交 | POST `/api/v1/brand/locations` 返 201；列表显示一条；step `location` status=`completed`；进度条 2/8 |
| H6 | 点"下一步"进入第 3 步 `staff`，看到"敬请期待"占位 + "跳过此步"按钮 | 显示占位文案；进度条第 3 格高亮但灰显 |
| H7 | 点"跳过此步"，确认弹窗后跳过 | PATCH `/api/v1/brand/onboarding/steps/staff/skip` 返 200；status=`skipped`；进度 3/8；自动跳到第 4 步 |
| H8 | 第 4-8 步（`course_category` / `course_template` / `entitlement_template` / `class_session` / `mini_program_qrcode`）均跳过 | 进度依次 4/8、5/8、6/8、7/8、8/8 |
| H9 | 8/8 后弹"完成开通"按钮 | POST `/api/v1/brand/onboarding/complete` 返 200；brands.onboarding_status=`completed`、onboarding_completed_at 非空；前端跳工作台 `/` |
| H10 | 工作台首页 | 不再显示 onboarding 进度条（已 completed）；显示常规空状态 |

---

## Edge Cases — 入口 / 守卫

| # | 场景 | 预期结果 |
|---|---|---|
| E1 | 未登录访问 `/onboarding` | 重定向 `/login?next=/onboarding` |
| E2 | brand.status=`pending`（支付未成功）登录后访问 onboarding | 后端 GET `/onboarding/status` 返 403 `BRAND_NOT_ACTIVE`；前端跳支付页 |
| E3 | brand.onboarding_status=`completed` 后再访问 `/onboarding` | 后端返 200 但前端检测到 completed 时自动跳工作台 |

---

## Edge Cases — 品牌资料（第 1 步）

| # | 场景 | 预期结果 |
|---|---|---|
| E4 | description 为空提交 | 前端 RHF 校验提示"请填写品牌简介"；不发请求 |
| E5 | industry_type 未选 | 前端校验提示；不发请求 |
| E6 | description > 2000 字符 | 前端校验提示"最多 2000 字"；后端兜底返 400 `INVALID_PARAM` |
| E7 | 调 PATCH `/onboarding/steps/brand_profile/skip` | 返 400 `STEP_NOT_SKIPPABLE` |
| E8 | brand_code 重复（与其他 brand 撞） | 后端返 409 `BRAND_CODE_DUPLICATED`；前端 inline 错误 |
| E9 | 尝试修改 `name` / `contact_phone` 字段 | 后端忽略（白名单只接受 logo_url/description/industry_type/brand_code/contact_email）；返回的 brand 中这两个字段保持不变 |

---

## Edge Cases — Location（第 2 步）

| # | 场景 | 预期结果 |
|---|---|---|
| E10 | Location 名称为空 | 前端校验；不发请求 |
| E11 | Location 同名重复（同 brand 已有 active 同名） | 后端返 409 `LOCATION_NAME_DUPLICATED`；前端 inline 错误 |
| E12 | 已建满 max_locations（plan id=2 默认 3），再 POST 第 4 个 | 后端返 409 `QUOTA_EXCEEDED` + `{ current: 3, max: 3 }`；前端显示"门店数已达套餐上限 3/3，请联系平台升级套餐" |
| E13 | Subscription 状态 `frozen` 时 POST Location | 后端返 403 `SUBSCRIPTION_RESTRICTED`；前端提示"订阅已冻结，无法新增" |
| E14 | 同名 Location 已 soft-deleted 后，重新创建同名 | 成功创建（唯一索引 `WHERE deleted_at IS NULL` 允许） |
| E15 | 调 PATCH `/onboarding/steps/location/skip` | 返 400 `STEP_NOT_SKIPPABLE` |
| E16 | Location 启用/停用（PATCH `/:id/status`） | status 更新；写入 OperationLog `action=location_status_changed`；step `location` 仍 `completed`（不因 inactive 跌回 not_started） |
| E17 | DELETE Location（软删） | 204 No Content；列表过滤掉；OperationLog `action=location_deleted`；quota 计数减 1（释放配额给后续新建） |
| E18 | DELETE 不属于当前 brand 的 location id | 404 `LOCATION_NOT_FOUND`（跨品牌隔离） |
| E19 | PATCH 非 brand 自身的 location | 404 `LOCATION_NOT_FOUND` |

---

## Edge Cases — 跳过 / 完成

| # | 场景 | 预期结果 |
|---|---|---|
| E20 | 8 步未全部 completed/skipped 调 POST `/onboarding/complete` | 后端返 400 `ONBOARDING_NOT_READY` + 当前未完成 step 列表 |
| E21 | 重复调 POST `/onboarding/complete` | 第一次成功；第二次直接返 200 幂等（brand.onboarding_status 已是 completed） |
| E22 | metadata 清空验证 | onboarding completed 后查询 brand_onboarding_steps，所有行 metadata = `{}` |
| E23 | PATCH 不存在的 step_key（如 `/skip/foo`） | 400 `INVALID_STEP_KEY` |

---

## Edge Cases — 并发 / quota 串行化

| # | 场景 | 预期结果 |
|---|---|---|
| E24 | 并发 POST 2 个 Location（已存在 max-1 个），同时到达 | 一成（201）一败（409 `QUOTA_EXCEEDED`）；DB 中 locations 计数严格 = max |
| E25 | 并发 POST + DELETE 同时进行 | 串行化生效；不出现"quota 看到旧 count 误判"导致超额 |

---

## 执行方式

**单元测试**：
- `internal/application/onboarding/service_test.go`：覆盖 step 完成判定逻辑（不同 count vs target 矩阵）
- `internal/application/location/service_test.go`：覆盖 quota check / subscription guard 分支（E12 / E13 / E14 / E18）

**集成测试**（GORM 事务）：
- `internal/infrastructure/persistence/location_repository_integration_test.go`：用 testcontainer 起 pg，跑并发 quota 场景（E24 / E25）
- onboarding repo COUNT 聚合查询的正确性（H2 / H5）

**端到端（browser-automation skill 跑 Playwright）**：
- 完整 Happy Path H1–H10
- 入口守卫 E1 / E2 / E3
- 品牌资料表单校验 E4 / E5 / E6
- Location 创建 + quota 提示 E10 / E11 / E12 / E13

**手动 curl 抽检**：
```bash
# 拿 token
TOKEN=$(curl -s -X POST http://localhost:8081/api/v1/brand/login \
  -H "Content-Type: application/json" \
  -d '{"phone":"18816820405","password":"test1234"}' | jq -r .data.token)

# 查 onboarding 状态
curl -s http://localhost:8081/api/v1/brand/onboarding/status \
  -H "Authorization: Bearer $TOKEN" | jq

# 填品牌资料
curl -s -X PATCH http://localhost:8081/api/v1/brand/profile \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"description":"测试简介","industry_type":"fitness"}' | jq

# 建 Location
curl -s -X POST http://localhost:8081/api/v1/brand/locations \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"总店","address":"测试地址","phone":"010-12345678"}' | jq

# 跳过 staff 步骤
curl -s -X PATCH http://localhost:8081/api/v1/brand/onboarding/steps/staff/skip \
  -H "Authorization: Bearer $TOKEN" | jq
```
