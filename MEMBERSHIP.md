# 会员卡模块 — 产品设计规格（PDS）

整理日期：2026-05-24
状态：Phase 2 草稿，待评审

---

## 目录

1. [背景与目标](#1-背景与目标)
2. [术语对齐](#2-术语对齐)
3. [核心设计决策](#3-核心设计决策)
4. [数据模型](#4-数据模型)
5. [功能规格 — Phase 2](#5-功能规格--phase-2)
   - 5.1 会员卡产品管理
   - 5.2 学员开卡与持卡
   - 5.3 预约用卡校验
   - 5.4 卡券消费与退还
   - 5.5 卡冻结与解冻
6. [API 端点概览](#6-api-端点概览)
7. [状态流转](#7-状态流转)
8. [与其他模块的关系](#8-与其他模块的关系)
9. [分期交付计划](#9-分期交付计划)
10. [已确认决策](#10-已确认决策)

---

## 1. 背景与目标

### 1.1 背景

Phase 1 已完成排课 + 预约核心骨架，预约为免费（不消耗卡券）。Phase 2 引入**会员卡/次卡/课包体系**，让预约行为与权益消耗挂钩。

典型场景：
- 学员 A 在门店 A 购买了「教练 A 瑜伽团课-20节包」，该卡只能在门店 A 上「教练 A 瑜伽团课」这门课，其他课程/门店不可用。
- 学员 B 购买了品牌通用月卡，在品牌任意门店上任意课程均可使用。
- 不同学员根据需求买 20 节、30 节、40 节，各对应独立规格的产品。

### 1.2 目标

| 指标 | 目标 |
|---|---|
| 品牌可配置多种类型的会员卡产品 | Phase 2 交付 |
| 支持按门店 + 课程限定用卡范围 | Phase 2 交付 |
| 预约时自动校验并扣减用卡次数 | Phase 2 交付 |
| 取消预约后自动退还次数 | Phase 2 交付 |
| 爽约扣减次数 | Phase 1 已支持（字段预留），Phase 2 接入卡系统 |

### 1.3 范围说明

本文档覆盖 **Phase 2** 范围。在线支付购卡属于 Phase 3，不在本文约束范围内。Phase 2 购卡由 brand_admin 或 receptionist 在后台手动开卡（线下收款）。

---

## 2. 术语对齐

| 术语 | 含义 |
|---|---|
| MembershipProduct（会员卡产品） | 品牌配置的可售卡种模板，描述卡的类型、价格、有效期、适用范围 |
| LearnerMembership（学员持卡） | 学员购买后持有的卡实例，包含剩余次数、到期时间、状态 |
| MembershipTransaction（消费流水） | 每次预约扣减、取消退还、爽约扣减等操作的记录 |
| credits（次数） | 卡内剩余可使用次数；时间卡（time_pass）无此概念，不限次 |
| store_scope | 卡的门店适用范围：`brand`（全品牌）或 `store_specific`（指定门店） |
| course_scope | 卡的课程适用范围：`all`（所有课程）或 `course_specific`（指定课程模板） |

---

## 3. 核心设计决策

### D-M01：产品规格固定，不支持自选节次

20节包、30节包、40节包各是独立的 MembershipProduct，各自有独立名称和价格。品牌通过建立多个产品实现阶梯定价，不在单一产品内支持"自选节次"。

### D-M02：门店范围（store_scope）分两档

| store_scope | 含义 | 典型场景 |
|---|---|---|
| `brand` | 全品牌所有门店通用 | 时间月卡、品牌通用次卡 |
| `store_specific` | 仅限指定门店 | 某门店专属课程包 |

具体门店通过 `membership_product_stores` 关联表管理，仅 `store_scope = store_specific` 时有记录。

### D-M03：课程范围（course_scope）分两档

| course_scope | 含义 | 典型场景 |
|---|---|---|
| `all` | 门店范围内所有课程 | 通用次卡、月卡 |
| `course_specific` | 仅限指定课程模板 | 「教练A瑜伽团课」专属包 |

具体课程通过 `membership_product_courses` 关联表管理，仅 `course_scope = course_specific` 时有记录。

### D-M04：不在卡层级绑定教练

「教练 A 的课」通过 `course_scope = course_specific` 绑定对应课程模板实现，不在会员卡层面硬绑 coach_id。教练离职或调课不影响卡的有效性，只影响是否还有对应课次可约。

### D-M05：购买门店仅记录，不参与用卡校验

`learner_memberships.purchase_store_id` 记录在哪个门店购买（用于报表），校验依据是 product 的 scope 配置。

### D-M06：无匹配卡时不可预约

Phase 2 起，预约课程必须持有可用的匹配卡。无匹配卡时返回错误并引导购卡，不再提供免费预约兜底。

### D-M07：Phase 2 购卡为线下手动开卡

由 brand_admin 或 receptionist 在 Brand Backoffice 后台手动为学员开卡（线下收款后操作），在线支付购卡为 Phase 3。

---

## 4. 数据模型

### 4.1 实体关系概览

```
Brand
  └── MembershipProduct（会员卡产品，0..n）
        ├── MembershipProductStore（可用门店，0..n）→ Store
        └── MembershipProductCourse（可用课程，0..n）→ CourseTemplate

LearnerMembership（学员持卡）
  ├── product_id        → MembershipProduct
  └── learner_id        → AppUser

MembershipTransaction（消费流水）
  ├── membership_id     → LearnerMembership
  ├── session_id        → Session（可选）
  └── booking_id        → Booking（可选）
```

### 4.2 表定义

#### `membership_products`

```sql
id              bigserial primary key
brand_id        bigint not null references brands(id)
name            varchar(100) not null
description     text
type            varchar(30) not null
                -- class_pack      通用次卡（N 次任意课）
                -- time_pass       时间卡（月卡/季卡/年卡，不限次）
                -- course_bundle   指定课程次卡
                -- pt_pack         私教包（绑定私教类型课程，不绑具体教练）
credits         int                           -- null 表示 time_pass（不限次）
duration_days   int not null                  -- 有效期天数（购买即激活起算）
price           numeric(10,2) not null
store_scope     varchar(20) not null default 'brand'
                -- brand | store_specific
course_scope    varchar(20) not null default 'all'
                -- all | course_specific
status          varchar(20) not null default 'active'  -- active | inactive
created_at      timestamptz not null default now()
updated_at      timestamptz not null default now()
```

#### `membership_product_stores`

仅 `store_scope = store_specific` 时写入，记录该产品可用的门店。

```sql
id                      bigserial primary key
membership_product_id   bigint not null references membership_products(id) on delete cascade
store_id                bigint not null references stores(id) on delete cascade
brand_id                bigint not null references brands(id)
created_at              timestamptz not null default now()

unique (membership_product_id, store_id)
```

#### `membership_product_courses`

仅 `course_scope = course_specific` 时写入，记录该产品可用的课程模板。

```sql
id                      bigserial primary key
membership_product_id   bigint not null references membership_products(id) on delete cascade
course_id               bigint not null references courses(id) on delete cascade
brand_id                bigint not null references brands(id)
created_at              timestamptz not null default now()

unique (membership_product_id, course_id)
```

#### `learner_memberships`

```sql
id                  bigserial primary key
brand_id            bigint not null references brands(id)
learner_id          bigint not null references app_users(id)
product_id          bigint not null references membership_products(id)
remaining_credits   int                        -- null=time_pass；其他类型必填
started_at          timestamptz not null       -- 激活时间（手动开卡时设定）
expires_at          timestamptz not null       -- = started_at + duration_days（冻结期不计入）
purchase_store_id   bigint references stores(id)  -- 购买门店（记录用，不参与校验）
status              varchar(20) not null default 'active'
                    -- active | expired | frozen | depleted
frozen_at           timestamptz                -- 最近一次冻结时间
freeze_days         int not null default 0     -- 累计冻结天数（到期时顺延）
created_at          timestamptz not null default now()
updated_at          timestamptz not null default now()
```

**索引：** `(learner_id, brand_id, status)`, `(product_id)`

#### `membership_transactions`

```sql
id              bigserial primary key
brand_id        bigint not null references brands(id)
membership_id   bigint not null references learner_memberships(id)
learner_id      bigint not null references app_users(id)
session_id      bigint references sessions(id)
booking_id      bigint references bookings(id)
action          varchar(20) not null
                -- purchase        手动开卡
                -- consume         预约扣减
                -- refund          取消预约退还
                -- no_show_deduct  爽约扣减
                -- freeze_adjust   冻结期顺延到期（仅 time_pass）
                -- admin_adjust    后台手动调整
delta           int not null          -- 负数=扣减，正数=增加（time_pass 固定 ±1 表示次）
balance_after   int                   -- null=time_pass
note            text
operated_by     bigint references brand_users(id)  -- 后台操作时记录操作人
created_at      timestamptz not null default now()
```

---

## 5. 功能规格 — Phase 2

### 5.1 会员卡产品管理

由 brand_admin 在 Brand Backoffice「会员服务」菜单中配置。

#### 创建产品

| 步骤 | 说明 |
|---|---|
| 1. 选类型 | class_pack / time_pass / course_bundle / pt_pack |
| 2. 填基础信息 | 名称（必填）、描述、价格（必填）、有效期天数（必填） |
| 3. 填课次数 | 非 time_pass 必填；time_pass 无此字段 |
| 4. 选门店范围 | 「全品牌通用」or「指定门店」（多选） |
| 5. 选课程范围 | 仅 course_bundle / pt_pack 显示；「所有课程」or「指定课程」（多选）。若已选指定门店，课程列表只展示所选门店 `is_available=true` 的课程 |
| 6. 保存 | 同时写入 membership_products + membership_product_stores + membership_product_courses |

#### 编辑产品

可修改名称、描述、价格、有效期、门店/课程范围（全量替换关联表）。

**注意**：已开出的 learner_memberships 不受影响（持卡快照继承开卡时的 scope，不跟随产品变更）。

#### 下架产品

状态改为 `inactive`，不可再为学员开新卡，已持有的卡继续有效。

### 5.2 学员开卡与持卡

#### 手动开卡（Phase 2）

receptionist 或 brand_admin 在学员详情页操作：
1. 选择产品（只展示 status=active 的产品）
2. 确认价格（系统显示产品价格，可填备注）
3. 确认后创建 learner_memberships（status=active，started_at=now()，expires_at=now()+duration_days）
4. 写入 membership_transactions（action=purchase，delta=+credits）

#### 学员查看持卡（Learner App）

「我的权益」页展示：
- 卡名称、类型
- 剩余次数 / 到期时间
- 卡状态（使用中 / 已冻结 / 已到期 / 已用完）
- 使用记录明细（消费/退还流水）

### 5.3 预约用卡校验

学员预约 Session（session.store_id=S，session.course_id=C）时：

```
1. 查找学员有效卡：
   status=active AND expires_at > now()
   AND (credits > 0 OR type=time_pass)

2. 按门店范围过滤：
   product.store_scope = 'brand'
   OR (store_scope = 'store_specific' AND S IN membership_product_stores)

3. 按课程范围过滤：
   product.course_scope = 'all'
   OR (course_scope = 'course_specific' AND C IN membership_product_courses)

4. 若无匹配卡 → 返回 422「无可用卡券，请先购买课程包」，不创建预约

5. 若有多张匹配卡 → 优先使用到期日最近的卡（先到期先用）

6. 扣减：remaining_credits -1，写 membership_transactions（action=consume，delta=-1）
   time_pass 不减 remaining_credits（null），流水 delta 固定 -1 表示计一次

7. 若 remaining_credits = 0 → 卡 status 改为 depleted
```

### 5.4 卡券消费与退还

#### 取消预约退还

学员取消预约（在允许时间内）时：

1. 查找该 Booking 对应的消费流水（action=consume）
2. 将对应 LearnerMembership 的 remaining_credits +1
3. 写入 membership_transactions（action=refund，delta=+1）
4. 若卡状态为 depleted 且 remaining_credits > 0 → status 改回 active
5. time_pass 同样写退还流水（delta=+1），不影响 remaining_credits

**课次取消**（Brand Admin 取消整个 Session）时，后端批量对所有 confirmed Booking 执行上述退还逻辑。

#### 爽约扣减

Coach 在后台标记学员爽约（no_show）时：
- 若学员持有已消费次数的有效卡：写入 membership_transactions（action=no_show_deduct，delta=-1，无退还）
- 不退还已消费的那次，等同于消耗一次课次
- 无卡时仅记录爽约，不阻止下次预约

### 5.5 卡冻结与解冻

适用场景：学员受伤/出行等特殊情况申请暂停卡有效期。

| 操作 | 说明 |
|---|---|
| 冻结 | brand_admin / receptionist 操作；status 改为 frozen，记录 frozen_at；冻结期内不可用于预约 |
| 解冻 | 手动解冻；status 改为 active；expires_at 顺延 freeze_days 天（= 冻结至今的天数） |

冻结期间学员不可预约；不影响已有预约，但如需取消需手动操作。

---

## 6. API 端点概览

### Brand Backoffice（`/api/v1/brand`）

#### 会员卡产品管理

```
POST   /membership-products                  创建产品
GET    /membership-products                  产品列表
GET    /membership-products/:id              产品详情
PUT    /membership-products/:id              编辑产品
POST   /membership-products/:id/deactivate   下架产品
```

#### 学员持卡管理

```
POST   /learners/:learner_id/memberships     手动开卡
GET    /learners/:learner_id/memberships     学员持卡列表
POST   /memberships/:id/freeze               冻结
POST   /memberships/:id/unfreeze             解冻
PUT    /memberships/:id/credits              后台手动调整次数（admin_adjust）
```

#### 流水查询

```
GET    /memberships/:id/transactions         流水明细
```

### Learner App（`/api/v1/app`）

```
GET    /memberships                          我的持卡列表
GET    /memberships/:id/transactions         我的使用记录
```

---

## 7. 状态流转

### MembershipProduct 状态

```
active ──(brand_admin 下架)──> inactive
```

### LearnerMembership 状态

```
active ──(次数耗尽)──────────> depleted
active ──(到期)──────────────> expired（Cron Job 每日扫描）
active ──(brand_admin 冻结)──> frozen ──(解冻)──> active（expires_at 顺延）
depleted / expired ─────────> 不可恢复（次数调整走 admin_adjust）
```

---

## 8. 与其他模块的关系

### 与 course_store_availability 的关系

| 层级 | 控制什么 |
|---|---|
| `course_store_availability` | 哪些课程可以在哪些门店**排课**（排课层） |
| `membership_product_stores` + `membership_product_courses` | 哪些卡可以用于哪些门店和课程的**预约消费**（卡券层） |

两者独立控制，互不干扰。某课程在门店 A 可排课，但某张卡不一定能在门店 A 消费，取决于卡的 scope 配置。

### 与 booking 模块的关系

- 预约创建（POST /book）→ 触发 5.3 用卡校验 → 扣减 credits
- 预约取消（DELETE /bookings/:id）→ 触发退还逻辑（5.4）
- 课次取消（POST /sessions/:id/cancel）→ 批量退还所有预约对应的 credits
- 爽约标记（PUT /bookings/:id/no-show）→ 触发爽约扣减（5.4）

### 与 RBAC 的关系

| 操作 | 权限码 |
|---|---|
| 查看会员卡产品/学员持卡 | membership:read |
| 创建/编辑产品、手动开卡、冻结/解冻、调整次数 | membership:write |

---

## 9. 分期交付计划

### Phase 2（本文档范围）

1. 数据库迁移：membership_products / membership_product_stores / membership_product_courses / learner_memberships / membership_transactions
2. 会员卡产品管理 API（CRUD）
3. 手动开卡 API
4. 预约用卡校验（接入 SCHEDULE_BOOKING §5.3.2 预约流程）
5. 取消预约退还次数
6. 课次取消批量退还
7. 爽约扣减（衔接 Phase 1 爽约逻辑）
8. 卡冻结 / 解冻
9. Learner App「我的权益」页面
10. Cron Job：每日扫描到期卡，expires_at ≤ now() 且 status=active → 改为 expired

### Phase 3（商业化）

11. 在线购卡（微信小程序/抖音小程序支付）
12. 退款流程（原路退回）
13. 优惠码 / 折扣活动

---

## 10. 已确认决策

| 决策点 | 结论 | 影响 |
|---|---|---|
| D-M01 产品规格 | 固定课次，分开建产品 | 20节/30节/40节各为独立产品，各自定价；不支持购买时自选节次 |
| D-M02 门店范围 | 两档：brand / store_specific | store_specific 时用 membership_product_stores 关联表 |
| D-M03 课程范围 | 两档：all / course_specific | course_specific 时用 membership_product_courses 关联表 |
| D-M04 不绑教练 | 卡层级不绑 coach_id | 通过绑定课程模板实现"指定教练课"效果；教练变更不影响卡有效性 |
| D-M05 购买门店 | 仅记录，不参与校验 | purchase_store_id 用于报表，校验只看 product scope |
| D-M06 无卡不可约 | Phase 2 起必须持有有效卡 | 无匹配卡返回 422，引导购卡；不再提供免费兜底 |
| D-M07 Phase 2 购卡方式 | 后台手动开卡（线下收款） | 在线支付为 Phase 3；Phase 2 由 receptionist / brand_admin 操作 |
