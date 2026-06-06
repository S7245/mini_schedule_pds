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

## 2026-06-06 契约 approve 流程缺少"问题降级"机制

**现象**：batch-03 契约第 386-394 行 "用户需确认 5 问"包含了一个跨 Batch 调度问题（"异常订单补偿何时启动（下一批次）"），用户实际 approve 时只能整体 yes/no，没法说"前 4 个我答了，第 5 个推迟到下个 Batch"。

**建议**：CLAUDE.md 步骤 3（"等待用户 approve 契约"）补一句：

> 用户答复需对每个待确认问题单独标 `confirmed / deferred / rejected`。`deferred` 的问题自动迁移到 PROGRESS.md 的 Next bucket，并在契约文件保留链接，不阻塞 approve。
