# Batch 14b 业务验收自包含 prompt（C 端我的权益 + 上课记录 + 加入候补）

> 复制全文到新会话执行。14a 已验收 push；14b 增量。契约：`pds/batches/batch-14b-self-extras.md`；测试矩阵：`pds/batches/batch-14b-self-extras-tests.md`。

## 环境
```bash
cd /Users/liushan/Documents/zkw/mini_schedule/backend
go build -o /tmp/api-app ./cmd/api-app
lsof -ti:8082 | xargs kill -9 2>/dev/null
CONFIG_PATH=configs/config-app.yaml nohup /tmp/api-app > /tmp/api-app.log 2>&1 &   # 需 Postgres(mini_schedule)+Redis
cd /Users/liushan/Documents/zkw/mini_schedule/web
rm -rf apps/app/.next && pnpm --filter @mini-schedule/app dev   # :3003，curl 预热
```
- ⚠️ api-app 必须 `CONFIG_PATH=configs/config-app.yaml`（默认 config.yaml 跑成 :8081）。
- 沿用 14a 身份 alice（dev openid `dev_alice` → profile 14）/bob，brand21。零 migration（DB v12）。

## 数据准备（brand owner 18816820405/admin123）
- alice profile 发 active 权益（次卡）；造满员场次 S_full（capacity 占满）；给 alice 某 booking 标到课(13e)产「已到课」、另一 booking 结束场次+确认爽约产「已爽约」。

## Happy（app UI）
| # | 步骤 | 预期 |
|---|---|---|
| H1 | 「我的」页 | 见 hub 入口：我的权益 / 上课记录 |
| H2 | 点「我的权益」 | 权益卡片：产品名+剩余/总(或不限次)+状态(生效中)+到期；过期权益显「已过期」(settle) |
| H3 | 下单消耗后回我的权益 | 剩余即时刷新（app-entitlements 失效） |
| H4 | 点「上课记录」 | 列已到课/已爽约 + 消耗权益；不含 booked/cancelled |
| H5 | 课程表 S_full 行 | 显「加入候补」(非 disabled) |
| H6 | 点加入候补 | toast「已加入候补」；**psql** waitlist_entries(status=waiting, **operated_by NULL**, position), operation_logs(actor_type=learner) |
| H7 | 我的预约「候补中」筛选 | 列候补卡片(课程/时间/门店/候补第N位) |
| H8 | 候补卡片「取消候补」 | toast；**psql** status=cancelled, operated_by NULL；列表移除 |

## 桥接锚点 / Edge（API+psql）
| # | 场景 | 预期 |
|---|---|---|
| B1 | brand 侧 S_full 候补 drawer | 见 alice entry(同 profile, operated_by 空) |
| E1 | 未满场次加入候补 | WAITLIST_SESSION_NOT_FULL 409 |
| E2 | 已约场次加入候补 | BOOKING_DUPLICATE 409 |
| E3 | 重复加入 | WAITLIST_DUPLICATE 409 |
| E4 | bob(无权益)加入候补 | 成功（候补不锁权益 §22.4） |
| E5 | ownership：alice 取消 bob 候补 | WAITLIST_ENTRY_NOT_FOUND 404 |
| E6 | 上课记录多状态 GET /bookings?status=attended,no_show | 仅两终态 |

## psql
```sql
SELECT id,class_session_id,brand_learner_profile_id,status,position,operated_by FROM waitlist_entries WHERE brand_learner_profile_id=? ORDER BY id;
SELECT actor_type,actor_id,action FROM operation_logs WHERE action IN ('waitlist_joined','waitlist_cancelled') ORDER BY id DESC LIMIT 5;
SELECT id,status,remaining_credits,expires_at FROM learner_entitlements WHERE brand_learner_profile_id=?;
```

## 通过标准
- Happy H1-H8 + Edge 全过；候补 operated_by NULL + audit learner + ownership 正确（psql）。
- 回归：13d staff 候补 + 13c-13e 仍正常。prod `pnpm --filter @mini-schedule/app build` exit 0。
