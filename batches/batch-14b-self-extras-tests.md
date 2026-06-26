# Batch 14b 测试场景 —— C 端学员自助增量（我的权益 + 上课记录 + 加入候补）

> 契约：[batch-14b-self-extras.md](batch-14b-self-extras.md)。**C 端**：api-app :8082、app :3003、filter `@mini-schedule/app`。
> 混合执行（镜像 14a）：happy 走 app UI；候补落库(operated_by NULL/audit learner/position)/多状态 filter/ownership 走 API + **psql 实查**。
> 沿用 14a 测试身份 alice/bob（dev openid `dev_<code>`），brand21，门店 讯美广场 loc1。

## 前置数据
- alice 已桥接（profile 14）；brand owner `18816820405 / admin123` 给 alice 发 active 权益（次卡）。
- scheduled 场次：S_open（有空位）、S_full（capacity 占满，allow_waitlist 默认开）、S_attended（alice 已约且 brand 侧标到课=已到课）、S_noshow（alice 已约 + 结束场次 + 确认爽约=已爽约）。
- bob 桥接但无权益（测无权益/ownership）。

## Happy（app UI 优先）

| # | 步骤 | 预期 |
|---|---|---|
| H1 | 「我的」页 | 显示 hub 入口：我的权益 / 上课记录（+ 退出登录） |
| H2 | 点「我的权益」 | 列 alice 权益卡片：产品名 + 剩余/总(或不限次) + 到期 + 状态 badge；空则「暂无权益」 |
| H3 | 下单消耗后回「我的权益」 | 剩余即时刷新（`app-entitlements` 14a 已预埋失效） |
| H4 | 点「上课记录」 | 列终态：S_attended「已到课」+ 消耗权益、S_noshow「已爽约」；不含 booked/cancelled |
| H5 | 课程表 S_full 行 | 显示「加入候补」（非 disabled「已满」） |
| H6 | 点「加入候补」 | 成功 toast；**psql** `waitlist_entries`(status=waiting、**operated_by NULL**、position 递增)、`operation_logs`(actor_type=`learner`, actor_id=alice profile) |
| H7 | 我的预约页「候补中」筛选 | 列 alice 候补：课程/时间/门店 + 位置 position |
| H8 | 候补卡片「取消候补」确认 | entry→cancelled；**psql** status=cancelled；列表移除 |

## 桥接锚点（API + psql + brand 交叉）

| # | 场景 | 预期 |
|---|---|---|
| B1 | alice 加入 S_full 候补后，**brand 侧** S_full 候补 drawer | 见该 entry（同一 profile 14、operated_by 空） |
| B2 | brand 给 alice profile 发的权益，C 端「我的权益」 | 可见（同一 profile） |

## Edge（API + psql）

| # | 场景 | 预期 |
|---|---|---|
| E1 | 未满场次 S_open 加入候补 | `WAITLIST_SESSION_NOT_FULL` 409 |
| E2 | 已约该场次再加入候补 | `BOOKING_DUPLICATE` 409 |
| E3 | 重复加入同场次候补 | `WAITLIST_DUPLICATE` 409 |
| E4 | waitlist_limit 满 | `WAITLIST_FULL` 409 |
| E5 | 不允许候补场次（allow_waitlist=false override） | `WAITLIST_NOT_ALLOWED` 409 |
| E6 | bob（无权益）加入候补 | 成功（候补不锁权益，§22.4）——权益在转正时才校验 |
| E7 | 冻结学员加入候补 | `LEARNER_NOT_BOOKABLE` 409 |
| E8 | **ownership**：alice 取消 bob 的候补 entry id | `WAITLIST_ENTRY_NOT_FOUND` 404（不泄漏） |
| E9 | **ownership**：alice GET /waitlist 不含 bob 候补 | 仅本人 |
| E10 | 上课记录多状态：`GET /bookings?status=attended,no_show` | 仅返回这两终态；`status=attended` 单值仍可用 |
| E11 | 我的权益 settle：过期权益（expires_at 过去） | 状态显「已过期」（读触发 settle 落库） |

## DB 单测（newMigratedTestDB，v12 无新 migration）
- `TestLearnerWaitlist_Join`：self-service join → operated_by NULL + audit learner + position；staff Join（13d）operated_by=actor 回归。
- `TestLearnerWaitlist_JoinEdges`：未满/已约/重复/limit/不允许/冻结复用 13d 校验。
- `TestLearnerWaitlist_ListByLearner`：仅本 profile 活跃候补，按时间序。
- `TestLearnerWaitlist_CancelOwnership`：本人取消成功 / 取消他人 → WAITLIST_ENTRY_NOT_FOUND。
- `TestBooking_ListFilterStatuses`：Statuses=[attended,no_show] IN 过滤；单 Status 仍生效（brand 回归）。
- `TestLearnerEntitlements_ListSettles`：过期权益读后落库 expired。

## psql 实查
```sql
SELECT id, class_session_id, brand_learner_profile_id, status, position, operated_by FROM waitlist_entries WHERE brand_learner_profile_id=? ORDER BY id;
SELECT actor_type, actor_id, action FROM operation_logs WHERE action IN ('waitlist_joined','waitlist_cancelled') ORDER BY id DESC LIMIT 5;
SELECT id, status, remaining_credits, expires_at FROM learner_entitlements WHERE brand_learner_profile_id=?;
SELECT id, status FROM bookings WHERE brand_learner_profile_id=? AND status IN ('attended','no_show');
```

## 执行方式
api-app `CONFIG_PATH=configs/config-app.yaml` 重建重启 :8082；app `rm -rf apps/app/.next && pnpm --filter @mini-schedule/app dev` :3003 + curl 预热。Happy 走 app UI（chrome-devtools），桥接锚点/Edge/落库/ownership 走 API + **psql**。DB 单测 `newMigratedTestDB`（v12，无新 migration）。除 e2e 外必跑 prod `pnpm --filter @mini-schedule/app build`。复跑 13c–13e + 13d 回归。
