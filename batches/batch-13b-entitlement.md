# Batch 13b — 权益产品 + 发放（EntitlementProduct / LearnerEntitlement / Transaction）

> Batch 13（学员预约闭环）第二子批。链：13a 学员 ✅ → **13b 权益** → 13c 预约 → {13d 候补, 13e 签到}。
> 拍板（2026-06-18 会话内）：复用粗权限码 entitlement.view/manage/adjust + 补 §21.1 缺失映射 ｜ 手动调整 = 额度增减 + 状态(冻结/作废/恢复) ｜ expired/depleted **落库**（读触发 settle sweep，无 cron）｜ 会员卡=不限次 + 有效期按 validity_days 自动算。

## 0. 本批定位

权益产品（模板）+ 给学员发放权益 + 额度账 + 流水 + 人工调整。lock→consume 模型本批只做**前半**（grant 设 total/remaining；adjust 改 remaining；`locked`/`consumed` 恒 0，13c hold / 13e consume 才动）。表全已建（000003），零代码。依赖 13a 学员 ✅ / B11 课程 ✅ / B4 门店 ✅，**无 booking 前向依赖**。填 13a 学员详情页「权益」Tab 占位。

**不做（延后）**：hold/release/consume/no_show_consume 流水（13c/13e）；权益到期/额度提醒通知；权益转卡/请假延期/共享卡（blueprint §5.7 复杂卡，后续需求池）；有效期延长（adjust 仅额度+状态，到期延长转 FR）。

## 1. 关键设计点

### 1.1 额度模型（lock→consume，本批半程）
- grant：按 product 快照 `total_credits`→`remaining_credits`（会员卡 total=NULL→不限次，entitlement total/remaining 也 NULL）；`locked=consumed=0`。写 `entitlement_transactions(action=grant, delta=total或0, balance_after=remaining)`。
- manual adjust：`SELECT FOR UPDATE` entitlement 行 → remaining += delta（delta 可正负，必填原因）→ 写流水(action=manual_adjust, delta, balance_after) + OperationLog。remaining 会 <0 → 业务错误 `ENTITLEMENT_INSUFFICIENT`（不裸触 CHECK 23514 → 500）。不限次卡（total NULL）不允许额度调整（无意义）。
- `SELECT FOR UPDATE` 为 13c hold 预留并发安全。

### 1.2 状态机 + settle（expired/depleted 落库）
LearnerEntitlement status：`active / frozen / cancelled`（手动）+ `expired / depleted`（派生**但落库**）。统一纯函数 `settleStatus(e)`：
```
cancelled → cancelled（终态）
frozen    → frozen（手动 hold，reactivate 前不变）
expires_at <= now           → expired
total_credits 非空 且 remaining<=0 → depleted
否则 → active
```
**落库时机（无 cron）**：① 读触发——`GET /learners/:id/entitlements` 在 SELECT 前对该学员 active 权益跑一次 settle 批量 UPDATE（per-learner，轻量）；② grant / adjust / status 变更后对该行 settle 落库。settle 只改 status，不写流水（无额度变动）。未被访问的过期权益在首次读到时才落库——V1 取舍（接受）。
- 手动状态：freeze（active/depleted/expired→frozen）；reactivate（frozen→ settle，可能立即又 expired/depleted）；cancel（→cancelled 终态）。cancelled 上不可再 adjust/freeze（`ENTITLEMENT_NOT_ADJUSTABLE`）。
- EntitlementProduct status：`active / inactive`（启用/停用，无 deleted_at，无硬删）。停用产品不可再发放（`ENTITLEMENT_PRODUCT_INACTIVE`）。

### 1.3 产品类型与额度/scope
- product_type：`class_pack`/`trial_pack`（total_credits 必填 >0）｜`membership_card`（total_credits=NULL 不限次）。DB CHECK entitlement_products_credit_model_valid 已兜。
- 频次上限 daily/weekly/monthly/concurrent_booking_limit：各可空(NULL=不限)或 >0。本批只存，13c 下单时校验。
- scope：location_scope/course_scope = `all`/`specific`。specific 时经 `entitlement_product_locations`/`_courses` 内联硬删重插（镜像 course_location_availability）；create/update 校验 ids ⊆ 本 brand active → 否则 `ENTITLEMENT_SCOPE_INVALID`。`all` 时关联清空。

### 1.4 权限（复用 + 补映射）
复用 000003 的 `entitlement.view`(只读) / `entitlement.manage`(产品 CRUD + 发放) / `entitlement.adjust`(对已发权益的额度/状态干预)。**无 data_scope**（§21 权益是品牌级；店长/前台默认无权益）。migration `000011` 不加新码，只补 §21.1 缺失映射 + 存量 backfill：
- brand_admin += `entitlement.adjust`（§21.1 品牌管理员=全部）
- course_operator += `entitlement.view`（§21.1 课程运营=查看；000003 漏给）
- finance_support += `entitlement.adjust`（§21.1 财务/售后=全部）

### 1.5 发放日期
grant：`starts_at` 可选默认今天；`expires_at = starts_at + product.validity_days`（自动算，不让员工手填）。未来 starts_at 允许存（13c usable 查询会校 starts_at<=now），本批显示日期不加「未生效」派生态。

---

## 契约

### API 接口（前缀 `/api/v1/brand`）

**权益产品（品牌级，gate entitlement.view/manage）**

| 方法 | 路径 | 请求字段 | 响应字段 |
|---|---|---|---|
| GET | /entitlement-products | `status`, `product_type`, `page`, `page_size` | items[]{id,name,product_type,total_credits,validity_days,daily/weekly/monthly/concurrent_booking_limit,location_scope,course_scope,status,issued_count,created_at}, total |
| GET | /entitlement-products/:id | — | 上 + description + location_ids[] + course_ids[] |
| POST | /entitlement-products | name, description, product_type, total_credits, validity_days, *_booking_limit, location_scope, course_scope, location_ids[], course_ids[] | 产品对象 |
| PATCH | /entitlement-products/:id | 白名单：name, description, total_credits, validity_days, *_booking_limit, location_scope, course_scope, location_ids[], course_ids[]（product_type 不可改） | 产品对象 |
| PATCH | /entitlement-products/:id/status | status(active/inactive) | 产品对象 |

**学员权益 + 流水（gate 见列）**

| 方法 | 路径 | 请求字段 | 响应字段 | 权限 |
|---|---|---|---|---|
| GET | /learners/:id/entitlements | — | items[]{id,product_id,product_name,product_type,status(settle 后),total_credits,remaining_credits,locked_credits,consumed_credits,starts_at,expires_at,remark,created_at} | view |
| POST | /learners/:id/entitlements | product_id, starts_at?(默认今天), remark? | 发放后的权益对象 | manage |
| GET | /entitlements/:id/transactions | — | items[]{id,action,delta_credits,balance_after,note,operated_by,created_at} | view |
| POST | /entitlements/:id/adjust | delta(非0), reason(必填) | 调整后的权益对象 | adjust |
| PATCH | /entitlements/:id/status | status(frozen/cancelled/active 即冻结/作废/恢复), reason? | 更新后的权益对象 | adjust |

### 错误码（新增 7）

| code | HTTP | 触发 |
|---|---|---|
| `ENTITLEMENT_PRODUCT_NOT_FOUND` | 404 | 产品不存在 |
| `ENTITLEMENT_PRODUCT_NAME_DUPLICATED` | 409 | 同品牌 active 产品重名（unique(brand,name) where active） |
| `ENTITLEMENT_PRODUCT_INACTIVE` | 409 | 从已停用产品发放 |
| `ENTITLEMENT_SCOPE_INVALID` | 400 | location_ids/course_ids 含非本 brand active 项 |
| `ENTITLEMENT_NOT_FOUND` | 404 | 学员权益不存在 |
| `ENTITLEMENT_INSUFFICIENT` | 409 | adjust 后 remaining<0；或对不限次卡调额度 |
| `ENTITLEMENT_NOT_ADJUSTABLE` | 409 | 对 cancelled 权益 adjust/改状态 |

复用：`LEARNER_NOT_FOUND`、`INVALID_PARAM`。唯一冲突按 23505 分流不裸 500。

### 数据库迁移 `000011_entitlement_role_mappings`

不动表、不加新权限码。三段镜像 000009：仅补 `entitlement.adjust→brand_admin`、`entitlement.view→course_operator`、`entitlement.adjust→finance_support` 的 role_template_permissions 映射 + 存量 brand backfill（JOIN role_template_permissions）。down 删这 3 条映射。

---

### 前端页面模块

| 页面/模块 | 类型 | 关键字段/操作 |
|---|---|---|
| `/entitlement-products` 产品管理页 | 页面 | 表（名称/类型/次数·有效期/频次上限/适用范围/状态 + issued_count）+ 状态/类型筛选 + 分页；行 编辑 / 启停（gate manage + Hint）|
| EntitlementProductFormDialog | 弹窗 | RHF+zod；type 选择（创建后锁定）；total_credits 仅 pack 显示；validity_days；4 个频次上限（选填）；location_scope/course_scope radio all/specific + specific 时门店/课程多选 chip；重名/scope 错误 inline |
| 学员详情「权益」Tab（落地占位） | 模块 | 权益列表（产品名/类型/剩余·总/不限次/到期/状态 badge[active·已冻结·已作废·已过期·已用完]）；「发放权益」按钮 |
| EntitlementGrantDialog | 弹窗 | 选 active 产品（下拉，显示类型/次数/有效期）+ starts_at(可选默认今天) + remark；展示将自动算的到期日 |
| EntitlementAdjustDialog | 弹窗 | delta(±数字) + reason(必填)；不限次卡禁用额度调整；`ENTITLEMENT_INSUFFICIENT` inline |
| 权益状态操作 | 按钮+Confirm | 冻结/恢复/作废（gate adjust + Hint），作废二次确认 |
| 权益流水抽屉/弹窗 | 弹窗 | 列 action(开通/调整/...)/delta/balance_after/原因/操作人/时间 |

### 前端实现约束

- 复用 /web 现有组件（DataTable/Dialog/ConfirmDialog/Hint/RHF+zod），不引入新 UI 库；仅改 `apps/brand`。
- 新 api client `entitlement-products.ts` + `entitlements.ts` 必须登记 `packages/api/package.json` exports；types + errors 7 码 + `PERMISSIONS.ENTITLEMENT_VIEW/MANAGE/ADJUST`。
- 数组字段（location_ids/course_ids/items/transactions）后端 DTO nil 规整空切片，不 omitempty。
- 导航加「权益产品」(`/entitlement-products`)；`NAV_HREF_PERMISSIONS['/entitlement-products']=entitlement.view`。学员「权益」Tab 复用 13a 详情页 Tabs 框架。

---

## 本批待 approve 的细节（已取默认，可改）

1. 产品**只启停不硬删**（无 deleted_at，镜像 course-categories）→ 取此。
2. 权益操作端点：发放/列表挂 `/learners/:id/entitlements`，调整/状态/流水挂 `/entitlements/:id/*` → 取此。
3. 不限次会员卡**禁额度调整**（adjust 仅对有限次卡）→ 取此。
4. settle 落库为**读触发 + 写变更后**（无 cron），未访问的过期权益首次读时才落库 → 取此。
5. adjust 仅额度+状态，**到期延长转 FR** → 取此。

approve 后：写 `batch-13b-entitlement-tests.md` → 主线程逐 task TDD（migration→domain/persistence+DB测→service+单测→handler+wire→前端）→ build/test → /code-review → 业务验收。
