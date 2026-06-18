# Batch 13a — 学员档案管理（BrandLearnerProfile + 标签）

> Batch 13（学员预约闭环）的第一个子批。依赖链：**13a 学员 → 13b 权益 → 13c 预约 → {13d 候补, 13e 签到}**。
> 拍板结论（2026-06-17 会话内）：5 子批 ｜ 全程 brand staff_assisted（C 端延后 Batch 14）｜ 13a = 核心档案 + 冻结 + 标签 ｜ bookings/waitlist 取消后重约改 partial unique（13c）。

## 0. 本批定位

学员档案是整个预约闭环的**前置主数据**：没有 `brand_learner_profiles`，13b 无法发放权益、13c 无法下单。当前**零代码**（表已在 000003 建好；brand 端 `/users`、`/trainings` 是 legacy `app_users` / 健身训练记录，与本表无关）。

本批做：学员档案 CRUD + 冻结/解冻 + 标签管理 + 学员额度门。镜像 location/staff 纵切片流水线。

**不做（明确延后）：**
- 员工服务关系 `learner_staff_assignments`（顾问/主教练/跟进人）→ 后续批（喂 instructor「自己相关学员」data_scope，但那是 C 端/instructor 视图关注点）。
- 学员详情页的「权益 / 预约 / 履约」Tab 内容 → 13b / 13c / 13e 落地时填充，本批先放空态占位。
- 批量导入学员 → 后续需求池（§20.7 提及，V1 先单条创建）。
- 微信 openid 真实绑定 → C 端批（本批用合成 open_id 占位）。

## 1. 关键设计点

### 1.1 无微信学员的身份创建（schema 强约束）
`learner_identities.wechat_open_id` 是 `NOT NULL` + unique。员工后台创建学员时尚无微信身份，做法：
- 用**合成占位 open_id**：`manual:<brand_id>:<phone>`（保证唯一、可读、可对账）。
- `learner_identities.phone` 有 partial-unique（`WHERE phone IS NOT NULL`）→ **find-or-create by phone**：同一手机号在多品牌复用同一 identity（底层支持一身份多 brand，§19）；同品牌重复手机号撞 `idx_brand_learner_profiles_brand_identity` → `LEARNER_ALREADY_EXISTS`。
- 日后 C 端微信登录时，按 phone 把真实 `wechat_open_id` 回填到既有 identity（本批不做，留 FR）。

### 1.2 学员额度门（已就绪，§4.1）
`brand_learner_profiles` Create 走 `SubscriptionGuard.CheckAndCount(ctx, tx, brandID, ResourceLearner, 1)`——`ResourceLearner` / `MaxLearners` 已在 `subscription_guard.go` 接好，本批只在 create tx 内调用即可。超限返 `QUOTA_EXCEEDED`（复用）。entitlement/booking 本身不限量。

### 1.3 data_scope
学员域按 `primary_location_id` ∈ assigned_locations 过滤（镜像 location/classsession：`scopeFilterIDs` 用于 list，`guardLocationInScope` 用于 detail/write，out-of-scope 返 404 不泄漏存在性）。`primary_location_id` 可为 NULL（未分配主门店）——assigned_locations 模式下，NULL 主门店的学员对非 all_brand 员工不可见（与 location 行为一致）。

### 1.4 软删 + 引用保护（提前写，镜像 12a）
`brand_learner_profiles` 用 `gorm.DeletedAt` 软删。Delete 前 `countLearnerActiveReferences`（active `learner_entitlements` + 未来/active `bookings`）> 0 → `LEARNER_IN_USE`。13a 落地时这两表无数据恒 0，提前写避免 13b/13c 返工（12a→12b 先例零返工）。

### 1.5 冻结语义（§20.7）
`status: active ↔ frozen`（+ inactive 保留）。冻结后学员不能自助预约（C 端批校验）；**不自动取消已有预约**；存在未来预约时系统提示操作人处理（未来预约校验留到 13c bookings 存在后补，本批仅翻 status + 写 OperationLog）。

---

## 契约

### API 接口

> 前缀 `/api/v1/brand`。鉴权 + RBAC `require(code)`（checker==nil bypass）+ data_scope，镜像 staff/location handler。列表统一 `response.SuccessPage`，写操作 `response.Success`，错误 `response.Error`。

**学员档案**

| 方法 | 路径 | 请求字段 | 响应字段 | 权限 |
|---|---|---|---|---|
| GET | /learners | `q`(昵称/手机号/学号模糊), `status`, `primary_location_id`, `page`, `page_size` | `items[]`{id, learner_no, nickname, phone, avatar_url, primary_location_id, **primary_location_name**, status, tags[]{id,name,color}, created_at}, `total`, `page`, `page_size` | learner.view |
| GET | /learners/:id | — | 同上单条 + `remark`, `learner_identity_id`；预留 `entitlements/bookings/records`（本批不返或空数组占位） | learner.view |
| POST | /learners | `phone`(必填), `nickname`, `primary_location_id`, `learner_no`, `remark`, `tag_ids[]` | 创建后的学员对象 | learner.create |
| PATCH | /learners/:id | 白名单：`nickname`, `primary_location_id`, `learner_no`, `remark`, `tag_ids[]`(全量替换) | 更新后的学员对象 | learner.edit |
| PATCH | /learners/:id/status | `status`(active/frozen) | 更新后的学员对象 | learner.freeze |
| DELETE | /learners/:id | — | 204 | learner.delete |

**学员标签**（品牌级主数据，无 location scope）

| 方法 | 路径 | 请求字段 | 响应字段 | 权限 |
|---|---|---|---|---|
| GET | /learner-tags | `status` | `items[]`{id, name, color, status, created_at} | learner.view |
| POST | /learner-tags | `name`(必填), `color` | 标签对象 | learner.edit |
| PATCH | /learner-tags/:id | `name`, `color` | 标签对象 | learner.edit |
| PATCH | /learner-tags/:id/status | `status`(active/inactive) | 标签对象 | learner.edit |

> 标签不做硬删（镜像 course-categories：仅停用）。学员↔标签关联通过 `PATCH /learners/:id` 的 `tag_ids[]` 全量替换维护（事务内 DELETE WHERE profile_id + 逐条 INSERT，镜像 staff_location_assignments / course_category_assignments）。

### 错误码（新增）

| code | HTTP | 触发 |
|---|---|---|
| `LEARNER_NOT_FOUND` | 404 | 学员不存在 / out-of-scope |
| `LEARNER_ALREADY_EXISTS` | 409 | 同品牌该手机号已有学员档案（撞 `idx_brand_learner_profiles_brand_identity`） |
| `LEARNER_NO_DUPLICATED` | 409 | 同品牌学号重复（撞 `idx_brand_learner_profiles_brand_learner_no`） |
| `LEARNER_IN_USE` | 409 | 删除时仍有 active 权益 / 未来预约（本批恒不触发，提前留） |
| `LEARNER_TAG_NOT_FOUND` | 404 | 标签不存在 / `tag_ids` 含无效 id |
| `LEARNER_TAG_NAME_DUPLICATED` | 409 | 同品牌标签名重复（撞 `idx_learner_tags_brand_name`） |
| `QUOTA_EXCEEDED` | 409 | 学员数超订阅额度（复用 SubscriptionGuard，自 Batch 4 起为 409） |

> 唯一冲突一律 `errors.As(*pgconn.PgError)` + SQLSTATE 23505 **按约束名分流**（镜像 Batch 7/12a），不裸 500。

### 数据库迁移 `000009_learner_permissions`

不动表结构，三段式镜像 000008：
1. **permissions**：`learner.view / learner.create / learner.edit / learner.delete / learner.freeze`（domain=`learner`）。
2. **role_template 映射**（按 §21.1/§21.2 矩阵）：
   - `brand_owner` / `brand_admin`：全 5 码。
   - `course_operator`（课程运营）/ `finance`（财务售后）：`learner.view`。
   - `location_manager`（店长）/ `receptionist`（前台）：`learner.view`（本 Location，靠 data_scope）。
   - `instructor`：`learner.view`（自己相关学员——服务关系本批未做，先按 scope view）。
3. **存量 brand backfill**：`brand_role_permissions` JOIN `role_template_permissions`（镜像 000008，给已存在 brand 的对应角色补码）。

> `learner.delete` 的 Expand 自动隐含 view+edit；映射仍全列以保证 PermissionSet 显式一致（同 000008 注释）。

---

### 前端页面模块

| 页面/模块 | 类型 | 关键字段/操作 |
|---|---|---|
| `/learners` 学员列表 | 页面 | DataTable（学号/昵称/手机号/主门店/标签 chips/状态 badge）+ 状态筛选 Select + 主门店筛选 Select + 搜索 Input（q）+ 分页；行操作 编辑 / 冻结切换 / 删除（按 learner.edit/freeze/delete gate + Hint）；空态文案 |
| LearnerFormDialog | 弹窗 | RHF + Zod：phone（创建必填、编辑只读展示）、nickname、primary_location（active 门店下拉）、learner_no（选填）、remark、tags（active 标签多选 chip）；`QUOTA_EXCEEDED` toast + inline 双展示（镜像 StaffCreateDialog）；`LEARNER_ALREADY_EXISTS`/`LEARNER_NO_DUPLICATED` inline 错误 |
| LearnerStatusToggle | 弹窗/按钮 | 冻结/解冻（按钮 + ConfirmDialog，镜像 LocationStatusToggle；非 shadcn switch） |
| `/learners/[id]` 学员详情 | 页面 | 基础信息卡（昵称/手机号/学号/主门店/备注/状态）+ 标签区 + 编辑入口；**「权益 / 预约 / 履约」Tab 占位空态**（13b/13c/13e 填充） |
| `/learner-tags` 标签管理 | 页面 | 表（name/color/status）+ 状态切换 + 标签表单弹窗（name/color）；镜像 `/course-categories` |
| 导航 | — | 加「学员管理」(`/learners`) + 「学员标签」(`/learner-tags`)；`NAV_HREF_PERMISSIONS` 加 `learner.view` 门 |

### 前端实现约束

- 复用 /web 现有组件和设计风格（DataTable / ConfirmDialog / Hint / RHF+zod form-dialog），不引入新 UI 库。
- 仅改 `apps/brand`，不跨端。
- **新 api client `packages/api/src/learners.ts` + `learner-tags.ts` 必须在 `packages/api/package.json` 的 `exports` 登记**（镜像 location-resources 踩坑），并加 `packages/types` 类型 + `errors.ts` 错误码常量 + `PERMISSIONS.LEARNER_*` 常量。
- 任何前端会 `.map()` 迭代的数组字段（`tags`、`items`）后端 DTO 不用 `omitempty`，nil 规整为空切片（镜像 Batch 5 踩坑）。
- mutation hook `invalidateQueries(refetchType:'all')`；data-testid 钩子按既有命名（`learner-create-button` / `learner-row` / `learner-field-*` / `learner-submit`）给 e2e。

---

## 本批待 approve 的细节（请确认或调整）

1. **标签独立页** `/learner-tags`（镜像 course-categories）vs 折叠进学员页侧栏 → 契约取**独立页**。
2. **learner_no 手填选填**（不自动生成）→ 契约取手填选填，可空。
3. **详情页 Tab 占位**：本批放空态，13b/13c/13e 填充 → 契约取占位。
4. **冻结仅翻 status**（未来预约校验/提示留到 13c）→ 契约取此。
5. **DELETE 软删 + 提前写 `LEARNER_IN_USE` guard**（本批恒 0）→ 契约取此。

approve 后：写 `batch-13a-learner-profile-tests.md` 测试场景 → 主线程逐 task TDD commit。
