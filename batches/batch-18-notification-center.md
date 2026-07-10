# Batch 18 契约 —— 员工/品牌后台站内通知中心（§20.13）

## Lifecycle

| 阶段 | 时间戳 | 触发人 |
|---|---|---|
| draft | 2026-07-10 | claude |
| approved | 2026-07-10（4 决策点会话内拍板，全取推荐项） | user |
| implementing | 2026-07-10（T1–T5 后端 + Stage B 前端，9 commits） | claude |
| static-verified | 2026-07-10（go test ./... 38 包绿；pnpm --filter brand build 通过） | claude |
| scenario-verified | 2026-07-10（TestEmitEndToEnd_* 真实 DB：owner 收/无角色 staff 不收/actor 排除） | claude |
| done | 2026-07-10 | claude |

## 范围与决策（grill 后 4 决策点，全取推荐）

1. **事件范围 v1 = 仅预约域 5 类**：新预约 / 取消预约 / 候补变化 / 待确认爽约 / 场次取消。事件源全在已完成的 booking/waitlist/session 域（13c–15）。商业化事件（订阅异常 / 额度预警）留 FR。
2. **集成方式 = 事务后 best-effort 异步 emit**：业务 tx 提交成功后在 application 层 emit，失败仅记日志，绝不回滚业务（§20.13「失败不影响主流程」）。
3. **data_scope 本店过滤本批就做**：emit fan-out 时按 `assigned_locations` 过滤——店长/前台/instructor 只收本店场次事件，owner/admin 收全品牌。复用 RBAC `DataScope`。
4. **migration 000013 仅 seed 权限码 + per-user 落行**：复用**已存在**的 `notifications` 表（000003:1112，per-user 收件箱形态），不建表、不改表结构；只 seed `notification.view` / `notification.mark_read` 权限码 + 角色映射（镜像 000009）。

> ⚠️ 学员端微信订阅消息（§7.5）**不在本批**（卡 per-brand 小程序模板审核，留 FR）。本批只做**后台站内通知**（channel=`in_app`）。

## 现状核查（grounded in code，交接假设已修正）

- **表已存在**：`notifications`（`migrations/000003_course_booking_schema.up.sql:1112`），字段 `brand_id / recipient_user_type[platform_admin|brand_user|learner] / recipient_id / channel[in_app|wechat_subscribe] / event_type / title / body / status[pending|sent|failed|read] / related_type / related_id / sent_at / read_at / error_message`；已建索引 `idx_notifications_recipient_lookup(recipient_user_type, recipient_id, status, created_at DESC)`。→ schema 已是 per-user fan-out，交接文件「零通知系统 / 必建新表」的假设不成立，migration 面显著缩小。
- **零 Go 代码**引用该表（`grep -ril notification internal/**/*.go` 空）→ 新建 `domain/notification` 全链。
- **emit 范式**：镜像 `internal/audit`（`tx.Table("operation_logs").Create(map)` 规避循环依赖）。
- **recipient 定向原语**：RBAC `LoadEffectiveRaw(brandID, userID)→(codes, DataScope{Kind, LocationIDs}, isOwner)`；staff `List(ScopeLocationIDs)`。
- **事件源写事务**：`booking_repository.go` Create/CreateByLearner/Cancel/CancelByLearner/EndSession/EndSessionSystem；`waitlist_repository.go` Join/Promote/Cancel；`class_session_repository.go` Cancel。booking model 带 `class_session_id` → join 出 location_id + starts_at 供 data_scope + 文案。

## 事件 → 收件人 → 文案 映射

| event_type | 触发点（application 层，tx 提交后） | related_type / id | 收件人（brand_user，按 location scope + 排除 actor 本人） | 标题 |
|---|---|---|---|---|
| `booking_created` | 新预约（Create / CreateByLearner） | booking / booking_id | 该场次 location 内有 `notification.view` 的员工 | 新预约 |
| `booking_cancelled` | 取消预约（Cancel / CancelByLearner） | booking / booking_id | 同上 | 预约取消 |
| `waitlist_changed` | 候补变化（Join / Promote / Cancel） | waitlist_entry / id | 同上 | 候补变化 |
| `attendance_pending_noshow` | 待确认爽约（EndSession / EndSessionSystem 产出待确认爽约） | class_session / session_id | 同上 | 待确认爽约 |
| `session_cancelled` | 场次取消（ClassSession.Cancel 级联） | class_session / session_id | 同上 | 场次取消 |

**收件人解析规则**（emit 时 fan-out）：
1. 枚举本品牌 active brand_user；
2. 逐人 `LoadEffectiveRaw` → 有 `notification.view` 且（`isOwner` ∨ `Kind==all_brand` ∨（`Kind==assigned_locations` ∧ 事件 location ∈ `LocationIDs`））→ 命中；
3. 排除 actor 本人（actor 为 brand_user 时；learner 自助操作不排除任何员工）；
4. 每个命中收件人插一行：`recipient_user_type='brand_user'`, `recipient_id=brandUserID`, `channel='in_app'`, `status='sent'`, `sent_at=now`。
5. 空收件人集 = 静默 no-op（best-effort，不报错）。

## 契约

### API 接口（`interfaces/brand`，recipient_id 恒 = 当前登录 brand_user，data_scope 已在 emit 时落定，读侧只按 recipient_id 过滤）

| 方法 | 路径 | 请求字段 | 响应字段 | 权限门 |
|---|---|---|---|---|
| GET | `/api/v1/brand/notifications` | `status`(unread\|read\|all，默认 all) / `page` / `page_size` | `items[]{id, event_type, title, body, status, related_type, related_id, read_at, created_at}` / `total` / `unread` | `notification.view` |
| GET | `/api/v1/brand/notifications/unread-count` | — | `count` | `notification.view` |
| POST | `/api/v1/brand/notifications/:id/read` | — | `id, status, read_at` | `notification.mark_read` |
| POST | `/api/v1/brand/notifications/read-all` | — | `updated` | `notification.mark_read` |

> 读侧不再做 location data_scope（fan-out 已按 scope 落行，越店事件根本不会落到该 user）；标记已读须校验 `recipient_id = 当前 user`（防越权改他人通知）。

### 前端页面模块（brand app）

| 页面/模块 | 类型 | 关键字段/操作 |
|---|---|---|
| 通知消息页 `/notifications` | 页面 | Tab（全部/未读/已读）+ 列表（标题/正文/事件类型/时间/未读点）+ 单条「标记已读」+「全部已读」，复用 Card/Table/Tabs/Badge，不引新库 |
| 顶栏未读徽标 | 组件 | 轮询 `unread-count`（TanStack Query，30s），未读数 >0 显红点数字，点击进 `/notifications` |
| 导航入口 | 菜单 | 「通知消息」入口，`notification.view` 门控（无权限不显示） |

### migration 000013（仅权限，复用已存在 notifications 表）

- up：`INSERT permissions('notification.view','notification.mark_read')` + `role_template_permissions` 映射到**全部后台角色模板**（brand_owner/brand_admin/course_operator/finance_support/location_manager/receptionist/instructor/location_assistant——§14.2 品牌菜单 + §14.3 员工视图均含通知）+ `brand_role_permissions` backfill 存量品牌（镜像 000009 三步）。
- down：逆序 DELETE 三层映射 + permission 行。表结构不动。

## 前端实现约束

- 复用 `/web/apps/brand` 既有组件与布局（Card/Table/Select/Badge/Tabs），不引入新 UI 库。
- 不跨端改动（app/admin 不动）。
- 未读徽标轮询用现有 TanStack Query 惯例，不引 websocket。

## 任务拆解（plan，TDD 逐 task commit）

**Stage A — 后端**（完成跑 `/code-review`）
- **T1** migration 000013 up/down（权限 seed）；DB 测试：空库顺序 up 到 13 + down 干净。
- **T2** `domain/notification`：Entity + `EventType` 枚举（5 值）+ `Recipient` + `Repository` 接口 + `RecipientResolver` 接口 + 校验；单测。
- **T3** `persistence/notification_repository`（List/UnreadCount/MarkRead/MarkAllRead + `Emit` 批量落行）+ `RecipientResolver` 实现（staff 枚举 + rbac scope 过滤）；DB 单测（scope 过滤 / actor 排除 / 分页 / 标记已读越权拒绝）。
- **T4** `application/notification` Emitter + 注入 5 触点（learnerbooking / booking / waitlist / sessionautomation / classsession service），tx 提交后 best-effort（同步 Emit + 调用点 `go` 包裹）；DB 单测断言每事件落行 + scope + actor 排除；**复跑 13c–15 全 DB 单测证零回归**。
- **T5** `interfaces/brand` handler（4 端点）+ 路由 + RBAC 门 + `make wire` 重生成；handler 测试。

**Stage B — 前端**（完成跑 `/code-review`）
- **T6** `packages/api/notifications.ts` + types。
- **T7** brand `/notifications` 页（Tab + 列表 + 标记已读/全部已读）+ 导航入口门控。
- **T8** 顶栏未读徽标（轮询 unread-count）。
- **T9** `pnpm --filter brand build` + 静态验证。

**验收**：起 api-brand 连 dev DB → 触发 5 类事件 → psql 实查 `notifications` 落行正确（scope/actor 排除）→ owner 收全品牌、店长只收本店 → 未读徽标 + 标记已读闭环 → 越权角色（无 notification.view）`PERMISSION_DENIED`、越店不可见。
