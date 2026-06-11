## 2026-06-06 Pending — batch 文件命名 / 链接 / 时间戳一致性

扫描 `pds/batches/` 现状：

| 文件 | 配套 tests 文件 | 头部状态行 | 头部"更新时间" vs 标题里日期 |
|---|---|---|---|
| `batch-01-public-signup.md` | `batch-01-public-signup-tests.md` ✓ | `状态：实现中`（已实际完成，未回填） | `2026-05-29` |
| `batch-02-wechat-pay.md` | **缺失** ✗ 无 `batch-02-wechat-pay-tests.md` | `状态：已完成 ✅（2026-05-29）` | `更新时间：2026-05-28` 与完成日期不一致 |
| `batch-03-wechat-callback.md` | `batch-03-wechat-callback-tests.md` ✓ | `状态：契约草稿 ⏳` | `2026-06-02`（PROGRESS 已标 ✅ 但契约文件状态未回填） |

### Pending 整改项（待用户确认后再统一改）

1. **batch-02 缺测试文件**：CLAUDE.md 步骤 4 强制要求每个 Batch approve 后写 `tests.md`，但 batch-02 跳过了这一步直接被标完成。需补写 `batch-02-wechat-pay-tests.md` 或在 CLAUDE.md 中显式说明"无前端 Edge case 的纯接口 Batch 可豁免"。
2. **状态字段未跟随 PROGRESS 回填**：batch-01 状态停留在"实现中"、batch-03 停留在"契约草稿"，但 PROGRESS.md 中均已 ✅。两份文档作为 source of truth 出现漂移。建议每次 PROGRESS 状态变更同步回写 batch 文件头部（或反过来用脚本/hook 把 batch 文件头部当成唯一 source）。
3. **"更新时间"与状态变更日期不绑定**：batch-02 显示 `更新时间：2026-05-28` 但完成时间是 `2026-05-29`。该字段语义模糊（创建？最后改动？完成？）。建议拆成 `created / approved / completed` 三个明确字段，参考 LEARNINGS 2026-06-06 关于 lifecycle 标签的建议。
4. **契约文件与 tests 文件的双向链接缺失**：batch-03 契约第 347 行只单向引用 tests 文件；tests 文件没回链契约。后续生成新 Batch 时建议固定首段写一对互链。

## 2026-06-11 Batch 6 踩坑 — RBAC enforcement + data_scope 落地

### 1. 验收邮件含糊带过未完成项（流程级，已被用户纠正）

- 现象：T07 / T10 实际未完成，但验收/收尾邮件里被含糊措辞带过，让 blocked / skipped 项看起来像已闭环。
- 纠正：用户要求收尾时逐项标 done / skipped / blocked，并区分"主动选择跳过"vs"真卡住"。
- 修复方式：步骤 10 验收邮件 + PROGRESS 回填一律带 task 级三态表；skipped 写理由 + 是否转 FR，blocked 写卡点 + 解锁条件。详见 LEARNINGS 同日 §1。
- Pending exposure：需回查本批是否已把 T07 / T10 的真实状态如实写进 PROGRESS.md 与契约文件头部，避免 source of truth 再次漂移（参考本文件 2026-06-06 Pending §2 状态漂移）。

### 2. subagent 被 403 kill 后留下 test 编译错误，主线程接管时才发现

- 现象：后端 subagent 多次被 403 中断，崩溃点可能停在"签名改了一半 / import 缺失 / mock 未补全"，导致 `go test` 连编译都过不了（红都跑不起来）。
- 修复方式：主线程接管后先 `go build ./... && go vet ./...` 复位编译态，再续下一个 task；不要假设上个 task 留下的是干净的红或绿。
- Pending exposure：被中断 task 周边的 test 文件可能残留半改的 mock / fixture，续跑时若只看主代码绿不看 test 文件，可能把假绿当真绿。建议接管后对最后一个 commit 涉及的 `*_test.go` 单独 review。

### 3. 验收期前端 bug：disabled tooltip 不显示 + 跨用户缓存泄漏（未被自动测试覆盖）

- 现象一：受 RBAC 控制的按钮 disabled 后，tooltip / 提示文案不显示，用户不知道为何点不动（权限拒绝缺 UI 反馈）。
- 现象二：切换账号（A 登出 → B 登录）后，A 的权限/数据在前端残留（缓存隔离失败）。
- 共同根因：契约的测试场景只覆盖 happy path，没有"权限拒绝 UI 反馈"和"账号切换/缓存隔离"两条交互路径，自动测试自然漏。
- 修复方式：§4.2 测试场景模板新增两组强制 Edge Case（权限拒绝 UI 反馈 / 账号切换缓存隔离），凡 RBAC 相关 Batch 必含。详见 LEARNINGS 同日 §3。
- Pending exposure：前端缓存泄漏需排查所有持久层（前端 store / react-query cache / localStorage）在登出时是否全清；本批只修了暴露出来的那一处，其他受权限影响的页面可能同样残留，建议全端 grep 登出清理逻辑。

### 4. data_scope 越权语义（404 vs 403）拖到实现期才拍板

- 现象：越权访问越权资源应返 404（不泄漏存在性）还是 403，grill / 契约都没定，实现时才明确为 404。
- 风险：若不同接口实现者各自理解，可能一部分返 403（泄漏了"资源存在但你无权"）、一部分返 404，安全语义不一致。
- 修复方式：把"越权读/写各返什么状态码、是否泄漏存在性"提为 grill 必答根决定点，结论写进契约错误码列。详见 LEARNINGS 同日 §4。
- Pending exposure：需全量核对本批所有受 data_scope 约束的接口是否统一为 404，不能有漏网的 403；建议 grep 所有越权分支的返回码做一次一致性审计。
