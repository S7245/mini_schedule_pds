# Batch 15 测试场景 —— 场次状态自动化（asynq）

> 契约：[batch-15-session-automation.md](batch-15-session-automation.md)。
> 本批后端为主，无 UI 改动 → 不跑 Playwright。验证两路：**DB 单测**（真实 Postgres `newMigratedTestDB`，可控 `now`）+ **主线程 live 验收**（起 worker + 造「过去 ends_at」数据 + psql 实查）。

## Happy Path

| # | 步骤 | 预期结果 |
|---|---|---|
| H1 | 造场次 A：`status=scheduled`、`starts_at` 过去、`ends_at` 未来（进行中窗口）。跑一轮 sweep | A → `in_progress`（无 audit 写入，纯显示态） |
| H2 | 造场次 B：`status=scheduled`、`ends_at` 过去、含 1 条 `booked` 未签到 booking。跑 sweep | B → `completed`；该 booking → `pending_no_show`；`booked_count` 不变；hold 不动；**不**产 `no_show` |
| H3 | H2 后查 `operation_logs` | 有一行 `action=session_ended`、`actor_type='system'`、`actor_id` 为 NULL、`brand_id` = B 的品牌、`metadata.after.pending_no_show_count=1` |
| H4 | 造场次 C：`status=in_progress`、`ends_at` 过去。跑 sweep | C → `completed`（守卫允 `in_progress→completed`），其 `booked` 同样转 `pending_no_show` |
| H5 | 造场次 D：`status=scheduled`、`starts_at`+`ends_at` 均过去（短课，从未 in_progress）。跑 sweep | D **直接** → `completed`（`ListDueSessionIDs` 覆盖 scheduled+到点；不经 in_progress 中间态） |
| H6 | sweep 一轮处理含 A(转 in_progress)+B/C/D(转 completed) 的混合批 | `RunSweep` 返回 `Summary{started:1, ended:3, skipped, failed:0}`，handler 记一行日志 |
| H7 | 手动 override：H1 后对 A（in_progress）手动点「结束场次」(`POST /class-sessions/:id/end`) | A → `completed`，13e 手动入口在自动化后仍可用 |

## Edge Cases

| # | 场景 | 预期结果 |
|---|---|---|
| E1 | **幂等**：H2 后再跑一轮 sweep | B 仍 `completed`（`EndSessionSystem` 状态守卫 → 空操作）；**无**新增 `session_ended` audit；booking 仍 `pending_no_show`，无重复结算 |
| E2 | **未到点不动**：场次 `scheduled`、`starts_at` 未来 | sweep 后仍 `scheduled`（`MarkSessionsInProgress` 与 `ListDueSessionIDs` 均不命中） |
| E3 | **进行中未到点不结束**：场次 `in_progress`、`ends_at` 未来 | sweep 后仍 `in_progress`（`ends_at <= now` 不成立） |
| E4 | **已取消免疫**：场次 `cancelled`、`ends_at` 过去 | sweep 不动它（`MarkSessionsInProgress` 限 `scheduled`；`ListDueSessionIDs` 限 `scheduled/in_progress`） |
| E5 | **已完成免疫**：场次 `completed` | sweep 不动；`EndSessionSystem` 对 completed 返 `SESSION_NOT_ENDABLE`（被 skipped 计数，不报错中断 sweep） |
| E6 | **§22.6 边界**：H2 的 B 结束后，其 booking 停在 `pending_no_show` | 自动化**绝不**把 booking 推到 `no_show`、绝不扣课/退课（hold 原样）；扣课须 staff 手动 `ConfirmNoShow` |
| E7 | **占位预约**：B 含一条无 hold 的占位 `booked`（requires_entitlement_fix） | 同样 `booked→pending_no_show`，无结算（与有 hold 的 booked 一致；EndSession 只动 status） |
| E8 | **跨品牌**：两个不同 brand 各有到点场次 | 单次 sweep 全部处理（`ListDueSessionIDs` 跨品牌，`EndSessionSystem` 从行读 `brand_id`，各自 audit 带对应 brand_id） |
| E9 | **`booked_count` 不变**：B 有 2 booked | 结束后 `booked_count` 仍为 2（结束/签到/爽约不退名额，仅取消退） |
| E10 | **单场次失败隔离**：sweep 批中一条 EndSession 报错（模拟） | 其余场次照常处理；该条 log+continue，下一 tick 自愈；systemic 查询失败才返 error 触发 asynq 重试 |
| E11 | **TX-3 零回归**：自动结束产生的 `completed` 场次再尝试 `Cancel` | 返 `SESSION_CANCEL_NOT_ALLOWED`（`completed` 不可取消，状态机闭合，TX-3 不触及 `pending_no_show`） |

## DB 单测清单（新增）

| 测试 | 覆盖 |
|---|---|
| `TestMarkSessionsInProgress` | scheduled+到点→in_progress；未到点/completed/cancelled 不动；多行批量；幂等（重跑 RowsAffected=0） |
| `TestListDueSessionIDs` | scheduled+到点 / in_progress+到点 命中；future/completed/cancelled 不命中；跨品牌全返 |
| `TestEndSessionSystem` | scheduled→completed + booked→pending_no_show（actor=system / audit actor_id NULL）；in_progress→completed；completed→`SESSION_NOT_ENDABLE`；不动 booked_count；不动 hold；不产 no_show；占位预约 |
| `TestRunSweep`（app service） | 注入固定 now，断言 `Summary{started,ended,skipped,failed}` 计数 + 编排顺序（先 mark 后 end） |

## 零回归复跑（actor 参数化后必须全绿）
- 13e：`TestEndSession_*`、`TestConfirmNoShow_*`、`TestAttend_*`
- 13c：`TestBooking_*`、`TestSessionCancel_*`
- 13d：`TestWaitlist_*`
- `go build ./... && go test ./...`（全包绿，含新 `cmd/worker` 编译）

## 执行方式
1. **DB 单测**：`cd backend && go test ./internal/infrastructure/persistence/... ./internal/application/sessionautomation/...`（真实 Postgres，`now` 注入参数化，无墙钟依赖）。
2. **Live 主线程验收**（= 业务验收）：
   - `CONFIG_PATH=configs/config-app.yaml go run ./cmd/worker`（连 dev DB v12 + Redis）。
   - psql / brand API 造 H1（过去 starts/未来 ends）+ H2（过去 ends + booked 未签到）数据。
   - 等 1 个 sweep tick（`@every 1m`）或临时把 `sweep_cron` 调短。
   - psql 实查 H1–H3 + E1 幂等 + E6 §22.6 边界，逐条核对。
