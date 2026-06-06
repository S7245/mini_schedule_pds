## 2026-06-06 验收宽严度：契约/测试设计 ≠ 业务验证通过

**现象**：Batch 3（微信支付回调）在 `PROGRESS.md` 中标 ✅，但：

- `batch-03-wechat-callback-tests.md` 列出 19 个场景（H1-H8 + E1-E19），文件内没有任何"实际执行/通过/失败"的记录。
- 多数 Edge case 标注"集成测试待跑"或依赖未实现的 mock 注入钩子（E18 / E19）。
- 验收清单（契约文件第 367-382 行）里"真实微信回调能成功验签""BrandSubscription 快照时间范围正确"等业务项没有勾选证据。

等于"测试场景文档已写"被当作"业务验证已通过"。

**风险**：

1. 后续 Batch 依赖回调正确性（如 BrandSubscription 快照 → 额度硬限制），实际 bug 会在更下游 Batch 才暴露，定位成本翻倍。
2. PROGRESS.md 失去"上线就绪状态"的指示意义。

**建议沉淀到 CLAUDE.md 的"每轮实现流程"中**：

1. 步骤 6（静态验证）和步骤 7（场景执行）之间，**强制**要求 tests.md 每条场景填入"执行状态 + 证据链接"（日志、截图、commit hash）。
2. 步骤 9 标 ✅ 前，PROGRESS 自动跑一个检查：tests.md 是否还有 `pending` / `待跑` / `mock 待补` 字样；若有，状态只能写 `scenario-partial` 而非 ✅。
3. 区分两类 Batch 完成度：
   - `contract-done`：契约+测试设计完成，可进入实现。
   - `verified-done`：所有 Happy + 关键 Edge case 实跑通过，可作为下游依赖。
   PROGRESS.md 同时显示两个里程碑日期。

## 2026-06-06 契约文件 lifecycle 字段未结构化（Batch 4 仍只在头部一行状态）

**现象**：LEARNINGS §5 已建议契约文件头部用统一 lifecycle 枚举 `draft → approved → implementing → static-verified → scenario-verified → done / parked`，每次变更补 changelog。但 Batch 4 实际仍延续旧格式——仅在头部一行写 `状态：已 approve ✅` + `approve 时间：2026-06-06`，无 changelog 块，状态机切换不可追溯。

**风险**：

1. 跨会话恢复时无法看出该批"目前卡在哪一步"——是 approve 完待写 tests？还是 tests 完待 spawn？还是已实现待验收？需要回溯多轮对话。
2. 多 Batch 并行时（如 batch-04 实现中 + batch-05 草稿中）无法用 grep 统一过滤当前活跃批。

**建议**：

1. CLAUDE.md §4.1 契约模板头部固定字段块：
   ```markdown
   ## Lifecycle

   | 阶段 | 时间戳 | 触发人 |
   |---|---|---|
   | draft | 2026-06-06 14:00 | claude |
   | approved | 2026-06-06 15:30 | user |
   | implementing | 2026-06-06 15:35 | claude |
   | static-verified | — | — |
   | scenario-verified | — | — |
   | done | — | — |
   ```
2. 每次状态变更必须同步回填表格，PROGRESS.md 只引用当前阶段，不复述。
3. 兼容已 done 的旧批：补一次 backfill PR 把 batch-01/02/03/04 lifecycle 表补全（一次性，非每批重做）。

## 2026-06-06 契约 approve 流程缺少"问题降级"机制

**现象**：batch-03 契约第 386-394 行 "用户需确认 5 问"包含了一个跨 Batch 调度问题（"异常订单补偿何时启动（下一批次）"），用户实际 approve 时只能整体 yes/no，没法说"前 4 个我答了，第 5 个推迟到下个 Batch"。

**建议**：CLAUDE.md 步骤 3（"等待用户 approve 契约"）补一句：

> 用户答复需对每个待确认问题单独标 `confirmed / deferred / rejected`。`deferred` 的问题自动迁移到 PROGRESS.md 的 Next bucket，并在契约文件保留链接，不阻塞 approve。
