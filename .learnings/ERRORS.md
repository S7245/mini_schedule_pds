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
