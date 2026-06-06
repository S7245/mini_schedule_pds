## 2026-05-27 self-improving-agent association

- Associated skill: `/Users/liushan/.agents/skills/self-improving-agent/SKILL.md`
- Project: `/Users/liushan/Documents/zkw/mini_schedule/pds`
- Project-local memory: `/Users/liushan/Documents/zkw/mini_schedule/pds/.learnings`
- Rule: run skill-related work from `/pds` or its child paths so the hook helper can discover this `.learnings` directory by walking parent directories.

## 2026-05-28 PDS implementation workflow

- Added `CLAUDE.md` as the persistent agent instruction file for `/pds`.
- Added `PROGRESS.md` as the implementation ledger for Now / Next / Later delivery.
- Pattern: use `/pds` as source of truth, then execute one vertical batch at a time across `/backend` and `/web`.

## 2026-05-28 Batch 1 public signup order

- Public signup order endpoint is `POST /api/v1/public/signup/orders`.
- Batch 1 creates a `pending` Brand, an active BrandUser owner placeholder, and a `pending_payment` SaaSPlanOrder.
- The SaaS order amount must be derived inside the backend transaction from the active SaaSPlan monthly/yearly price; frontend amount is never accepted.
- SMS remains mock-first in development. Production without a real provider must not silently allow public signup.

## 2026-06-06 Batch contract authoring patterns

回顾 batch-01 / batch-02 / batch-03 三份契约 + 两份测试场景文件，归纳契约文档书写层面的可复用经验：

### 1. API 接口表必须带"类型 + 错误响应"两列

- 现状：batch-01 写 `phone, sms_code, password`，batch-02 写 `order_id: string`，batch-03 用嵌套 JSON 块代替表格。粒度不统一导致实现层对字段类型 / nullable / 单位（元 vs 分）会再次返工。
- 建议固定列：`方法 | 路径 | 请求字段(name: type, 必填?) | 成功响应字段 | 错误码 + 触发条件`。错误码列必须列出至少：参数错（400）、鉴权/签名错（401）、幂等命中（200）、业务冲突（409/422）、服务端错（500）。
- 反例：batch-03 把"验签失败返回 401 / 订单不存在返回 200"放在验收清单和测试文件，没回填到接口表，等于接口表本身契约不完整。

### 2. Edge cases 表按"域"分组比按"入口"更不易漏

- batch-01 按入口分（注册页 / 套餐页 / 支付页），结果完全没有"事务回滚 / 并发 / 幂等"组，这些恰好是后端 bug 高发区。
- batch-03 改成按域分组（幂等 / 金额 / 验签 & 防重放 / 订单状态 / trade_state / 事务回滚），覆盖明显更完整。
- 规则：新 Batch 强制至少包含以下分组当中"业务相关的"：输入校验、鉴权/签名、幂等/重放、状态机非法跃迁、金额/单位、并发、事务回滚、外部依赖失败。无关项写"N/A"也要列出，避免遗漏。

### 3. 数据库变更段落必须"已存在 / 新增 / 待确认"三分

- batch-03 第 3 节写 SaaSPlanOrder（已存在，需确认 third_party_trade_no 字段）—— 这正是"已存在"和"待确认"混在一起的坏味道。结果实现阶段才发现该列实际可能未建。
- 推荐结构：
  - `### 已存在（无需迁移）`：直接列字段，注明"本 Batch 仅消费"。
  - `### 本 Batch 新增`：完整 DDL + 索引 + check constraint + down 迁移。
  - `### 待确认（migration 前必须 grep schema）`：列出怀疑存在但未验证的列，要求实现 agent 先跑 `\d table_name` 或 `grep -r` 后回填到上面两段之一。

### 4. "待确认 N 问"必须粒度到可 yes/no，且不混入跨 Batch 调度

- batch-03 5 个问题里："API 是否满足微信 v3 格式""数据库是否完整""事务步骤是否合理"这种问法用户无法直接回答 yes/no，只能"嗯都看了看挺好"，等于走过场。
- 第 5 个问题"异常订单补偿何时启动"属于下个 Batch 调度问题，不应该阻塞当前契约 approve。
- 规则：每个问题必须满足：(a) 用户能用一句话拍板；(b) 不答会直接卡住实现某一步；(c) 范围严格限定本 Batch。跨 Batch 决策放进 PROGRESS.md 的 Next / Later，不放契约。

### 5. 契约文件头部需要统一的 lifecycle 标签

- batch-01 写"实现中"、batch-02 写"已完成 ✅"、batch-03 写"契约草稿 ⏳"。三种状态机不一致，跨 Batch 检索时无法用相同关键字过滤。
- 推荐固定枚举：`draft` → `approved` → `implementing` → `static-verified` → `scenario-verified` → `done` / `parked`。每次状态变更补一行 changelog（时间 + 状态 + 触发人）。

### 6. 测试场景文件必须区分"已执行 / 未执行"，不应只列设计

- batch-03 tests 文件 19 个场景全是设计描述，没有"实际执行结果 / 通过率 / 截图或日志位置"的列。PROGRESS 却把 Batch 3 标 ✅，等于"写了测试用例"被等同于"测试通过"。
- 推荐每行加最后一列"执行状态"（pending / passed / failed / skipped + 一句话理由），并在文件顶部加汇总（"H1-H8 通过 / E1-E19 待集成测试"），让 PROGRESS 引用这个汇总而非整个文件。

