# Batch 14a 业务验收自包含 prompt（C 端学员自助预约核心环）

> 复制本文件全文到新会话执行。目标：验证 C 端学员自助预约闭环 + **桥接正确性**（C 端 openid→profile 与 brand 侧操作同一 `brand_learner_profile`）。
> 契约：`pds/batches/batch-14a-self-booking.md`；测试矩阵：`pds/batches/batch-14a-self-booking-tests.md`。

## 环境（必做，旧二进制/旧 .next 掩盖改动）
```bash
# 后端 api-app :8082（C 端）
cd /Users/liushan/Documents/zkw/mini_schedule/backend
go build -o /tmp/api-app ./cmd/api-app && (旧进程先杀) /tmp/api-app &   # 需本地 Postgres + Redis + config-app.yaml
# 前端 app :3003
cd /Users/liushan/Documents/zkw/mini_schedule/web
rm -rf apps/app/.next && pnpm --filter @mini-schedule/app dev          # 起后 curl http://localhost:3003/login 预热
```
- 后端 dev DB 保持 v12（本批零 migration）。
- brand 端 :8081 / :3002 用于准备数据与交叉验证（owner `18816820405 / admin123`，brand21）。

## 数据准备（brand 侧 owner 登录）
1. /schedule 建 **scheduled 场次**（门店 讯美广场 loc1）：
   - S1 次日 09:00–10:00；S2 次日 09:30–10:30（与 S1 时段重叠）；S3 次日 11:00–12:00；S4 容量=1。
2. **先让 C 端学员登录建出 profile**（见 H1），再回 brand /learners 找到该 profile → 权益 Tab 发放：次卡（count-based，10 次）。
   ⚠️ 顺序铁律：必须「C 端先登录建 P → brand 再给 P 发权益」。反向（brand 按手机号先建档）是另一个 profile，C 端看不到。

## Happy / 桥接锚点
| # | 步骤 | 预期 |
|---|---|---|
| H1 | app :3003 `(auth)/login` 输 brand_id(默认 brand21) + 登录码 `alice` 登录 | 进首页。**psql**：`learner_identities`(wechat_open_id=`dev_alice`, phone NULL) 1 行 + `brand_learner_profiles`(brand21+该identity) 1 行 |
| H2 | 退出再用 `alice` 登录 | 同一 profile（幂等）；psql identity/profile 仍各 1 行 |
| H3 | 底部「课程表」 | 列 S1/S2/S3/S4（scheduled+未来），每行 课程/时间/门店/剩余 N + 预约 |
| H4 | S1 点「预约」→ 弹窗显示「将使用权益：次卡（剩余 10 次）」→ 确认 | 预约成功 toast；**psql** `bookings`：source=`learner_self_service`、**assisted_by NULL**、profile=alice、status=booked；`entitlement_holds` held；次卡 remaining=9/locked=1；`entitlement_transactions` operated_by NULL；`operation_logs` actor_type=`learner` |
| **B1** | **brand 侧 /bookings 查** | **见该 booking：source=learner_self_service、assisted_by 空、profile=alice（同一行）** ← 桥接核心 |
| B2 | 底部「我的预约」=即将上课 | 列 S1（已预约 + 次卡） |
| H5 | S1 行「取消」→ 确认 | 已取消 toast；**psql** cancel_source=`learner`、cancelled_by NULL、hold released、次卡 remaining=10/locked=0、booked_count-- |
| H6 | 取消后再约 S1 | 成功（partial unique 允许，新行；旧 cancelled 行保留） |

## Edge（API 直连 + psql）
| # | 场景 | 预期 |
|---|---|---|
| E1 | 约重叠场次：先约 S1，再约 S2 | `BOOKING_TIME_CONFLICT` 409；约不重叠 S3 成功 |
| E2 | 无权益学员（新登录码 `bob`，未发权益）约 S3 | `ENTITLEMENT_NONE_AVAILABLE` 409；前端「你暂无可用权益，请联系机构」 |
| E3 | 满员 S4（先用一学员占满）再约 | `SESSION_FULL` 409（前端「已满」） |
| E4 | **ownership**：alice 取消 bob 的 booking id（API 直连改 id） | `BOOKING_NOT_FOUND` 404（不泄漏） |
| E5 | **ownership**：alice 的 GET /bookings 不含 bob 的预约 | 仅本人 |
| E6 | 旧 token（手动去掉 profile_id 或用桥接前 token）调 POST /bookings | 401 需重新登录 |

## psql 实查
```sql
SELECT li.wechat_open_id, li.phone, p.id AS profile_id, p.brand_id
  FROM learner_identities li JOIN brand_learner_profiles p ON p.learner_identity_id=li.id
  WHERE li.wechat_open_id LIKE 'dev_%';
SELECT id, source, status, assisted_by, brand_learner_profile_id, cancel_source FROM bookings ORDER BY id DESC LIMIT 10;
SELECT action, delta_credits, operated_by FROM entitlement_transactions ORDER BY id DESC LIMIT 10;
SELECT actor_type, actor_id, action FROM operation_logs WHERE action IN ('learner_self_registered','booking_created','booking_cancelled') ORDER BY id DESC LIMIT 10;
```

## 通过标准
- 桥接锚点 B1 通过（C 端 booking 在 brand 侧可见、source/assisted_by/同一 profile 正确）= 本批核心。
- Happy H1–H6 + Edge E1–E6 全过；落库 psql 与预期一致。
- 回归：brand 侧代预约/代取消/签到（13c–13e）仍正常（抽查 1–2 条）。

验收结论回本会话（PASS / 问题清单），通过后我更新 PROGRESS + 三库 .learnings 并 push + dev FF main。
