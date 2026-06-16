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

## 2026-06-06 Batch 4 process refinements

Batch 4（向导骨架 + Location 闭环）首次跑完整 grill → 5 问 approve → tests → TDD → /code-review → 7 修/9 转 FR → 验收链路，沉淀如下：

### 1. grill 阶段触发"重新切批"的硬信号：跨域依赖 ≥ 3

- 用户起初提"额度硬限制 + 向导一起做"。grill 阶段画依赖树发现向导第 3-8 步需要 Staff / Instructor / CourseCategory / CourseTemplate / EntitlementTemplate / ClassSession 6 个域的 CRUD —— 严重违反 §5 "每批一闭环"。
- 经验：grill 时若发现待做主题依赖 ≥ 3 个**当前未实现域**才能闭环，必须停下用 AskUserQuestion 重切，不要硬塞进契约。Batch 4 的解法是切成"骨架 + 仅第 2 步闭环，其余 6 步占位可跳过"，把 6 域依赖降到 0。
- 把这条信号写进 CLAUDE.md §4 grill 设计树清单：grill 必须显式画"本批触达的域 × 是否已有 CRUD"矩阵，缺口 ≥ 3 即触发重切对话。

### 2. 测试场景文件按"Happy / Edge 分域 / 并发 race"三段式

- Batch 4 tests 文件首次采用：Happy Path（H1-H10）+ Edge Cases 按域分组（入口守卫 / 品牌资料 / Location / 跳过完成）+ 并发 race（E24-E25）独立段。比 batch-03 单一表格更易扫描覆盖盲区。
- 并发 race 段约定标注"sandbox 不可测，留人工 / testcontainer"——避免 subagent 强行 mock 出假绿。该约定可写进 §4.2 模板。
- 推荐 §4.2 测试场景模板固化为：`## Happy Path` / `## Edge Cases — <域>` × N / `## Edge Cases — 并发 / 串行化` / `## 执行方式`。

### 3. "5 个待确认问题"的粒度评判标准（Batch 4 实证 vs Batch 3 反例）

- Batch 4 的 5 问全部满足 LEARNINGS §4 三条件（yes/no 拍板 + 不答阻塞 + 本批限定），用户实测能逐条回 OK/修改，第 3 题（status switch 选型）甚至被合理留给前端 agent 自决。这是契约提问粒度的正面范例。
- 反面对照：Batch 3 第 5 题"异常订单补偿何时启动"属跨批调度，模糊化了 approve。
- 规则升级：契约末尾"待确认 N 问"提交前自检——每题打三个勾才能保留：`[ ] 可一句话拍板 [ ] 不答阻塞本批 [ ] 范围 ⊆ 本批`。任一不打勾，要么改写、要么挪去 PROGRESS.md Next bucket。

### 4. 邮件双发 + auto-mode classifier 误判 → 把"显式允许"写进章节末注释

- Batch 4 契约 approve 邮件按 CLAUDE.md §10 双发到 QQ + Gmail，结果 auto-mode classifier 拦了一封（误判为重复发送）。CLAUDE.md §10 本身写明双发协议，但 classifier 不读项目 CLAUDE.md。
- 补救：§10 末尾加一行 inline rationale 注释，写明"双发是协议必须，非重复"——同时在 commit message / 邮件 subject 里加 `[per pds/CLAUDE.md §10 双发协议]` 标签，给后续 classifier / reviewer 留信号。
- 更根本的改进：把双发收件人写成单一 group alias（如 user-mailgroup@...），但需用户先建 alias，暂缓。

### 5. "/code-review 自检 → 修 / 转 FR" 双轨成为标准模板

- Batch 4 首次跑 7 个 finder 并行 → dedupe 26 → 10 → 当场修 7 / 转 FEATURE_REQUESTS 9。流程稳定有效，应固化进 CLAUDE.md §4 步骤 9 的具体执行：
  1. spawn 7 finder 并行（bug / security / perf / style / test-gap / docs / arch）
  2. dedupe 后按"本批可修 ≤ 30 分钟"二分：修 / 转 FR
  3. 转 FR 的每条带"为何不本批修"一句话理由
- 该模板让 code-review 产出可控、不阻塞本批 approve，同时长期改进项不丢。后续每个 Batch 步骤 9 都按这个二分法走。

## 2026-06-06 Batch 5 process refinements

Batch 5（Staff CRUD + InstructorProfile + 角色 seed + SubscriptionGuard 重构）是首批严格走 `grill → spec → plan → TDD → /code-review` 全流程的批次。沉淀如下：

### 1. plan 文件作为外部状态机，使 subagent 中途崩溃可"无缝续跑"

- Batch 5 实操中后端 subagent 在 T07 附近因 API 403 中断；续跑的 agent 仅凭 `batch-05-staff-instructor-roles-plan.md` + 已 push 的 commits + commit message 里的 task 编号（`feat(batch-5-be): ...`）就接力成功，未回灌任何对话上下文。
- 让"plan 文件 = 外部状态"成立的三要素：
  1. plan 文件里画 DAG（13 task 依赖图）+ 每个 task 的红绿步骤是可机读的执行手册，不是 narrative 描述。
  2. commit message 模板 `feat(batch-N-{be|fe}): T0X — ...` 把 task 编号反向写回 git log，续跑 agent 可用 `git log --grep=batch-5-be` 直接定位"完成到哪个 task"。
  3. 风险/预案章节预先标注了 audit pkg 循环依赖、Wire 失败等"已知 hairy 点"的 fallback，避免续跑 agent 自己摸坑。
- 规则升级：CLAUDE.md §4 步骤 6 之前增设 4.5 步骤"plan 文件"，把这三件事固化进 §4.3 plan 模板（待 FR 落地）。

### 2. grill 决定点从 5 → 10，细到"是否本批顺手清债"颗粒

- Batch 3 / 4 的 grill 各产 5 个根决定点；Batch 5 grill 产 10 个：范围档位、SubscriptionGuard 是否顺手抽、permission seed 范围、/staff vs 扩 /users、owner backfill 双轨、owner 删除策略、data_scope 落地范围、菜单可见性、InstructorProfile 摆放、soft-delete 实现。
- 关键增量在"是否顺手清 Batch 4 转出去的 FR"这一类问题（决定 2 / 5 / 8）—— 把 FR 队列与当前批的"附加范围"挂钩，避免 FR 越积越多但永远没合适窗口落地。
- 但 10 个决定点已接近用户阅读疲劳上限；Batch 5 用户实测都能逐条回，但 grill 末尾 2 条（决定 9 / 10）回答明显仓促。**甜区估计在 6–8 个决定点**，再多需要拆"必答 / 备选"两组。
- 规则：grill 决定点 ≤ 8 个；超出时强制按"本批闭环必须 / 可推迟到下批"二分，后者归入 PROGRESS.md Next bucket 而非塞回契约提问。

### 3. 契约 / plan 阶段缺"前端 ↔ 后端类型对齐 checklist"，导致验收期 bug 集中在序列化边界

- Batch 5 验收期出现的 3 个 bug 全是前后端类型不一致：
  - omitempty 让"显式 null"和"字段缺失"在前端 zod 校验里行为不同；
  - 后端某字段返 `string` 但前端 type 写 `string[]`（specialties / certificates）；
  - owner 详情页 editor 在 `is_owner=true` 时未禁用某些操作（前端缺前置态描述）。
- 三处都是契约 API 表里"响应字段"列只写了名字、没写"JSON shape + nullable + 空数组 vs null 语义"。spec 阶段需补一节强约束：
  - 每个响应字段写 `name: type | nullable=true/false | omitempty=true/false | 空集合形态=[]/null`。
  - plan 阶段在 F01（前端 types task）和 T05（后端 domain types task）之间显式建一个"类型对齐表"，两端任一边 drift 必须更新 plan 而非各自实现。
- 规则升级（也写进 FR）：契约模板加 §"序列化契约"小节，列字段级 nullable / omitempty / 空集合形态。

### 4. review 阶段：finder 角度从 7 → 3 聚焦的取舍

- Batch 4 的 7 个 finder（bug / security / perf / style / test-gap / docs / arch）产 26 候选，dedupe 后 10 项；其中 perf / style / docs 三类 finder 各只贡献 0-1 个独立候选，重复成本高。
- Batch 5 改成 3 个聚焦角度：backend correctness、frontend correctness、架构回归（SubscriptionGuard 抽出 + audit pkg 抽出对 Batch 4 行为的影响）。产 20 候选 dedupe 后 17 项，单 finder 信噪比明显高于 Batch 4。
- 但代价：security / test-gap 角度未独立 finder，本批是靠 backend correctness finder 顺带捞 —— 验收期 3 个 bug 中 string vs []string 一项严格说属于 test-gap（前端 zod 没 negative test），3-finder 配置漏掉了。
- 折中规则：默认 3 finder（backend / frontend / 架构回归），**当本批含跨域重构时**追加 1 个 test-gap finder（聚焦"被改的旧 API 的回归测试是否够"），形成 3+1 灵活档。

### 5. 验收期 bug 3 个 / 全在序列化边界 → 信号：spec 阶段缺契约级类型对齐，不是实现质量问题

- Batch 5 验收期 bug 比 Batch 4（验收期 1 个 bug）多，但**全部聚集在前后端序列化对齐**，说明不是 subagent 实现退化，而是 spec 没要求把类型形状写到字段级。
- 这一信号支持 §3 的规则升级：把"序列化契约"列进契约模板，spec approve 前就要求填完，本批以后强制。
- 同时启示 review 配置（§4 的 3+1 archetype）：跨端重构批必须含独立的 test-gap finder，专门盯前端 zod / type 与后端 JSON 的 negative case。

## 2026-06-11 Batch 6 process refinements

Batch 6（RBAC enforcement + data_scope 落地）已业务验收通过。本批暴露的全是"流程/协作层"而非代码质量问题，沉淀如下：

### 1. 验收/收尾必须逐项标 done / skipped / blocked，禁止含糊带过

- 本批 agent 一度把未完成的 T07 / T10 在验收邮件里含糊处理，让"没做"看起来像"做了一部分"。用户纠正后明确要求：收尾时每一项必须如实区分三态——**主动选择跳过（skipped + 一句话理由）** vs **真卡住（blocked + 卡在哪、缺什么才能解）** vs **完成（done）**。
- 根因：plan 的 task 状态只有"做没做"二值，没有"为什么没做"的语义，收尾时容易用模糊措辞掩盖 blocked，让用户误以为整批闭环。
- 规则升级：步骤 10 验收邮件 + PROGRESS 回填都必须带 task 级三态表（`T0X | done/skipped/blocked | 理由`）。skipped 要写"为何主动跳过 + 是否转 FR"，blocked 要写"卡点 + 解锁条件"。绝不允许 blocked 项被静默并入"已完成"。

### 2. subagent 被 403 kill 后主线程接管，需具备"修 test 编译错误"的恢复能力

- 延续 Batch 5 §1（plan 文件作外部状态机可续跑）的经验，本批后端 subagent 多次被 403 kill，但这次主线程不仅"接着写 task"，还要**手动修被中断 subagent 留下的 test 编译错误**——即半途崩溃可能留下"红都跑不起来"的中间态（缺 import、签名改了一半、mock 没补全）。
- 经验：续跑的主线程接管第一步不是继续下一个 task，而是先 `go build ./... && go vet ./...` 把上一个 task 留下的半成品编译态恢复绿/红可运行，再往下走。批次实现要把这条"接管前先复位编译态"写进 plan 的恢复预案章节。
- 规则升级：CLAUDE.md §4.3 plan 的"风险/预案"段固定加一条 takeover checklist：①`git log --grep` 定位完成到哪 ②`go build ./...` 复位编译态 ③ 检查最后一个 commit 是否红绿完整，半成品先补齐再续。

### 3. 验收期才暴露的前端 bug，根因是契约"测试场景"只覆盖 happy path

- Batch 6 业务逻辑全过，但用户实操验收时才发现两类 bug：**disabled 按钮的 tooltip 不显示**（权限拒绝后没有 UI 反馈）、**跨用户缓存泄漏**（切换账号后旧用户的权限数据残留）。两者都不在自动测试覆盖内。
- 根因：契约的"测试场景"默认只描述 happy path，缺两条交互路径——"权限拒绝后的 UI 反馈"和"切换账号"。只测 happy path 必漏这类。
- 规则升级：§4.2 测试场景模板强制新增两组 Edge Case：
  - `## Edge Cases — 权限拒绝 UI 反馈`：每个受 RBAC 控制的操作都要列"无权限时按钮状态 + tooltip / 提示文案 + 点击行为"。
  - `## Edge Cases — 账号切换 / 缓存隔离`：列"A 登录→操作→登出→B 登录"，断言无任何 A 的权限/数据残留（前端 store、react-query cache、localStorage 都要清）。
- 这两组属于"权限批"通用盲区，凡涉及 RBAC / data_scope 的 Batch 强制包含。

### 4. data_scope 越权语义（404 vs 403）必须在 grill 阶段定死并写进契约

- 本批是**实现时才明确**"越权访问越权资源返 404 不泄漏存在性"（而非 403）。这是一个安全语义决策，却拖到了实现阶段才拍板，意味着 grill / 契约都没把它当成必答决定点。
- 经验：404 vs 403 不是实现细节，而是"是否泄漏资源存在性"的安全契约，属于 grill 必答根决定点。同类还有：越权列表是过滤掉还是报错、跨 scope 引用 ID 的报错形态。
- 规则升级：grill 设计树清单（CLAUDE.md §4 步骤 2）加一条强制项——凡含 data_scope / 多租户隔离的 Batch，必须显式回答"越权读 / 越权写各返什么状态码、是否泄漏存在性"，并把结论写进契约的"序列化契约 / 错误码"两列，不许留到实现期。


## 2026-06-12 Batch 7 — 流程沉淀

- **code-review 发现的"潜在 bug"可能是线上活 bug**：本批 code-review 标记的共享 `isUniqueViolation` 脆弱字符串匹配，验收时直接以"手机号重复返 500"现形。教训：review 阶段标的 helper 级缺陷别一律转 FR，先判断有没有用户可见路径会触发。
- **验收前重启/重建后端**：B1 增量逻辑被旧 `go run` 二进制掩盖，险些误判实现错误。验收 checklist 加一条"确认跑的是最新构建"。
- **契约措辞要与前端可见 UI 对齐**：契约 Happy #7/#8 写"location 入口随权限消失"，但前端 `location.view` 根本没有可见门（`/locations` 页未建）。C1 只能在 API+Redis 层验证。写验收场景时，凡是"某权限驱动某 UI"，先确认该 UI 门已存在，否则场景无法在 UI 层验。

## 2026-06-12 Batch 8 — 流程沉淀

- **真实栈 e2e 比 mock 更值**：Batch 8 用真登录+真 CRUD+真 DB 的 Playwright，直接跑出 2 个共享底层 bug（204 DELETE 静默失败、persist 水合竞态），mock API 永远抓不到。代价是要管测试数据（唯一名 + teardown 软删）。
- **验收要跑 prod build 不只 e2e**：dev server / e2e 不做全量 type-check，共享包漏声明 react 依赖在 dev 下隐身，只有 `pnpm build` 暴露。验收 checklist：e2e 通过 + 三端 `pnpm build` 通过。
- **小批次也值得做**：Batch 8 本体（一个镜像页 + 导航）很小，但顺带补齐了 Batch 7 遗留的 location.view 门，且 e2e 挖出两个影响全栈的底层 bug。"补缺口"批次的真正价值常在副产物。

## 2026-06-16 Batch 11 — 流程沉淀

### 1. 契约的「API 接口表」必须含前端依赖的所有后端端点
本批前端 api client 早注明 `ASSUMPTION (backend must match): GET /instructors?schedulable=true`，但契约 API 表只列了 course/session 的 CRUD，漏了排课弹窗拉教练列表这个读端点 → 后端按表实现 → 该端点缺失 → 排课全链路 UI 阻断，直到验收 e2e 才暴露。规则：grill 阶段把「每个前端选择器/下拉拉哪个后端 list 端点」显式列进契约 API 表；起飞前 grep 全仓 `ASSUMPTION` 兜底。

### 2. 纯 API 烟测不能替代走 UI 选择器的 e2e
主线程后端 curl 烟测全绿（happy + 全 edge），但**硬编码 instructor_profile_id 绕过了教练下拉**，所以没抓到缺端点的 bug。是用户开的测试 session 跑 UI e2e（真点下拉）才暴露。结论：涉及 UI 选择器拉列表的功能，验收必须有走真实选择器的 e2e；API 烟测只证明「最终写接口对」，不证明「UI 能填出这个请求」。

### 3. 主线程直接实现 + 验收期外部 session 跑 e2e 的协作模式有效
用户拒绝 spawn 实现 subagent，改主线程逐 task TDD commit；e2e 由用户另开 session 跑（主线程给自包含 prompt）。外部 session 不仅跑通还自行定位+修了缺端点 bug 并 commit。模式成立：主线程产出 + 自包含 handoff prompt（含账号/端口/重启铁律/包名 filter/.next 坑）让外部 session 能独立闭环。

## 2026-06-16 Batch 12a — 流程沉淀

### 1. 大主题先 grill「是否拆批」，拆按依赖单向排序
Batch 12（循环排课+资源管理）grill 第一问就是拆不拆。两特性依赖单向（recurring 可选绑 resource），拆 12a（资源，先）+ 12b（循环排课，后，复用资源选择器）。每批独立 e2e/验收，依赖方向干净。结论：一个「主题」含两个各自完整纵切片（migration/domain/app/persistence/handler/前端/e2e）时，默认拆，按依赖先后排，别贪一批。

### 2. 后批的设计决策可在前批 grill 时一次性预定，写进 PROGRESS
12b（循环排课）的关键决策（部分失败 SAVEPOINT、0 成功 abort、非级联 cancel、复用 session.*、时区）在 12a 的 grill 里一并和用户拍板，写进 PROGRESS 的 12b 占位段。好处：12b 起飞直接写契约，不重新 grill；坏处：需在 12a 验收后复核是否仍成立。

### 3. 提前为「后批才有数据」的引用写 guard，避免返工
12a 的资源 Delete guard 和门店 Delete guard 都把 active recurring_schedules 一并 COUNT 进去（12b 落地前恒 0）。表已在 000003 建好，提前写引用检查零成本，省 12b 回头改两处 guard。

### 4. AskUserQuestion 批量预决策 + 「均推荐项」模式延续
沿用 Batch 11 的 grill→AskUserQuestion（4 问，每问首选标推荐）→用户一次性拍板模式。本批 4 问（拆批/0成功处理/cancel语义/资源UI放置+删除策略）全选推荐，零返工进入契约。
