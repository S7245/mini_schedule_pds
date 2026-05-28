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
读取 PDS
-> 更新/确认 PROGRESS.md 的当前目标
-> 对照 backend/web 现状
-> 选一个小批次任务
-> 实现 backend
-> 实现 web
-> 补测试/构建验证
-> 更新 PROGRESS.md
-> 必要时写入 .learnings
-> 提交并 push
```

不要跳过 `PROGRESS.md`。它是跨会话恢复上下文的主要入口。

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

`/pds/.learnings` 已关联 `self-improving-agent`。项目级经验写入：

- `.learnings/LEARNINGS.md`
- `.learnings/ERRORS.md`
- `.learnings/FEATURE_REQUESTS.md`

跨项目通用模式再考虑更新全局 skill memory。

## 10. 当前优先级

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
