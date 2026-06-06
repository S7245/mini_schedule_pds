# Batch 04 — 品牌初始化向导骨架 + Location 闭环

状态：**已 approve** ✅ — 进入实现阶段
approve 时间：2026-06-06
更新时间：2026-06-06
PROGRESS 链接：见 `pds/PROGRESS.md` Batch 4 区块（接 approve 后回填）

---

## 业务背景

Batch 3 完成微信支付回调后，Brand 从 `pending` 切到 `active`，并创建了 BrandSubscription 快照。本批要打通**支付成功 → 登录 → 进入初始化向导 → 完成第一条业务链路**的入口闭环。

由于"向导 8 步要求 6 个域的 CRUD"和"一批一闭环"冲突，本批只做 **向导骨架 + 第 2 步 Location 真实闭环**，其余 6 步显示"敬请期待"占位 + 可跳过。后续 Batch 依次填实。

```text
品牌支付成功
  → 登录 brand 端
  → 引导到 /onboarding
  → 第 1 步：完善品牌资料（实做）
  → 第 2 步：创建第一个 Location（实做，含 quota 校验）
  → 第 3-8 步：占位"敬请期待" + 跳过
  → 全部完成 / 全部跳过 → 工作台
  → 工作台首页显示未完成进度条引导继续
```

**额度硬限制范围**：本批仅落 Location 的 quota check（max_locations）。Staff / Learner 的 quota 留给后续 Batch 在对应 CRUD 落地时一并加。

---

## 设计决定（grill 阶段产出）

1. **向导步骤 source of truth**：`brand_onboarding_steps` 表是唯一权威；`brands.onboarding_status` 是 denormalized 缓存，由 application 层在每次 PATCH step 时同步刷新（`not_started` / `in_progress` / `completed` / `skipped_partial`）。
2. **8 个 step_key 命名**（按蓝图 §5.1 + §20.4 — approve 后定版）：
   - `brand_profile` — 完善品牌资料（本批实做）
   - `location` — 创建第一个 Location（本批实做）
   - `staff` — 创建员工 / 授课员工（合并，需要 active staff ≥ 1 且 instructor_profile ≥ 1 才 completed）
   - `course_category` — 创建课程分类
   - `course_template` — 创建课程
   - `entitlement_template` — 创建学员权益模板
   - `class_session` — 创建第一节场次
   - `mini_program_qrcode` — 发布小程序入口 / 二维码
3. **完成判定策略**：每个 step 的 `status` 由 application 层在 GET 接口实时计算（query：`select count(*) from locations where brand_id=? and deleted_at is null` 等），不靠业务事件钩子反向更新。理由：避免业务事件遗漏导致状态漂移。读路径稍贵但 8 张表的 COUNT 在百万级以内可接受。
4. **手动跳过**：第 3-8 步允许"跳过"，第 1-2 步不允许跳过（必须实做）。跳过的 step 计入"完成"，进度条满 8/8 后弹"完成开通"。
5. **quota 校验时机**：Location POST 前做 quota check，使用 `SELECT FOR UPDATE brand_subscription WHERE brand_id=?` 串行化，避免并发 INSERT 突破上限。计数 = `count(locations where brand_id=? and deleted_at is null)`（**包含 inactive**，避免 disable → 腾位 hack）。
6. **Subscription 状态门禁**：BrandSubscription 状态非 `active` 一律拒绝 POST Location，返回 `SUBSCRIPTION_RESTRICTED` 错误（与 quota 错误分开）。
7. **支付成功 → onboarding 跳转**：支付页轮询到 `paid` 后跳 `/onboarding`，前端 onboarding 守卫检测未登录则带 `next=/onboarding` 跳 `/login`，登录成功后 redirect 回。
8. **品牌资料第 1 步表单字段**：`logo_url` / `description`（**新增 brand.description 字段**） / `industry_type` / `brand_code`（可选）/ `contact_email`（如已填则只读）。 → **本批需要一次小 migration 加 `brands.description VARCHAR(2000)`**。
9. **Location 名称重复策略**：依赖现有唯一索引 `(brand_id, name) WHERE deleted_at IS NULL` → INSERT 冲突时返回 `LOCATION_NAME_DUPLICATED` + 409。

---

## API 接口契约

### 后端：onboarding 域

| 方法 | 路径 | 鉴权 | 请求 | 响应 |
|---|---|---|---|---|
| GET  | `/api/v1/brand/onboarding/status` | brand JWT | — | `{ overall_status, steps: [{step_key, status, completed_at, skipped_at, count, target}], next_step_key }` |
| PATCH | `/api/v1/brand/onboarding/steps/:step_key/skip` | brand JWT | `{ reason?: string }` | `{ step_key, status, skipped_at }` |
| POST | `/api/v1/brand/onboarding/complete` | brand JWT | — | `{ overall_status, onboarding_completed_at }` （8 步全 completed/skipped 时才允许，否则 400） |

**说明**：
- 第 1-2 步无 PATCH `/skip`（业务上禁止跳过）；调 skip 返 `STEP_NOT_SKIPPABLE` 400
- 完成动作（如填表 / 建 Location）通过对应资源 API（如下方 brand profile UPDATE / location POST）触发，无需单独 PATCH `/complete`
- GET 接口在每次响应中实时计算 count vs target（如 location step 的 count = locations 表数，target = 1）

### 后端：brand profile（向导第 1 步）

| 方法 | 路径 | 鉴权 | 请求 | 响应 |
|---|---|---|---|---|
| GET  | `/api/v1/brand/profile` | brand JWT | — | brand 完整字段（name / logo_url / contact_name / contact_phone / contact_email / industry_type / brand_code / description / status） |
| PATCH | `/api/v1/brand/profile` | brand JWT | `{ logo_url?, description?, industry_type?, brand_code?, contact_email? }` (name / contact_name / contact_phone 不允许改) | 更新后的 brand |

完成判定：brand.description IS NOT NULL AND brand.industry_type IS NOT NULL → `brand_profile` step 计 completed。

### 后端：Location 域（向导第 2 步 + 后续日常使用）

| 方法 | 路径 | 鉴权 | 请求 | 响应 |
|---|---|---|---|---|
| GET  | `/api/v1/brand/locations` | brand JWT | `?page=1&page_size=20&status=active/inactive/all`（默认 all） | `{ items: [...], total, page, page_size }` |
| GET  | `/api/v1/brand/locations/:id` | brand JWT | — | location 完整字段 |
| POST | `/api/v1/brand/locations` | brand JWT | `{ name, address?, phone?, remark? }` | 新建 location |
| PATCH | `/api/v1/brand/locations/:id` | brand JWT | `{ name?, address?, phone?, remark? }` | 更新后的 location |
| PATCH | `/api/v1/brand/locations/:id/status` | brand JWT | `{ status: 'active' \| 'inactive' }` | 更新后的 location |
| DELETE | `/api/v1/brand/locations/:id` | brand JWT | — | `204 No Content`（软删） |

**错误码**：
| Code | HTTP | 场景 |
|---|---|---|
| `SUBSCRIPTION_RESTRICTED` | 403 | 订阅状态 ≠ active |
| `QUOTA_EXCEEDED` | 409 | location 数 ≥ max_locations |
| `LOCATION_NAME_DUPLICATED` | 409 | 同品牌同名 active location 已存在 |
| `LOCATION_NOT_FOUND` | 404 | id 不属于当前 brand 或已删 |

**OperationLog 写入点**：Location 创建 / 状态切换 / 软删除 → action 为 `location_created` / `location_status_changed` / `location_deleted`，actor_type=`brand_user`。普通字段编辑（地址、电话、备注）不写 OperationLog。

---

## 前端页面模块（apps/brand）

| 页面 / 模块 | 类型 | 关键字段 / 操作 |
|---|---|---|
| `/onboarding` | 页面（守卫：需登录 + brand.status=active） | 步骤进度条（1-8）+ 当前步内容区 + "上一步 / 下一步 / 跳过"按钮组 |
| `/onboarding/brand-profile` | 子页 | 表单：logo 上传 / description textarea / industry_type Select / brand_code input（可选，含校验提示） / contact_email input；提交 → PATCH /profile |
| `/onboarding/locations` | 子页 | 列表（DataTable：名称 / 地址 / 状态 / 操作）+ "新增"按钮 → Dialog 表单；至少 1 个 active location 进入下一步；含 status switch、edit Dialog、删除 ConfirmDialog |
| `/onboarding/staff` … `/onboarding/class-session` | 子页（6 个） | "敬请期待"占位 + "跳过此步"按钮 |
| `/onboarding/complete` | 完成态 | "开通成功"祝贺 + "进入工作台"按钮 |
| 工作台首页 `/` | 页面 | 如 onboarding 未完成：顶部显示进度条 + "继续完成开通"CTA；其余空状态可参考 admin-system 模式 |
| `WizardShell` | 组件 | 抽象进度条 + 步骤导航壳，复用注册流程已有的 PageShell（pds/.learnings 提到的复用点） |
| `LocationFormDialog` | 组件 | 创建 / 编辑统一 Dialog，RHF + Zod 校验 |
| `LocationStatusSwitch` | 组件 | shadcn switch，需 `pnpm shadcn add switch`（参见 web/.learnings ERRORS） |

**前端实现约束**（per pds/CLAUDE.md §6 前端）：
- 复用 packages/api 的 http client + React Query hook 模式（`useBrandOnboardingStatus` / `useBrandLocations` / `useCreateLocation` 等）
- 错误展示走 `apiError` inline + sonner toast（per web/.learnings）
- 路由分组沿用 `(protected)`，登录守卫在 layout 层做
- 表单：React Hook Form + Zod schema
- 不引入新 UI 库；status switch 用 shadcn `switch`

---

## 数据库变更

### 唯一新增：brand.description 字段

```sql
-- 000004_brand_profile_extras.up.sql（新 migration）
ALTER TABLE brands
  ADD COLUMN IF NOT EXISTS description VARCHAR(2000);
```

```sql
-- 000004_brand_profile_extras.down.sql
ALTER TABLE brands DROP COLUMN IF EXISTS description;
```

其余表（brand_onboarding_steps / locations）**复用已有 schema**，不动迁移。

---

## 后端四层实现范围

### Domain（新增两个域）

`internal/domain/onboarding/`：
- `step.go`：StepKey 枚举（8 个常量）+ StepStatus 枚举
- `service.go`：`OnboardingStatus`（聚合多个 step 的视图模型）+ Repository interface（GetSteps / SkipStep / MarkComplete / RefreshDerivedStatus）

`internal/domain/location/`：
- `location.go`：Location 实体 + Status 枚举
- `service.go`：CRUD input/output + Repository interface

> 也可以挂在 `commercial` 域里，但更干净是分新域。决定：新建 `onboarding` 和 `location` 两个 domain package。

### Application

- `internal/application/onboarding/service.go`：编排 GET status（聚合 7 个表 COUNT 查询）/ Skip / Complete
- `internal/application/location/service.go`：编排 CRUD + 调 `commercial.SubscriptionGuard`（新抽出的小服务，做 Subscription 状态 + quota 校验）

新增轻量公共服务 `internal/application/commercial/subscription_guard.go`（也可挂 domain 层）：
- `CheckCanCreateLocation(ctx, brandID) error` — SELECT FOR UPDATE subscription + COUNT locations

### Infrastructure

- `internal/infrastructure/persistence/onboarding_repository.go` + `onboarding_models.go`
- `internal/infrastructure/persistence/location_repository.go` + `location_models.go`
- onboarding 的 COUNT 聚合查询要 7 张表（locations / staffs / instructor_profiles / course_categories / course_templates / learner_entitlements / class_sessions）— 后 6 张本批没 model，临时用 raw SQL `r.db.Raw("SELECT COUNT(*) FROM xxx WHERE brand_id=?")` 实现即可

### Interfaces

- `internal/interfaces/brand/onboarding_handler.go`（新）
- `internal/interfaces/brand/profile_handler.go`（新）
- `internal/interfaces/brand/location_handler.go`（新）
- 都挂在 `brand` group 下（已有 JWT middleware 保护）

### Wire 装配

- 三个新 service + 三个新 handler 加进 `cmd/api-brand/wire.go`
- **同时跑 `go generate ./...` 修正之前手改的 wire_gen 漂移**（per backend/.learnings 提到的 Wire 漂移问题）

---

## 测试场景概览

详细测试场景待 approve 后写入 `pds/batches/batch-04-onboarding-location-tests.md`。预期覆盖：

### Happy Path
- 登录 → 进 onboarding → 填品牌资料 → 进度 2/8 → 建 1 个 location → 进度 3/8 → 第 3-8 步全跳 → 8/8 → 工作台

### Edge Cases
- 第 1 步必填字段缺失 / industry_type 选项校验
- 第 2 步 Location 名称重复 → 409
- 第 2 步达到 max_locations → 409 QUOTA_EXCEEDED + 升级套餐提示
- Subscription 状态 frozen → 403 SUBSCRIPTION_RESTRICTED
- 并发创建 Location（2 个 tab 同时点提交，刚好命中 max）→ 一成一败
- 直接 PATCH `/onboarding/steps/brand_profile/skip` → 400 STEP_NOT_SKIPPABLE
- 8 步未全完成调 `/onboarding/complete` → 400
- 未登录访问 /onboarding → 重定向 /login?next=/onboarding
- Brand status=pending（未支付）访问 /onboarding → 后端 403 + 前端跳支付页

---

## 验收清单

**后端验收**：
- [ ] `go build ./... && go vet ./...` 通过
- [ ] `go generate ./...` 通过，wire_gen 与 wire.go 无 diff
- [ ] migration 000004 up/down 可顺序执行
- [ ] Location quota 并发测试（用 advisory lock 或 SELECT FOR UPDATE 验证）

**前端验收**：
- [ ] `pnpm --filter @mini-schedule/brand build` 通过
- [ ] `pnpm --filter @mini-schedule/brand exec tsc --noEmit` 通过（per web/.learnings ERRORS：next lint 已 deprecated）
- [ ] shadcn `switch` 已 add 并提交（含 lockfile）

**业务验收（curl + 浏览器手动）**：
- [ ] 登录 brand 端 → 进 /onboarding 看到 8 步进度条（实时计算第 1-2 步 status）
- [ ] PATCH /profile 填资料 → GET /onboarding/status 显示 step `brand_profile` completed
- [ ] POST /locations 建一个 → step `location` completed
- [ ] POST 第 4 个 location（plan id=2 默认 max=3） → 409 QUOTA_EXCEEDED
- [ ] PATCH `/onboarding/steps/staff/skip` → step status=skipped；全跳完 → POST /complete → brand.onboarding_status=completed

---

## 待用户 approve 的关键决定

1. **第 8 步是否合并到 class_session** — 我倾向合并（小程序二维码作为副产物在场次发布后自动暴露），共 8 个 step_key 减为 7 个。要不要保留独立第 8 步？
2. **brand.description 字段加 2000 字符是否合适** — 后续小程序详情页要展示，字数预算够吗？
3. **status switch 是不是用 shadcn `switch`** — 还是用既有 admin-system 的 Toggle / RadioGroup？
4. **onboarding 完成后是否清空所有步骤的 metadata** — 还是保留作为审计？
5. **本批是否要写 OperationLog 记录"品牌资料完善" / "向导某步跳过"** — 我倾向只对 Location 写（per OperationLog 商业关键操作约束），向导操作不写。同意吗？

逐条回复 OK 或修改意见，全部 OK 我会：

- 写 `batch-04-onboarding-location-tests.md` 测试场景
- 并行 spawn 后端 + 前端 subagent（按 TDD 逐 task commit）
- 跑 go build / pnpm build 静态验证
- 跑 /code-review 自检
- 发邮件请你做业务验收
