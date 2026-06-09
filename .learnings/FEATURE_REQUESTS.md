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

## 2026-06-06 CLAUDE.md §4 缺 plan 阶段模板（Batch 5 已自发产出，应固化）

**现象**：Batch 5 首次在 spec approve 与 TDD spawn 之间插入独立 plan 文件 `batch-05-staff-instructor-roles-plan.md`，包含后端 13 task DAG / 前端 7 task DAG / 每 task 红绿步骤 / 风险点 + 预案。实操中后端 subagent API 403 中断，续跑 agent 仅凭该文件 + commit message 里的 task 编号即接力成功，**完全不依赖对话上下文回灌**——验证了 plan 文件作为"外部状态机"的价值。

但 CLAUDE.md §4 当前只有步骤 1-11，没把 plan 阶段单列；plan 模板也没固化，下一批可能又回到 narrative 描述损失续跑能力。

**建议沉淀到 CLAUDE.md**：

1. §4 增设步骤 5.5「plan 阶段」位于 tests.md 之后、subagent spawn 之前：
   ```
   5.5. 输出 plan 文件 pds/batches/batch-NN-slug-plan.md，包含：
        - 后端 / 前端 task DAG（ASCII 依赖图）
        - 每个 task 的「红 / 绿 / commit message」三段
        - commit message 模板：feat(batch-N-{be|fe}): T0X — ...
          （task 编号 T0X 必须出现，使续跑 agent 可用 git log --grep 定位进度）
        - 风险点 + 预案（已知 hairy 点的 fallback，预防续跑 agent 重复摸坑）
   ```
2. 新增 §4.3 plan 模板，列出上述 4 段固定结构。
3. plan 文件不算停止点（不触发邮件），但是 spawn subagent 前的硬前置。
4. 兼容已 done 的旧批：无需 backfill plan 文件，从 Batch 6 起强制。

## 2026-06-06 契约模板缺「序列化契约」小节 → 验收期 bug 集中在前后端类型不一致

**现象**：Batch 5 验收期暴露的 3 个 bug 全在前后端类型边界：
- omitempty 让"显式 null"与"字段缺失"在前端 zod 校验里语义不同；
- 后端某字段返 `string` 但前端 type 写 `string[]`（specialties / certificates）；
- owner 详情页 editor 在 `is_owner=true` 时缺前置态约束。

三处都是契约 API 接口表只写了"字段名"，未约束"JSON 形态 + nullable + 空集合语义"。spec 没强制前后端在契约阶段对齐类型，导致 subagent 各自实现时按各端常识 drift，到验收才发现。

**风险**：
1. 跨端 batch 越多，序列化边界 bug 越多；目前靠 review 阶段 backend/frontend finder 兜底，但 Batch 5 实测 3-finder 配置仍漏掉了 string vs []string（属 test-gap 类）。
2. 契约文档作为 source of truth 的"类型权威性"不足，subagent 都会以各自语言的类型系统为准，反而把契约文档绕开。

**建议沉淀到 CLAUDE.md §4.1 契约模板**：

1. 接口表"响应字段"列拆成"字段名 / 类型 / nullable / omitempty / 空集合形态"五个子列；不能简写为单词。
2. 契约新增「序列化契约」小节，列出全部跨端共享的复杂类型（数组、object、enum）的两边权威形状（TS interface + Go struct tag）。
3. plan 模板（见上一条 FR）在前端 F01 与后端 domain types task 之间显式建一行"类型对齐 gate"——任一端实现期发现 drift 必须先回改契约，不允许各自实现。
4. review 阶段配套：跨端重构批的 finder 配置从 3（backend / frontend / 架构回归）升级为 3+1，追加 test-gap finder 专门盯前端 zod / type 与后端 JSON 的 negative case。

## 2026-06-06 契约 approve 流程缺少"问题降级"机制

**现象**：batch-03 契约第 386-394 行 "用户需确认 5 问"包含了一个跨 Batch 调度问题（"异常订单补偿何时启动（下一批次）"），用户实际 approve 时只能整体 yes/no，没法说"前 4 个我答了，第 5 个推迟到下个 Batch"。

**建议**：CLAUDE.md 步骤 3（"等待用户 approve 契约"）补一句：

> 用户答复需对每个待确认问题单独标 `confirmed / deferred / rejected`。`deferred` 的问题自动迁移到 PROGRESS.md 的 Next bucket，并在契约文件保留链接，不阻塞 approve。
