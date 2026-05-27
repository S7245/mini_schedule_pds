# Go 后端语言设计

整理日期：2026-05-27

状态：v0.1，作为课程预约 SaaS 后端实现约束

本文补充课程预约项目的 Golang 语言层设计，服务于 `/backend` 后续实现、代码评审和测试补齐。业务边界以 `COURSE_BOOKING_BUSINESS_BLUEPRINT.md` 为准；本文只约束 Go 代码如何表达这些业务。

参考本地 Skills：

- `go-style-core`
- `go-packages`
- `go-error-handling`
- `go-context`
- `go-testing`

## 1. 设计目标

后端第一阶段不是追求复杂框架，而是把平台商业化、品牌业务和学员预约的核心规则写得清楚、可测试、可演进。

优先级：

1. 业务语义清晰。
2. 租户隔离和权限边界明确。
3. 支付、订阅、权益、预约等关键状态变更可审计、可回滚。
4. 接口和数据库契约稳定。
5. 测试覆盖高风险业务规则，而不是追求形式化覆盖率。

## 2. 包结构约束

当前 `/backend` 采用分层结构，后续继续沿用：

```text
cmd/
  api-admin/
  api-brand/
  api-app/
internal/
  domain/
  application/
  infrastructure/
  interfaces/
pkg/
  errors/
  response/
migrations/
```

### 2.1 cmd

`cmd/api-*` 只负责：

- 配置加载。
- 依赖注入。
- Gin engine 构建。
- 服务启动和退出。

不要在 `cmd` 中写业务规则、SQL、支付逻辑或权限判断。

`os.Exit`、`log.Fatal` 只允许出现在 `main` 顶层或等价启动入口。可测试逻辑应下沉到返回 `error` 的函数。

### 2.2 domain

`internal/domain/*` 表达业务对象、状态枚举、仓储接口和输入结构。

要求：

- 不依赖 Gin、GORM、Redis、HTTP SDK。
- 不读取环境变量。
- 不做日志输出。
- 枚举值必须与迁移中的 check constraint 保持一致。
- 金额在第一阶段使用字符串承接 `numeric(12,2)`，避免 float 表达钱。

适合放在 domain 的内容：

- `SaaSPlanStatus`
- `BrandSubscriptionStatus`
- `BookingStatus`
- `Repository` interface
- 业务输入结构，例如 `ManualRenewBrandSubscriptionInput`

不适合放在 domain 的内容：

- `gorm.Model`
- `gin.Context`
- 第三方支付 SDK request/response
- 数据库事务实现

### 2.3 application

`internal/application/*` 是用例层，负责组织业务流程。

职责：

- 参数级业务校验。
- 状态流转规则。
- 调用仓储接口。
- 调用外部端口接口，例如支付、短信、通知。
- 将多步业务收敛成一个用例方法。

示例：

- 创建 SaaS Plan。
- 手动续期 BrandSubscription。
- 支付回调后开通订阅。
- 学员预约课程。
- 员工签到并消耗权益。

不应在 application 中直接写 GORM 查询。需要持久化时通过 domain repository interface 调用。

### 2.4 infrastructure

`internal/infrastructure/*` 负责技术实现。

包括：

- GORM model。
- Repository 实现。
- Redis cache。
- 微信支付、支付宝、短信供应商、微信订阅消息等 adapter。
- 配置结构。

Repository 实现应把 GORM model 转成 domain 对象，不把 GORM struct 泄露给 application 或 interfaces。

### 2.5 interfaces

`internal/interfaces/*` 是入站接口层。

包括：

- Admin API Handler。
- Brand API Handler。
- App API Handler。
- Middleware。

Handler 只负责：

- 解析 request。
- validator 基础校验。
- 从 JWT context 取 actor/brand/user。
- 调 application service。
- 用统一 response 输出。

Handler 不写跨表事务、不拼业务状态机、不直接调用 GORM。

## 3. 命名和代码风格

遵守 Go 语言习惯和当前项目局部风格。

基本规则：

- 必须 `gofmt`。
- 不创建 `util`、`helper`、`common` 这类泛包。
- 包名描述能力，不描述层级废话。
- 错误和特殊情况先处理，主路径保持低缩进。
- 中等以上函数显式 return，不使用 naked return。
- 不用 `init()` 做配置、I/O、外部注册或环境读取。

推荐函数形态：

```go
func (s *Service) UpdateBrandSubscriptionStatus(ctx context.Context, id int64, input commercial.UpdateBrandSubscriptionStatusInput) (*commercial.BrandSubscription, error) {
    if input.Reason == "" {
        return nil, apperr.ErrBadRequest("请填写操作原因")
    }

    return s.repo.UpdateBrandSubscriptionStatus(ctx, id, input)
}
```

不推荐：

```go
func (s *Service) Update(...) (*Thing, error) {
    if ok {
        if ready {
            // business path hidden inside nesting
        }
    }
    return nil, nil
}
```

## 4. 错误处理设计

项目已有 `pkg/errors.AppError` 和 `pkg/response`，第一阶段继续使用。

### 4.1 层间错误规则

Repository：

- 找不到数据转换为 `ErrNotFoundF`。
- 数据库错误转换为 `ErrInternalF`。
- 错误消息面向前端可读。
- 内部原始错误放入 `Err` 字段，不直接暴露。

Application：

- 业务校验失败返回 `ErrBadRequest`。
- 权限失败返回 `ErrForbiddenF`。
- 不重复包装已是 `AppError` 的错误。

Interfaces：

- 不拼业务错误。
- 统一 `response.Error(c, err)`。

### 4.2 log 与 return

同一个错误只处理一次：

- 底层函数返回错误。
- 顶层 middleware 或启动入口记录日志。
- 不在 repository/service 中边 log 边 return，除非错误被吞掉并继续执行。

### 4.3 可匹配错误

当前项目主要用业务错误码给前端消费。后续如需要内部逻辑匹配，应优先用 `errors.Is`/`errors.As` 兼容的 sentinel 或 typed error，不比较 `err.Error()` 字符串。

## 5. Context 传递

所有会访问数据库、Redis、HTTP、支付、短信、通知的函数都必须接收 `context.Context`。

规则：

- `ctx` 作为第一个参数。
- Handler 传 `c.Request.Context()`。
- Repository 使用 `db.WithContext(ctx)`。
- 外部 HTTP/支付 SDK 调用要绑定 ctx 或超时。
- 不把 `context.Context` 存到 struct 字段中。
- 不定义自定义 Context 类型。
- 业务参数优先显式传参，不把可显式传递的业务数据塞进 context value。
- 只有 request id、trace id、认证身份这类请求级横切数据适合放入 context value。
- 使用 `context.WithTimeout`、`context.WithCancel` 后必须立即 `defer cancel()`。

示例：

```go
func (r *commercialRepository) ListSaaSPlans(ctx context.Context, offset, limit int, includeInactive bool) ([]*commercial.SaaSPlan, int64, error) {
    query := r.db.WithContext(ctx).Model(&SaaSPlanModel{})
    // ...
}
```

## 6. 事务设计

以下场景必须使用数据库事务：

- 支付回调：订单状态、支付流水、订阅快照、品牌状态、回调日志。
- 订阅手工续期、额度调整、冻结/解冻和 OperationLog。
- 预约创建：名额占用、权益锁定、预约记录。
- 取消预约：释放名额、释放权益锁定、候补处理。
- 签到：Attendance、SessionRecord、权益消耗流水。

事务规则：

- 事务边界放在 repository 或专门的 transaction adapter 中，不放在 Gin Handler。
- 事务内先锁定需要变更的核心行。
- 所有审计日志与业务变更同事务提交。
- 失败时整体回滚，避免业务事实和审计事实不一致。

第一阶段可以先在 repository 中直接 `db.Transaction`，后续如果跨 repository 用例增多，再抽象 `UnitOfWork`。

## 7. Repository 设计

Repository interface 放在 domain 包，GORM 实现放在 infrastructure。

约束：

- application 依赖 interface，不依赖 GORM。
- repository 方法名使用业务语言。
- 查询列表返回 `items, total, error`。
- 分页的 page/page_size 在 service 层归一成 offset/limit。
- GORM model 只在 infrastructure 内部使用。

示例：

```go
type Repository interface {
    CreateSaaSPlan(ctx context.Context, input CreateSaaSPlanInput) (*SaaSPlan, error)
    ListSaaSPlans(ctx context.Context, offset, limit int, includeInactive bool) ([]*SaaSPlan, int64, error)
}
```

## 8. API 设计

三端 API 边界：

- Platform Admin：平台套餐、品牌、订阅、支付、补偿、平台管理员。
- Brand Backoffice：Location、员工、权限、课程、场次、学员、权益、签到。
- Learner App：微信小程序登录、品牌空间、可约课程、预约、取消、我的权益。

Handler DTO 可以与 domain input 分开，避免前端字段直接污染业务用例。

响应统一使用：

```json
{
  "code": "OK",
  "message": "success",
  "data": {}
}
```

错误响应统一通过 `pkg/response`。

## 9. 支付与外部接口设计

微信支付第一阶段作为 infrastructure adapter 接入，不让 SDK 类型进入 domain。

建议端口：

```go
type PaymentProvider interface {
    CreateNativeOrder(ctx context.Context, input CreateNativeOrderInput) (*CreateNativeOrderResult, error)
    VerifyCallback(ctx context.Context, headers map[string]string, body []byte) (*PaymentNotification, error)
    QueryOrder(ctx context.Context, outTradeNo string) (*PaymentQueryResult, error)
}
```

支付回调必须保证：

- 验签失败不修改订单。
- 金额不一致进入异常状态。
- 重复回调幂等成功返回。
- 订单已 paid 不重复创建订阅。
- 订单、流水、订阅、回调日志同事务处理。

## 10. 测试设计

测试优先覆盖高风险规则。

第一阶段必须补的测试：

- SaaS Plan 创建默认币种和状态。
- SaaS Plan 状态切换。
- BrandSubscription 手动续期：
  - 未过期从原 `expires_at` 顺延。
  - 已过期从当前时间重新计算。
  - Frozen 不自动解冻。
  - 必须写 OperationLog。
- BrandSubscription 额度调整必须写 OperationLog。
- 冻结/解冻必须写 OperationLog。
- Payment callback 幂等、金额不一致、找不到订单、验签失败。

测试写法：

- 简单规则用 table-driven tests。
- 需要数据库事务的流程用集成测试。
- helper 必须 `t.Helper()`。
- teardown 用 `t.Cleanup()`。
- 不比较错误字符串；比较错误码或错误语义。
- 复杂结构比较优先 `cmp.Diff`。

示例失败信息格式：

```go
t.Errorf("ManualRenewBrandSubscription(%d, %+v) expires_at = %v, want %v", id, input, got.ExpiresAt, want)
```

## 11. 当前实现缺口

基于当前第一片 admin/platform 实现，后续 Go 侧需要继续补齐：

- 微信支付 Native provider adapter。
- 支付回调验签、幂等和订阅开通事务。
- 平台公开注册：创建 Pending Brand、Pending Staff、首购订单。
- Brand/Staff/Learner 额度硬限制 middleware 或 service guard。
- BrandSubscription 自动进入 GracePeriod、Restricted、Expired 的定时任务。
- 关键 service/repository 单元测试和集成测试。
- API OpenAPI/Swagger 注释同步。

## 12. 代码评审清单

每次后端 PR 至少检查：

- 是否把业务规则写进 Handler。
- 是否把 GORM model 泄露到 domain/application。
- 是否漏传 `context.Context`。
- 是否在手动商业关键操作中漏写 OperationLog。
- 是否在事务外分步修改支付、订阅、权益、预约核心事实。
- 是否使用 float 表示钱。
- 是否用字符串比较 `err.Error()`。
- 是否新增了泛包名，例如 `util`、`helper`、`common`。
- 是否运行 `gofmt` 和 `go test ./...`。
