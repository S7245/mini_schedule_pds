# pds 工作指令

本文件是 `/pds` 需求仓库的常驻工作上下文，供 Claude/Codex/其他 Agent 在逐步实现课程预约 SaaS 时读取。

## 1. 项目定位

Mini Schedule 当前按 **通用课程预约 SaaS** 推进，不再限定为健身行业。底层术语使用 Brand、Location、Staff、Instructor、Learner、CourseTemplate、ClassSession、Booking、Learner Entitlement。

行业展示层可以映射为：

- 健身：门店、教练、会员、课包/会员卡。
- 语言培训：校区、老师、学员、课包/会员卡。

## 2. Source Of Truth

需求和实现冲突时，优先级如下：

1. `COURSE_BOOKING_BUSINESS_BLUEPRINT.md`
2. `GO_BACKEND_LANGUAGE_DESIGN.md`
3. `BACKGROUND.md`
4. `PROGRESS.md`
5. 代码现状：`/backend`、`/web`

如果代码与 PDS 冲突，默认以 PDS 为准；但实现前必须记录差异和迁移风险。

## 3. 仓库边界

工作区包含三个主要 git 仓库：

- `/pds`：产品背景、需求蓝图、实施计划、进度账本。
- `/backend`：Golang 后端，Gin/GORM/PostgreSQL/Redis/Wire。
- `/web`：Next.js monorepo，包含 admin、brand、app 三端前端。

根目录不是当前主要提交单位。提交时按仓库分别 commit/push。

## 4. 每轮实现流程

每次实现都按这个顺序执行：

```text
1. 读取 PROGRESS.md + PDS，确认当前 Batch 目标
2. **grill 设计树**：在写契约前，主线程内自我 grill，覆盖
   - 数据模型边界（已有表 vs 新增、字段缺口、状态机）
   - 跨域依赖（这批要做的事是否要求其他域有 API 才能闭环）
   - 失败模式（事务回滚、幂等、并发、外部依赖错误）
   - 验收闭环（最终怎么手动跑通业务流）
   如果发现合并方案不可行（如 Batch 4 起初提出的合并），必须在写契约前停下问用户。
3. 输出 Batch 契约（API 接口表 + 前端页面模块清单 + Wireframe）
   -> 写入 pds/batches/batch-NN-slug.md
4. 等待用户 approve 契约                          ← 必须停下等待（发邮件）
5. 输出 Batch 测试场景，写入 pds/batches/batch-NN-slug-tests.md
   -> 格式见 4.2 节
6. 并行 spawn 两个 subagent：
     - 后端 agent：按契约实现接口（遵守 GO_BACKEND_LANGUAGE_DESIGN.md）
     - 前端 agent：按契约 + 模块清单 + Wireframe 实现 UI
   subagent 应在仓库内**逐 task TDD commit**，不要把整批堆成一个大 commit：
   先红 → 实现 → 绿 → 重构 → 单 task commit，按文件 / 域为粒度划分。
7. 自动跑静态验证：go build ./... + pnpm lint + pnpm build
8. 用 browser-automation 或 senior-qa skill 执行测试场景文件
9. **stage code-review**：主线程调 `/code-review` 自检本批 diff，把发现的问题
   要么修掉、要么明确转 FEATURE_REQUESTS.md 里。
10. 等待用户人工验收业务逻辑                       ← 必须停下等待（发邮件）
11. 用户 approve → 更新 PROGRESS.md → 进入下一 Batch
    -> 写入 .learnings（记录本轮踩坑）
    -> 提交并 push
```

**两个强制停止点：** 步骤 4（契约 approve）和步骤 10（业务验收），缺一不可。两点都必须通过 `/resend` 发邮件给用户（870941563@qq.com），sender 用 `mini-schedule@zkwcloud.com`。邮件内容要包含：当前 Batch 编号 + 待确认事项 / 验收指令 + 关键文件路径。

不要跳过 `PROGRESS.md`，它是跨会话恢复上下文的主要入口。

## 4.1 Batch 契约格式

每个 Batch 开始时，Claude 创建独立契约文件 `pds/batches/batch-NN-slug.md`，并在 `PROGRESS.md` 对应 Batch 区块中添加链接引用。契约文件包含：

```markdown
#### 契约

**API 接口**

| 方法 | 路径 | 请求字段 | 响应字段 |
|---|---|---|---|
| POST | /api/v1/... | field_a, field_b | id, status |

**前端页面模块**

| 页面/模块 | 类型 | 关键字段/操作 |
|---|---|---|
| 支付页 | 页面 | 二维码、倒计时、轮询状态 |
| 取消弹窗 | 弹窗 | 确认取消按钮 |

**前端实现约束**

- 复用 /web 现有组件和设计风格，不引入新 UI 库。
- 按当前 app（admin/brand/app）的布局惯例实现，不跨端改动。
```

## 4.2 Batch 测试场景格式

每个 Batch 契约 approve 后，创建测试文件 `pds/batches/batch-NN-slug-tests.md`，格式：

```markdown
# Batch NN 测试场景

## Happy Path

| # | 步骤 | 预期结果 |
|---|---|---|
| 1 | 访问 /signup，填写合法手机号 + 密码 + 品牌信息，点"下一步" | 跳转 /signup/plan |
| 2 | 选择套餐，点"立即支付" | 跳转 /signup/payment/[order_id] |

## Edge Cases

| # | 场景 | 预期结果 |
|---|---|---|
| E1 | 密码少于 8 位 | 前端显示"密码至少 8 位" |
| E2 | 手机号已注册 | pre-validate 返回"该手机号已注册" |

## 执行方式

使用 browser-automation skill 或 senior-qa skill 根据上表生成 Playwright 脚本并执行。
```

**执行时机：** 步骤 6 静态验证通过后，步骤 7 调用 skill 执行测试场景，再进入步骤 8 人工验收。

## 5. 批次大小

每批只做一个可闭环主题，优先选择能端到端验证的纵切片。

推荐批次：

- 平台套餐管理。
- 公开注册和首购订单。
- 微信支付 Native 下单和回调。
- 品牌初始化向导。
- Location/Staff/权限。
- 课程模板和场次。
- 学员权益和预约。
- 签到和权益消耗。

不推荐：

- 一次性横向铺满所有表的 CRUD。
- 同时改 admin、brand、app 三端且没有共同验收场景。
- 先做复杂抽象再找业务落点。

## 6. 实现约束

### 后端

- 遵守 `GO_BACKEND_LANGUAGE_DESIGN.md`。
- Handler 不写业务状态机。
- Application 负责业务校验和流程编排。
- Repository 负责 GORM 和事务。
- 商业关键操作必须写 OperationLog。
- 金额不用 float。
- 所有 I/O 函数传 `context.Context`。

### 前端

- 优先复用 `/web/apps/admin`、`/web/apps/brand`、`/web/apps/app` 的既有组件和布局。
- 平台 admin 是运营工具，不做营销页。
- 复杂流程优先做清晰表格、筛选、状态、操作弹窗。
- 不把分页数据伪装为全局统计。

### 数据库

- 迁移必须支持从空库顺序 up。
- 可回滚范围内要补 down。
- 重要状态字段应有 check constraint。
- 外键列要有索引。
- 支付、订阅、预约、权益流水要保证幂等和审计。

## 7. 验证命令

后端常用：

```bash
cd /Users/liushan/Documents/zkw/mini_schedule/backend
go test ./...
```

前端 admin 常用：

```bash
cd /Users/liushan/Documents/zkw/mini_schedule/web
pnpm --filter @mini-schedule/admin lint
pnpm --filter @mini-schedule/admin build
```

注意：本地 sandbox 里 `go test` 可能需要访问 Go build cache；Next dev server 监听端口也可能需要提升权限。相关经验记录在各仓库 `.learnings`。

## 8. 进度更新格式

每轮结束必须更新 `PROGRESS.md`：

- 更新时间。
- 本轮完成了什么。
- 验证命令和结果。
- 剩余风险。
- 下一步建议。
- 是否已提交/push。

## 9. 自我学习

`/pds/.learnings`、`/backend/.learnings`、`/web/.learnings` 三个目录均关联 `self-improving-agent`。每个 Batch 收尾必须分别触发三个仓库的总结 agent，把本批次经验写入对应文件：

- `.learnings/LEARNINGS.md`：可复用经验、设计模式、约定
- `.learnings/ERRORS.md`：本批踩坑 + 修复方式 + 可能波及的其他文件（Pending exposure）
- `.learnings/FEATURE_REQUESTS.md`：本批暂未做但未来该做的事

跨项目通用模式再考虑更新全局 skill memory。

## 10. 邮件协议（/resend）

每个强制停止点必须通过 Resend 发邮件给用户：

- **收件人**：samlau7245@gmail.com（QQ 邮箱 870941563@qq.com 实测会被拦截/丢弃，不用）
- **发件人**：mini-schedule@zkwcloud.com（zkwcloud.com 域已在 Resend 验证）
- **触发节点**：
  1. 步骤 4：契约 approve 等待 — Subject 用 `[Batch NN] 契约待 approve — <slug>`，正文列待确认问题 + 契约文件路径 + GitHub link
  2. 步骤 10：业务验收等待 — Subject 用 `[Batch NN] 待业务验收 — <slug>`，正文列验收指令（curl / playwright / 手动步骤）+ 期望结果 + 关键 commit SHA

不发邮件的节点：契约草稿写作中、静态验证、测试场景生成、subagent 实现、code-review 自检等"无需用户介入"的内部步骤。

如果 Batch 中途发现合并不可行 / 范围决策（如 Batch 4 起步时发现需重新切分），也算停止点，发邮件并附决策选项。

## 11. 当前优先级

第一阶段优先完成平台商业化闭环：

```text
SaaS Plan
-> 公开注册
-> Pending Brand / Pending Staff
-> 微信支付 Native
-> 支付回调
-> BrandSubscription 快照
-> 额度硬限制
-> 平台补偿运营
```

品牌端、员工端、学员端在平台商业化闭环稳定后进入下一批。
