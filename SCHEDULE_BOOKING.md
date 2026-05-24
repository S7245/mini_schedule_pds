# 课程预约模块 — 产品设计规格（PDS）

整理日期：2026-05-24
状态：Phase 1 草稿，待评审

---

## 目录

1. [背景与目标](#1-背景与目标)
2. [术语对齐](#2-术语对齐)
3. [用户角色与使用场景](#3-用户角色与使用场景)
4. [数据模型](#4-数据模型)
5. [功能规格 — Phase 1](#5-功能规格--phase-1)
   - 5.1 门店与场地管理
   - 5.2 排课管理
   - 5.3 学员端预约
   - 5.4 候补队列
   - 5.5 签到确认
   - 5.6 预约规则配置
   - 5.7 通知
6. [API 端点概览](#6-api-端点概览)
7. [状态流转](#7-状态流转)
8. [分期交付计划](#8-分期交付计划)
9. [已确认决策](#9-已确认决策)
10. [待确认问题](#10-待确认问题)

---

## 1. 背景与目标

### 1.1 背景

Mini Schedule 当前 MVP 已具备品牌入驻、Learner 管理、课程管理（仅模板）、训练记录等基础能力。

下一个核心模块是**课程预约系统**，目标是让：

- Brand Administrator 能够排课（将课程模板实例化为具体的上课时间）；
- Learner 能够提前预约课次，并查看预约记录；
- Coach 能够确认学员签到。

### 1.2 目标

| 指标 | 目标 |
|---|---|
| 学员端可以完成预约全流程 | Phase 1 交付 |
| Brand Administrator 可以完成排课全流程 | Phase 1 交付 |
| 支持多门店数据隔离 | Phase 1 数据模型就绪 |
| 候补学员可及时收到升级通知 | Phase 1 交付 |

### 1.3 范围说明

本文档覆盖 **Phase 1** 范围。会员卡/次卡消耗、私教预约、在线支付等属于 Phase 2/3，不在本文约束范围内。

---

## 2. 术语对齐

在 `CONTEXT.md` 基础上，本模块新增以下术语：

| 术语 | 含义 |
|---|---|
| Store（门店） | Brand 旗下的实体经营场所；一个 Brand 可以有多个 Store |
| Room（场地） | Store 内的物理空间，有名称和容量限制（最大人数） |
| CourseTemplate（课程模板） | 课程的基础定义，描述课程内容，不绑定时间；对应现有 `courses` 表；归属品牌级，不挂 store_id |
| CourseStoreAvailability（课程门店可用性） | 控制某个 CourseTemplate 在哪些 Store 可以被用于排课的关联关系；默认不可用，需显式启用 |
| Session（课次） | 基于 CourseTemplate 的一次具体排课，有确定的开始时间、教练和场地 |
| Booking（预约） | Learner 对某个 Session 的预约记录 |
| Waitlist（候补） | Session 满员后的候补队列，按加入顺序排序 |
| CheckIn（签到） | Learner 实际出勤的确认动作，由 Coach 在后台手动操作 |
| BrandStaff（品牌员工） | Brand 旗下所有工作人员的统称，含 Brand Administrator 和 Coach |
| Coach（教练） | BrandStaff 中负责授课并在后台确认签到的角色 |

用词注意：

- 避免将 Session 称为 Class、Event 或直接用 Course 表示课次。
- 避免将 Booking 称为 Reservation 或 Appointment。
- 避免将 Store 称为 Branch、Location、Shop。

---

## 3. 用户角色与使用场景

### 3.1 Brand Administrator

| 场景 | 说明 |
|---|---|
| 管理门店 | 创建、编辑门店基础信息（名称、地址） |
| 管理场地 | 在门店内创建、编辑场地（名称、容量） |
| 单次排课 | 选择课程模板、日期时间、教练、场地，创建一个 Session |
| 循环排课 | 设置重复规则，批量生成多个 Session |
| 修改/取消课次 | 编辑已排课次的时间、教练或场地；取消课次并通知已预约学员 |
| 查看课表 | 按门店、日期、教练筛选课次列表 |
| 配置预约规则 | 设置可提前预约窗口、最晚取消时间、候补上限等 |

### 3.2 Coach（教练）

| 场景 | 说明 |
|---|---|
| 查看我的课次 | 查看自己负责的 Session 列表 |
| 确认签到 | 在 Session 签到列表中手动标记学员已到课 |
| 标记爽约 | 标记未出现的学员为爽约 |

### 3.3 Learner

| 场景 | 说明 |
|---|---|
| 浏览课程 | 查看品牌当前可预约的 Session 列表，支持按日期/类型筛选 |
| 预约课次 | 选择 Session 并确认预约 |
| 加入候补 | Session 满员时加入候补队列 |
| 候补确认 | 收到候补升级通知后，在限时内确认是否上课 |
| 取消预约 | 在允许时间内取消已预约的 Session |
| 查看我的预约 | 查看即将上课、历史记录、已取消的预约列表 |

---

## 4. 数据模型

### 4.1 实体关系概览

```
Brand
  └── Store（门店，1..n）
        └── Room（场地，1..n）

Brand
  └── CourseTemplate（课程模板，0..n）
        └── CourseStoreAvailability（可用门店，0..n）→ Store

Session（课次）
  ├── brand_id       → Brand
  ├── store_id       → Store
  ├── room_id        → Room
  ├── course_id      → CourseTemplate
  └── coach_id       → BrandUser（教练角色）

Booking（预约）
  ├── session_id     → Session
  └── learner_id     → AppUser

Waitlist（候补）
  ├── session_id     → Session
  └── learner_id     → AppUser

CheckIn（签到）
  ├── booking_id     → Booking
  └── confirmed_by   → BrandUser（教练）

BookingRule（预约规则）
  └── brand_id / store_id → Brand / Store（品牌级或门店级）
```

### 4.2 表定义（迁移文件目标）

#### `stores`

```sql
id              bigserial primary key
brand_id        bigint not null references brands(id)
name            varchar(100) not null
address         text
phone           varchar(20)
status          varchar(20) not null default 'active'  -- active / inactive
created_at      timestamptz not null default now()
updated_at      timestamptz not null default now()
```

#### `rooms`

```sql
id              bigserial primary key
store_id        bigint not null references stores(id)
brand_id        bigint not null references brands(id)
name            varchar(100) not null
capacity        int not null default 10
description     text
status          varchar(20) not null default 'active'
created_at      timestamptz not null default now()
updated_at      timestamptz not null default now()
```

#### `sessions`

```sql
id              bigserial primary key
brand_id        bigint not null references brands(id)
store_id        bigint not null references stores(id)
room_id         bigint not null references rooms(id)
course_id       bigint not null references courses(id)
coach_id        bigint not null references brand_users(id)
starts_at       timestamptz not null
ends_at         timestamptz not null
capacity        int not null            -- 本次课次容量，默认继承 Room 容量，可独立调整
booked_count    int not null default 0  -- 已预约人数，冗余字段加速查询
status          varchar(20) not null default 'scheduled'
                -- scheduled / in_progress / completed / cancelled
cancel_reason   text
recurring_id    bigint                  -- 循环排课批次 ID（null 表示单次排课）
created_at      timestamptz not null default now()
updated_at      timestamptz not null default now()
```

**索引：** `(brand_id, starts_at)`, `(coach_id, starts_at)`, `(room_id, starts_at)`

#### `bookings`

```sql
id              bigserial primary key
session_id      bigint not null references sessions(id)
learner_id      bigint not null references app_users(id)
brand_id        bigint not null references brands(id)
status          varchar(20) not null default 'confirmed'
                -- confirmed / cancelled / no_show
cancelled_at    timestamptz
cancel_reason   text
created_at      timestamptz not null default now()
updated_at      timestamptz not null default now()

unique (session_id, learner_id)
```

#### `waitlists`

```sql
id              bigserial primary key
session_id      bigint not null references sessions(id)
learner_id      bigint not null references app_users(id)
brand_id        bigint not null references brands(id)
position        int not null            -- 候补顺序（1 = 队首）
status          varchar(20) not null default 'waiting'
                -- waiting / notified / confirmed / expired / cancelled
notified_at     timestamptz             -- 发出候补升级通知的时间
confirm_deadline timestamptz            -- 限时确认截止时间
created_at      timestamptz not null default now()
updated_at      timestamptz not null default now()

unique (session_id, learner_id)
```

#### `check_ins`

```sql
id              bigserial primary key
booking_id      bigint not null unique references bookings(id)
session_id      bigint not null references sessions(id)
learner_id      bigint not null references app_users(id)
confirmed_by    bigint not null references brand_users(id)  -- Coach
checked_in_at   timestamptz not null default now()
```

#### `booking_rules`

```sql
id              bigserial primary key
brand_id        bigint not null references brands(id)
store_id        bigint references stores(id)   -- null 表示品牌级规则
book_ahead_hours_max  int not null default 168  -- 最多提前多少小时预约（默认 7 天）
book_ahead_hours_min  int not null default 1    -- 最少提前多少小时预约
cancel_deadline_hours int not null default 4   -- 最晚取消时间（开课前 N 小时）
waitlist_max          int not null default 5   -- 最多候补人数
waitlist_confirm_minutes int not null default 30  -- 候补确认限时（分钟）
no_show_penalty       boolean not null default true   -- 爽约后自动扣减学员对应次卡次数
created_at      timestamptz not null default now()
updated_at      timestamptz not null default now()

unique (brand_id, store_id)
```

#### `recurring_sessions`

```sql
id              bigserial primary key
brand_id        bigint not null references brands(id)
store_id        bigint not null references stores(id)
room_id         bigint not null references rooms(id)
course_id       bigint not null references courses(id)
coach_id        bigint not null references brand_users(id)
rrule           text not null   -- 重复规则描述（如 "WEEKLY;BYDAY=MO,WE,FR;COUNT=8"）
start_date      date not null
start_time      time not null
duration_min    int not null
capacity        int not null
status          varchar(20) not null default 'active'
created_at      timestamptz not null default now()
updated_at      timestamptz not null default now()
```

#### `course_store_availability`

CourseTemplate 归属品牌级，通过此表控制哪些门店可以用该课程排课。创建课程时 brand_admin 选择可用门店（默认全选），对应记录写入此表（`is_available = true`）。无记录或 `is_available = false` 表示该门店不能使用此课程排课。

```sql
id              bigserial primary key
brand_id        bigint not null references brands(id)
course_id       bigint not null references courses(id) on delete cascade
store_id        bigint not null references stores(id) on delete cascade
is_available    boolean not null default true
created_at      timestamptz not null default now()
updated_at      timestamptz not null default now()

unique (course_id, store_id)
```

**索引：** `(course_id)`, `(store_id)`, `(brand_id)`

**默认行为：**
- brand_admin 创建课程时，UI 提供门店多选（默认全选当前所有有效门店）
- 新增门店后，不自动继承已有课程可用性，需 brand_admin 手动配置
- store_manager 查看可用课程列表时，后端只返回该门店 `is_available = true` 的课程

---

## 5. 功能规格 — Phase 1

### 5.1 门店与场地管理

#### 门店管理（Brand Backoffice）

| 操作 | 说明 |
|---|---|
| 创建门店 | 填写名称（必填）、地址、电话 |
| 编辑门店 | 更新基础信息，不影响已排课次 |
| 停用门店 | 门店状态改为 `inactive`，不可再对该门店新建课次；已排未完成的课次保持有效 |
| 查询门店列表 | 返回当前 Brand 下所有门店，支持分页 |

#### 场地管理（Brand Backoffice）

| 操作 | 说明 |
|---|---|
| 创建场地 | 填写名称（必填）、容量（必填，≥1）、所属门店（必填） |
| 编辑场地 | 更新名称/容量；容量变更不影响已排课次的 `capacity` 字段 |
| 停用场地 | 场地状态改为 `inactive`；已排未完成课次保持有效 |
| 查询场地列表 | 按门店筛选，返回当前 Brand 下的场地 |

#### 课程门店可用性管理（Brand Backoffice）

由 brand_admin 在课程详情页配置，控制该课程可在哪些门店排课。

| 操作 | 说明 |
|---|---|
| 创建课程时配置可用门店 | 课程创建表单中包含门店多选组件，默认全选所有有效门店；提交时为每个选中门店写入 `course_store_availability` 记录 |
| 编辑可用门店 | 在课程详情页可随时增减可用门店；移除门店后，该门店已排的历史 Session 不受影响 |
| store_manager 查看可用课程 | 排课时课程选择器仅展示本门店 `is_available = true` 的课程，不可见其他课程 |

**冲突检测**：创建/编辑 Session 时，后端校验同一 Room 在所选时间段内（starts_at, ends_at）是否已有状态为 `scheduled` 或 `in_progress` 的课次，有则返回错误。

### 5.2 排课管理

#### 5.2.1 单次排课

请求参数：

| 字段 | 必填 | 说明 |
|---|---|---|
| store_id | 是 | 门店 |
| room_id | 是 | 场地，需属于该门店 |
| course_id | 是 | 课程模板，需属于该 Brand |
| coach_id | 是 | 教练，需属于该 Brand |
| starts_at | 是 | 开始时间（ISO8601，含时区） |
| ends_at | 是 | 结束时间，须 > starts_at |
| capacity | 否 | 本次容量，默认取 Room.capacity |

后端校验：

1. Room 时间冲突检测。
2. Coach 时间冲突检测（同一 Coach 同时间是否已有课次）。
3. starts_at 必须在当前时间之后。
4. ends_at > starts_at。
5. 课程门店可用性校验：`course_store_availability` 中存在 `course_id = ? AND store_id = ? AND is_available = true`，否则返回 422「该课程未在目标门店启用，请先在课程设置中添加该门店」。

#### 5.2.2 循环排课

请求参数：

| 字段 | 必填 | 说明 |
|---|---|---|
| store_id / room_id / course_id / coach_id | 是 | 同单次排课 |
| start_date | 是 | 第一次上课日期 |
| start_time | 是 | 每次开始时间（HH:MM，24 小时制） |
| duration_min | 是 | 课时长度（分钟） |
| capacity | 否 | 每次容量 |
| repeat_weekdays | 是 | 重复的星期，数组，如 [1,3,5] 表示周一三五 |
| repeat_weeks | 是 | 重复多少周，1–52 |

后端处理：

1. 将重复规则存入 `recurring_sessions`，生成 `recurring_id`。
2. 按规则生成 N 个 Session，每个 Session 的 `recurring_id` 指向同一批次。
3. 每个 Session 独立进行 Room/Coach 冲突检测；冲突的日期跳过并在响应中告知。

#### 5.2.3 课次编辑与取消

| 操作 | 说明 |
|---|---|
| 修改单个课次 | 可修改 starts_at / ends_at / coach_id / room_id / capacity；已预约学员有人时，变更时间/场地须触发通知 |
| 取消单个课次 | Session 状态改为 `cancelled`；已预约学员的 Booking 状态改为 `cancelled`；候补学员的 Waitlist 状态改为 `cancelled`；通知所有已预约学员 |
| 批量取消循环课次 | 可选"仅此课次"或"此课次及之后"，批量取消并通知 |

#### 5.2.4 课表查询

支持以下筛选维度：

- `store_id`：按门店过滤
- `coach_id`：按教练过滤
- `date_from / date_to`：日期范围
- `status`：课次状态

响应中每个 Session 包含：booked_count、capacity、waitlist_count。

### 5.3 学员端预约

#### 5.3.1 浏览可预约课次

Learner App 展示当前 Brand 下状态为 `scheduled` 的 Session 列表，并满足：

- starts_at > now()
- starts_at 在 `book_ahead_hours_max` 小时内

每个 Session 展示：课程名、教练名、场地名、开始时间、时长、容量、已预约人数、剩余名额、是否有候补位。

#### 5.3.2 预约

流程：

1. Learner 选择 Session，点击预约。
2. 后端校验：
   - Session 状态为 `scheduled`。
   - starts_at > now() + book_ahead_hours_min。
   - starts_at < now() + book_ahead_hours_max。
   - booked_count < capacity（否则返回满员错误，引导加入候补）。
   - Learner 没有该 Session 的有效 Booking（去重）。
   - Learner 在同一时段没有其他有效 Booking（时间冲突）。
3. 创建 Booking（status: `confirmed`），booked_count +1。
4. 发送预约成功通知。

Phase 1 预约为免费，不消耗会员卡/次卡。

#### 5.3.3 取消预约

1. Learner 发起取消。
2. 后端校验：
   - Booking 属于该 Learner。
   - Booking 状态为 `confirmed`。
   - starts_at > now() + cancel_deadline_hours（否则返回"已超过最晚取消时间"错误）。
3. Booking 状态改为 `cancelled`，booked_count -1。
4. 触发候补升级流程（见 5.4）。

爽约处理：Booking status 改为 `no_show`，若学员持有有效次卡则自动扣减 1 次；无次卡时仅记录，不阻止下次预约（Phase 1）。

#### 5.3.4 我的预约列表

返回 Learner 的 Booking 列表，分三个 Tab：

- **即将上课**：status=`confirmed`，starts_at > now()
- **历史记录**：starts_at <= now()，含 completed / no_show
- **已取消**：status=`cancelled`

### 5.4 候补队列

#### 加入候补

1. Learner 在满员 Session 详情页点击"加入候补"。
2. 后端校验：
   - Session 满员（booked_count >= capacity）。
   - 当前候补人数 < waitlist_max。
   - Learner 没有该 Session 的有效 Booking 或有效候补记录。
3. 创建 Waitlist（status: `waiting`，position = 当前队列长度+1）。

#### 候补升级流程

触发时机：某个 Booking 被取消（booked_count < capacity）时。

1. 查找该 Session 的候补队列中 position 最小且 status=`waiting` 的记录。
2. 将该 Waitlist status 改为 `notified`，记录 `notified_at`，设置 `confirm_deadline = now() + waitlist_confirm_minutes 分钟`。
3. 发送候补升级通知给该 Learner，告知需在 `confirm_deadline` 前确认。

#### 候补确认

1. Learner 收到通知后，在 App 内点击"确认参加"。
2. 后端校验：
   - Waitlist status=`notified`。
   - now() < confirm_deadline（未超时）。
3. 创建 Booking（status: `confirmed`），booked_count +1。
4. Waitlist status 改为 `confirmed`。

#### 候补超时

需要一个定时任务（Cron Job）：

1. 每分钟扫描 status=`notified` 且 `confirm_deadline < now()` 的 Waitlist 记录。
2. 将这些记录的 status 改为 `expired`。
3. 触发下一位候补的升级流程（递归，直到有人确认或候补队列耗尽）。

#### 取消候补

Learner 可以主动退出候补队列（status 改为 `cancelled`）。

### 5.5 签到确认

Coach 在 Brand Backoffice 进入课次详情，看到已预约学员列表：

| 字段 | 说明 |
|---|---|
| 学员姓名/昵称 | |
| 预约时间 | |
| 签到状态 | 未签到 / 已签到 / 爽约 |
| 操作 | 确认到课 / 标记爽约 |

操作逻辑：

- **确认到课**：创建 CheckIn 记录，Booking 状态不变（仍为 `confirmed`）；若 CheckIn 已存在则返回已签到错误。
- **标记爽约**：Booking status 改为 `no_show`；若学员持有有效次卡则自动扣减 1 次。

Coach 只能操作自己负责的 Session（后端按 coach_id 过滤）。

Session 开课后（starts_at 已过）才允许操作签到。Session status 为 `cancelled` 时不允许操作。

### 5.6 预约规则配置

Brand Administrator 在 Brand Backoffice 中配置预约规则，支持品牌级默认规则，未来可按门店覆盖。

Phase 1 可配置项：

| 配置项 | 字段 | 默认值 | 约束 |
|---|---|---|---|
| 最多提前预约时长 | book_ahead_hours_max | 168（7 天） | 1–720 小时 |
| 最少提前预约时长 | book_ahead_hours_min | 1 | 0–24 小时 |
| 最晚取消时间 | cancel_deadline_hours | 4 | 0–48 小时 |
| 候补上限 | waitlist_max | 5 | 0–20 |
| 候补确认限时 | waitlist_confirm_minutes | 30 | 5–120 分钟 |

Brand 初始化时自动创建一条默认品牌级 BookingRule（store_id=null）。

### 5.7 通知

Phase 1 通知渠道：**微信小程序服务通知** + **抖音小程序服务通知**（两个渠道，需对应 AppID 配置）。

各渠道按学员已绑定的小程序平台发送；若学员同时绑定两个平台则两个渠道均发送。

| 触发事件 | 通知对象 | 模板 |
|---|---|---|
| 预约成功 | Learner | "你已成功预约 {课程名}，{时间}，{门店} {场地}" |
| 候补加入成功 | Learner | "你已加入 {课程名} 候补队列，当前排名第 {N} 位" |
| 候补升级 | Learner | "你的 {课程名} 候补名额升级成功，请在 {时间} 前确认参加，否则自动顺延" |
| 候补超时未确认 | Learner | "你的 {课程名} 候补名额已过期" |
| 预约取消成功 | Learner | "你的 {课程名} 预约已取消" |
| 课次取消 | 所有已预约 Learner | "{课程名} 课次（{时间}）已取消" |
| 开课前提醒 | 已预约 Learner | "{课程名} 将于 1 小时后开始，请准时到达 {门店} {场地}" |

开课前提醒由 Cron Job 在 starts_at - 1 小时触发。

---

## 6. API 端点概览

### Brand Backoffice（`/api/v1/brand`）

#### 门店管理

```
POST   /stores                  创建门店
GET    /stores                  门店列表
GET    /stores/:id              门店详情
PUT    /stores/:id              编辑门店
POST   /stores/:id/deactivate   停用门店
```

#### 场地管理

```
POST   /stores/:store_id/rooms  创建场地
GET    /stores/:store_id/rooms  场地列表
PUT    /rooms/:id               编辑场地
POST   /rooms/:id/deactivate    停用场地
```

#### 排课管理

```
POST   /sessions                创建单次课次
POST   /sessions/recurring      创建循环课次
GET    /sessions                课次列表（支持筛选）
GET    /sessions/:id            课次详情
PUT    /sessions/:id            编辑课次
POST   /sessions/:id/cancel     取消课次
GET    /sessions/:id/bookings   查看已预约学员列表
POST   /sessions/:id/check-in   Coach 确认签到
PUT    /bookings/:id/no-show    Coach 标记爽约
```

#### 课程管理

```
GET    /courses/:id/store-availability         查看课程可用门店列表
PUT    /courses/:id/store-availability         更新课程可用门店（全量替换）
```

#### 预约规则

```
GET    /booking-rules           查看当前预约规则
PUT    /booking-rules           更新预约规则
```

### Learner App（`/api/v1/app`）

#### 课次浏览与预约

```
GET    /sessions                可预约课次列表
GET    /sessions/:id            课次详情（含名额/候补状态）
POST   /sessions/:id/book       预约课次
DELETE /bookings/:id            取消预约
POST   /sessions/:id/waitlist   加入候补
DELETE /waitlists/:id           退出候补
POST   /waitlists/:id/confirm   候补确认
```

#### 我的预约

```
GET    /bookings                我的预约列表（by status）
GET    /waitlists               我的候补列表
```

---

## 7. 状态流转

### Session 状态

```
scheduled ──(时间到达)──> in_progress ──(结束)──> completed
    │
    └──(Brand Admin 取消)──> cancelled
```

### Booking 状态

```
confirmed ──(取消预约/课次取消)──> cancelled
confirmed ──(Coach 标记)──────> no_show
confirmed + CheckIn ─────────> 视为已签到（CheckIn 单独记录，Booking 状态不变）
```

### Waitlist 状态

```
waiting ──(触发升级)──> notified ──(Learner 确认)──> confirmed
                              │
                              └──(超时)──> expired ──(触发下一位)──> ...
waiting / notified ──(Learner 主动退出)──> cancelled
waiting ──(课次取消)──> cancelled
```

---

## 8. 分期交付计划

### Phase 1 — 预约核心（本文档范围）

1. 多门店 + 场地数据模型（DB 迁移文件）
2. 排课管理：单次排课、循环排课、编辑/取消
3. 学员端预约/取消（免费，无需消耗卡券）
4. 候补队列 + 限时确认升级（候补确认限时 30 分钟，含 Cron Job）
5. Coach 后台手动签到确认；爽约标记自动扣减对应次卡次数
6. 预约规则配置
7. 通知：微信小程序服务通知 + 抖音小程序服务通知（两个渠道，需对应 AppID）
8. 私教预约（1v1 时间协商流程，与团课同期交付）

### Phase 2 — 增强

8. 会员卡/次卡/课包体系（预约消耗权益）
9. 员工多角色权限控制（Coach / Receptionist / Operations）
10. 代客预约（Brand Staff 帮学员预约）
11. 微信小程序/抖音小程序通知接入（需对应 AppID）

### Phase 3 — 商业化

13. 在线支付购卡（微信支付）
14. 品牌套餐计费（Platform 层）
15. 数据报表（预约率/上座率/收入趋势）
16. 学员训练计划（教练制定）

### Phase 4 — 增值

17. 课程/教练评价系统
18. 训练打卡/成就徽章
19. 学员二维码签到
20. AI 课程推荐
21. 直播课

---

## 9. 已确认决策

| 决策点 | 结论 | 影响 |
|---|---|---|
| 多门店支持时机 | Phase 1 同期做 | 数据模型从一开始含 Store/Room 层级，避免后期迁移成本 |
| 会员卡/计费 | Phase 1 免费预约 | 不创建消耗流水，简化 Phase 1 预约流程 |
| 签到方式 | Coach 在后台手动确认 | 无需扫码设备，最简实现 |
| 候补升级方式 | 通知 + 限时确认（30 分钟）| 需要 Cron Job 处理超时；Learner 须主动确认，防止资源浪费 |
| 私教预约 | Phase 1 包含 | Session 模型需支持 private 类型，1v1 时间协商与团课同期交付 |
| 通知渠道 | 微信小程序服务通知 + 抖音小程序服务通知 | 需对应 AppID 配置；两个渠道按学员绑定平台发送 |
| 候补确认限时 | 30 分钟（固定，不可配置） | waitlist_confirm_minutes 字段保留但 Phase 1 UI 不暴露配置入口 |
| 爽约惩罚 | 扣减次卡次数 | 标记爽约时自动扣减有效次卡 1 次；无次卡时仅记录 |
| 过渡收款方案 | Phase 3 前接受线下/手动收款 | Phase 1 不需要"手动开课包"操作入口 |
| D-10 课程模板归属层级 | 品牌级 + 门店可用性控制 | `courses` 表保持 brand_id，新增 `course_store_availability` 表控制哪些门店可用哪些课程；排课时校验课程是否在目标门店启用 |

---

## 10. 待确认问题

本文档所有关键决策已确认，无阻塞性待定问题。后续如有新增决策点，在此追加。
