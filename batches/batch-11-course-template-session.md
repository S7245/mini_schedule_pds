# Batch 11：课程模板 + 单场次排课（CourseTemplate / CourseCategory / ClassSession）

状态：契约待 approve（会话内确认）

## 0. 范围与决策（grill 定论）

闭环主题：**品牌配置课程分类 → 建课程模板（绑分类 + 选可用门店）→ 在可用门店排出单场次（教练时间冲突被拦）→ 课程表可见**，并点亮 onboarding 第 4/5/7 步。

| # | 决策 | 结论 |
|---|---|---|
| D1 | 范围切分 | 本批：CourseCategory + CourseTemplate（含分类绑定 + CourseLocationAvailability）+ 单场次 ClassSession（create/list/detail/cancel + 教练时间冲突）。**推迟 Batch 12**：RecurringSchedule 循环批量排课、Location Resource 资源管理（及资源冲突）、Booking/Waitlist/Attendance |
| D2 | 课程难度/类型字段 | migration 把 `courses.difficulty` / `courses.type` **DROP NOT NULL**；新表单丢弃健身枚举，改通用「级别」`level_label`（自由文本）。贴合通用 SaaS 定位。api-app 只读路径不受影响（null→空串） |
| D3 | legacy `/courses` | 用新 CourseTemplate 管理页 + 带 RBAC 门的新 handler **原地替换** brand 端 `/courses` route，退役 legacy brand 写接口；**api-app（C 端只读 ListPublished）完全不动**。状态词沿用 `draft/published/archived`，把 onboarding count 的 `status='active'` 改为 `'published'` 对齐 |
| D4 | 额度门 | 按 blueprint §4.1「第一版不限制课程/场次数量」，CourseTemplate / ClassSession **不挂 SubscriptionGuard** |
| D5 | 数据权限 | session/course/category 列表与详情接 Batch 6 ScopeResolver：all_brand（owner fast-path）/ assigned_locations（按 location_id 过滤 session）。`own_sessions`（教练只看自己）本批不实现 → 转 FR |
| D6 | 门店删除 guard | 把 class_sessions（status scheduled/in_progress）纳入 `CountActiveReferences`，落实 Batch 9 挂的 FR |

## 1. 契约

### 1.1 数据库（migration 000007）

```sql
-- 1) 放开 courses 健身枚举（D2）
ALTER TABLE courses ALTER COLUMN difficulty DROP NOT NULL;
ALTER TABLE courses ALTER COLUMN type       DROP NOT NULL;

-- 2) courses.status CHECK（此前无约束）：draft/published/archived
--    （DO $$ 包裹，幂等）
ALTER TABLE courses ADD CONSTRAINT courses_status_valid
    CHECK (status IN ('draft','published','archived'));

-- 3) 细粒度 permission（沿用 Batch 5 约定，与既有 coarse 码并存）
--    course.view 已在 000003 存在 → ON CONFLICT DO NOTHING 复用
INSERT INTO permissions(code,domain,action,name,description) VALUES
  ('course_category.view',   'course_category','view',  '查看课程分类',''),
  ('course_category.create', 'course_category','create','新增课程分类',''),
  ('course_category.edit',   'course_category','edit',  '编辑课程分类',''),
  ('course.create','course','create','新增课程模板',''),
  ('course.edit',  'course','edit',  '编辑课程模板',''),
  ('course.delete','course','delete','删除课程模板',''),
  ('session.view',  'session','view',  '查看课程场次',''),
  ('session.create','session','create','排课/新增场次',''),
  ('session.cancel','session','cancel','取消场次','')
ON CONFLICT (code) DO NOTHING;

-- 4) role_template 映射 + 5) 存量 brand backfill（镜像 000006）
--    brand_owner / brand_admin / course_operator：全部 9 码
--    location_manager：course_category.view, course.view, session.view/create/cancel
--    instructor / receptionist：session.view
```

`session.cancel` 不在 Expand 自动隐含表（只有 edit/create→view、delete→view+edit），故给到 cancel 的角色须同时显式 seed `session.view`。

### 1.2 后端 API

所有路径前缀 `/api/v1/brand`，JWT brand 中间件已提供 brandID/userID。错误统一走 `response.Error` + `AppError`。

**课程分类 CourseCategory**

| 方法 | 路径 | 权限门 | 请求字段 | 响应 |
|---|---|---|---|---|
| GET | /course-categories | course_category.view | status? | items[]{id,name,color,icon,sort_order,show_in_mini_program,status} |
| POST | /course-categories | course_category.create | name, color?, icon?, sort_order?, show_in_mini_program? | category |
| PATCH | /course-categories/:id | course_category.edit | name?, color?, icon?, sort_order?, show_in_mini_program?, status? | category |

唯一约束 `course_categories(brand_id,name)` → `CATEGORY_NAME_DUPLICATED`(409)。

**课程模板 CourseTemplate（route /courses 替换 legacy）**

| 方法 | 路径 | 权限门 | 请求字段 | 响应 |
|---|---|---|---|---|
| GET | /courses | course.view | status?, q?(title ILIKE), category_id?, page,page_size | items[]{id,title,level_label,duration_min,default_capacity,status,categories[],available_location_count}, total |
| GET | /courses/:id | course.view | — | course + category_ids[] + available_location_ids[] |
| POST | /courses | course.create | title, description?, cover_url?, level_label?, duration_min, default_capacity, category_ids[], location_ids[](可用门店；空=默认全选当前 active 门店), show_in_mini_program? | course（status=draft） |
| PATCH | /courses/:id | course.edit | 上述白名单字段（category_ids/location_ids 全量替换） | course |
| PATCH | /courses/:id/status | course.edit | status: draft\|published\|archived | course（published→published_at） |
| DELETE | /courses/:id | course.delete | — | 204；软删 |

校验：`category_ids` 必属本 brand 且 active 否则 `CATEGORY_NOT_FOUND`；`location_ids` 必属本 brand active 门店；DELETE 时若有 scheduled/in_progress 场次引用 → `COURSE_IN_USE`(409)。CourseLocationAvailability 由 create/update 的 location_ids 内联维护（硬删重插 active 行，镜像 staff_location_assignments 做法）。

**课程场次 ClassSession（route /class-sessions）**

| 方法 | 路径 | 权限门 | 请求字段 | 响应 |
|---|---|---|---|---|
| GET | /class-sessions | session.view | location_id?, course_id?, instructor_profile_id?, status?, from?, to?(时间窗), page,page_size | items[]{id,course_title,location_name,instructor_name,starts_at,ends_at,capacity,booked_count,status}, total |
| GET | /class-sessions/:id | session.view | — | session 详情 |
| POST | /class-sessions | session.create | course_id, location_id, instructor_profile_id, starts_at, ends_at, capacity?(默认 course.default_capacity), waitlist_limit? | session（status=scheduled） |
| PATCH | /class-sessions/:id/cancel | session.cancel | cancel_reason? | session（status=cancelled） |

场次创建校验（application 层，单事务）：
1. course 属本 brand 且 `status='published'`，否则 `COURSE_NOT_ACTIVE`。
2. location 属本 brand 且 active；course 在该 location 可用（course_location_availability.is_available），否则 `COURSE_LOCATION_UNAVAILABLE`。
3. instructor_profile 属本 brand 且 `is_schedulable AND status='active'`，否则 `INSTRUCTOR_NOT_SCHEDULABLE`。
4. `ends_at > starts_at` 且 `starts_at > now`，否则 `SESSION_TIME_INVALID`。
5. 直接落 `status='scheduled'`（跳过 draft，使 DB EXCLUDE 教练不重叠约束 + onboarding count 即时生效）。
6. 触发 `class_sessions_instructor_no_overlap` EXCLUDE 违反 → 捕获 SQLSTATE `23P01` → `SESSION_INSTRUCTOR_CONFLICT`(409)，**不裸透 500**。
7. data_scope：assigned_locations 时 location 必须在 scope 内，否则 404。

cancel：仅 `scheduled`/`in_progress` 可取消，否则 `SESSION_CANCEL_NOT_ALLOWED`。本批无 booking，cancel 只改状态 + 写 OperationLog（学员退预约/释放权益 Batch 13+）。

**门店删除 guard 扩展**：`CountActiveReferences` 增加 `class_sessions WHERE location_id=? AND status IN ('scheduled','in_progress')`。

**审计**：course_created/updated/status_changed/deleted、category_created/updated、session_created/cancelled 写 OperationLog（audit.Write）。

**新增错误码（~10）**：`CATEGORY_NOT_FOUND`(404) `CATEGORY_NAME_DUPLICATED`(409) `COURSE_NOT_FOUND`(404) `COURSE_NOT_ACTIVE`(409) `COURSE_IN_USE`(409) `COURSE_LOCATION_UNAVAILABLE`(409) `SESSION_NOT_FOUND`(404) `SESSION_TIME_INVALID`(400) `SESSION_INSTRUCTOR_CONFLICT`(409) `SESSION_CANCEL_NOT_ALLOWED`(409) `INSTRUCTOR_NOT_SCHEDULABLE`(409)。

### 1.3 前端页面模块（apps/brand）

| 页面/模块 | 类型 | 关键字段/操作 | 权限门 |
|---|---|---|---|
| `/course-categories` | 页面 | 表格(name/color 圆点/sort/status) + 状态筛选；新增/编辑 Dialog；状态切换 | course_category.* |
| `/courses` | 页面（替换 legacy） | 表格(title/分类 chips/级别/时长/默认容量/可用门店数/status badge) + 状态筛选 + name 搜索；行操作 编辑/发布·归档(status)/删除(ConfirmDialog, COURSE_IN_USE toast) | course.* |
| 课程模板 Dialog | 弹窗 | RHF+Zod：title, description, cover_url, level_label, duration_min, default_capacity, 分类多选 chip, 可用门店多选(默认全选 active), show_in_mini_program | — |
| `/courses/[id]` | 页面（替换 legacy） | 详情：基础信息 + 分类 + 可用门店 + 近期场次列表 | course.view |
| `/schedule` | 页面 | 场次列表/课程表(表格优先)：筛选 门店/课程/教练/状态/日期窗；行 取消(ConfirmDialog+原因) | session.* |
| 排课 Dialog | 弹窗 | course 选择器(按所选 location 的可用课程过滤) + 教练选择器(schedulable) + 日期+开始时间+时长→ends_at + 容量(默认课程默认容量)；SESSION_INSTRUCTOR_CONFLICT/COURSE_LOCATION_UNAVAILABLE inline+toast | — |
| onboarding 第 4/5/7 步 | 步骤页 | StepPlaceholder→真实 CTA：跳 /course-categories、/courses、/schedule；完成态实时反映 | — |
| 导航 | 菜单 | 「课程分类」「课程模板」「排课」入口 + `NAV_HREF_PERMISSIONS` 门 | 对应 view |

### 1.4 前端实现约束

- 复用 /web 现有组件（DataTable、ConfirmDialog、Dialog、Hint、usePermissions、分页、status badge）与 location/staff 既有模式，不引入新 UI 库。
- 复用 `packages/api` client + errors.ts；新增 `course-categories.ts` / `courses.ts` / `class-sessions.ts` hooks + `packages/types` 类型。
- 仅改 brand 端；admin/app 不动（api-app legacy 只读保持）。
- 任何前端会 `.map()` 的数组字段后端不 `omitempty`（Batch 5 铁律）。

## 2. 不在本批（→ Batch 12+ / FR）

- RecurringSchedule 循环批量排课 + recurring_schedule_weekdays。
- Location Resource CRUD + 资源时间冲突（session.location_resource_id 本批恒 null）。
- Booking / Waitlist / Attendance / EntitlementHold（学员预约批次）。
- `own_sessions` 数据权限（教练只看自己授课场次）。
- 场次改期（reschedule）；本批 cancel + 重排。

## 3. 验收闭环（owner=18816820405/admin123, brand 21）

登录 owner → 建分类「团课」→ 建课程模板「晨间瑜伽」(选分类+默认全选门店, 默认容量 8) → 发布(draft→published) → /schedule 排单场次(选门店+课程+教练+明早 9:00, 60min) → 课程表出现 scheduled 场次 → 同教练同时段再排一节 → 被 SESSION_INSTRUCTOR_CONFLICT 拦 → onboarding 第 4/5/7 步 completed。
