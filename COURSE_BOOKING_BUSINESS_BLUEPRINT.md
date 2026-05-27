# 课程预约业务蓝图

整理日期：2026-05-25

状态：v0.2，已补齐为完整需求蓝图

本文是课程预约项目的完整需求蓝图，用于承接后续 PRD、数据库设计、接口设计和实施计划。本文不直接描述代码实现，但会明确产品边界、角色、菜单、核心对象、状态流、第一版范围、验收场景和后续需求池。

参考来源：

- 当前 `pds/BACKGROUND.md`
- 根目录 `CONTEXT.md`
- 当前讨论中已确认的业务决策
- 只读参考远端分支 `remotes/origin/claude/read-background-md-3G8pb` 中的 `SCHEDULE_BOOKING.md`、`RBAC_PERMISSION.md`、`MEMBERSHIP.md`

## 0. 文档概览

| 项 | 内容 |
|---|---|
| 文档名称 | 课程预约业务蓝图 |
| 当前版本 | v0.2 |
| 当前状态 | 需求蓝图已收束，可进入 PRD/技术方案拆解 |
| 目标读者 | 产品、设计、后端、前端、测试、运营 |
| 项目类型 | 通用课程预约 SaaS |
| 第一版终端 | Platform Backoffice、Brand Backoffice、Learner 微信小程序 |
| 第一版重点 | 品牌自助购买平台套餐、品牌初始化、课程排课、学员权益预约、员工签到、基础运营看板 |

### 0.1 修订记录

| 版本 | 日期 | 修改内容 |
|---|---|---|
| v0.1 | 2026-05-25 | 基于讨论形成业务蓝图、对象架构、菜单架构和后续需求池 |
| v0.2 | 2026-05-25 | 结合上下文复核遗漏，补充功能域详细规格、默认权限矩阵、状态流、通知事件、验收场景和非功能需求 |
| v0.3 | 2026-05-26 | 补充平台商业化第一阶段实现边界：微信支付 Native 先行、Pending Brand/Pending Staff、手机号注册与短信验证码策略 |

### 0.2 需求完整性复核结论

已提交的 v0.1 覆盖了业务架构和关键决策，但作为完整需求文档仍缺少以下内容。本版本已在后续章节补齐：

- 平台自助注册、支付订单、订阅状态和额度限制的详细规则。
- 品牌后台核心模块的功能边界和异常处理。
- 员工默认角色、菜单权限、数据权限、操作权限矩阵。
- 学员小程序从进入品牌空间到预约、取消、候补、查看权益的完整流程。
- Learner Entitlement 的产品模板、学员持有实例、锁定、消耗、调整流水关系。
- 简单候补的转正责任边界：第一版由员工手动转正，自动限时确认后续做。
- 微信订阅消息和后台站内通知的事件清单。
- 第一版验收场景、测试场景和非功能要求。

## 1. 产品定位

Mini Schedule 从原先偏健身行业的课程/训练管理，升级为 **通用课程预约 SaaS**。

第一版不把底层模型锁死在健身行业。健身、英语培训、艺术培训、其他课程服务行业都应通过行业术语、课程分类、权益模板和页面文案适配，而不是通过不同底层数据模型适配。

核心业务主线：

```text
平台套餐
-> 品牌/机构自助注册
-> 微信支付/支付宝支付
-> 开通品牌订阅
-> 品牌初始化
-> 创建 Location
-> 创建员工/授课员工
-> 创建课程和权益
-> 创建课程场次
-> 学员微信小程序预约
-> 员工签到
-> 权益消耗 / 爽约处理
```

## 2. 通用术语

| 通用术语 | 含义 | 健身行业显示 | 英语培训显示 |
|---|---|---|---|
| Platform | SaaS 平台 | 平台 | 平台 |
| Brand | 入驻品牌/机构/租户 | 健身品牌 | 培训机构 |
| Location | 品牌下的服务网点 | 门店 | 校区 |
| Location Resource | Location 内可被排课占用的资源 | 瑜伽房/团操室 | 教室/线上教室 |
| Staff | 品牌员工账号 | 员工 | 员工 |
| Instructor | 具备授课能力的员工 | 教练 | 老师 |
| Learner | 预约和上课的终端用户 | 学员/会员 | 学员 |
| CourseTemplate | 课程模板，不绑定具体时间 | 课程 | 课程 |
| ClassSession | 一次具体课程场次 | 团课场次 | 班课/课节 |
| Booking | 学员对场次的预约 | 预约 | 预约 |
| Waitlist | 满员后的候补队列 | 候补 | 候补 |
| Attendance | 到课/签到事实 | 签到/到课 | 到课 |
| SessionRecord | 上课/服务履约记录 | 训练记录 | 上课记录 |
| Learner Entitlement | 学员可用于预约的权益 | 课包/会员卡 | 课包/会员卡 |

术语原则：

- 底层模型使用通用术语。
- 行业展示层可以把 Location 显示为门店或校区。
- 行业展示层可以把 Instructor 显示为教练或老师。
- 行业展示层可以把 SessionRecord 显示为训练记录或上课记录。

## 3. 四层角色架构

### 3.1 平台层

平台层服务 Platform Administrator，负责平台商业化和品牌管理。

核心职责：

- 管理 SaaS Plan
- 查看品牌注册和订阅状态
- 查看平台订单和支付流水
- 手动调整品牌套餐、额度、状态
- 管理平台管理员账号
- 管理行业/术语模板
- 查看平台运营基础看板

### 3.2 品牌层

品牌层服务品牌负责人、品牌管理员和品牌运营人员，负责品牌业务配置和日常运营。

核心职责：

- 完成品牌初始化
- 管理 Location
- 管理 Location Resource
- 管理员工、授课员工和权限
- 管理学员
- 管理课程、课程分类、课程可用 Location
- 管理学员权益/会员卡
- 排课、预约、候补、签到、爽约处理
- 查看品牌运营基础看板

### 3.3 员工层

员工层不是独立 App，第一版共用 Brand Backoffice，通过权限显示不同菜单和操作。

员工账号统一为 Staff。Staff 可以：

- 属于一个 Brand
- 被分配到多个 Location
- 拥有品牌级角色
- 在不同 Location 拥有不同 Location 级角色
- 通过角色模板、数据范围、操作点获得最终权限

授课员工不是独立账号，而是 Staff 的业务档案：

```text
Staff 员工账号
└── InstructorProfile 授课员工档案
```

只有具备授课职责的 Staff 需要 InstructorProfile。

### 3.4 学员层

学员端第一版以微信小程序为主要形态。

学员通过品牌专属二维码、小程序路径参数或机构码进入某个 Brand 的学员空间。

账号模型需要支持同一微信身份绑定多个 Brand：

```text
LearnerIdentity 微信身份
└── BrandLearnerProfile 品牌内学员档案
```

各 Brand 的权益、预约、上课记录相互隔离。

## 4. 平台层需求

### 4.1 SaaS Plan

平台套餐第一版按以下维度控制：

- Location 数
- 员工席位数
- 学员数
- 功能模块
- 是否支持多 Location
- 是否支持高级权限
- 是否支持报表
- 是否支持候补

不建议第一版限制课程数或场次数。课程预约产品的核心价值是让品牌持续排课，过早限制课程/场次会伤害产品使用动机。

### 4.2 品牌自助入驻

第一版支持品牌自助注册并购买 SaaS Plan。

流程：

```text
品牌访问注册入口
-> 创建品牌负责人账号
-> 填写品牌资料
-> 选择 SaaS Plan
-> 选择月付/年付
-> 微信支付/支付宝支付
-> 支付成功
-> 自动开通品牌订阅
-> 进入品牌初始化向导
```

### 4.3 支付

第一版支付模型同时保留微信支付和支付宝通道字段。平台商业化第一阶段先实现微信支付 Native 扫码支付，支付宝作为后续支付通道接入，不改变 SaaSPlanOrder、PaymentTransaction 和 BrandSubscription 的核心模型。

选择微信支付 Native 先行的原因：

- 平台套餐购买发生在 Web 注册/购买场景，扫码支付适配度高。
- Native 支付不依赖小程序 openid，适合品牌负责人公开购买入口。
- 第一阶段应优先跑通下单、验签、回调幂等、金额校验和订阅开通闭环。
- 支付宝后续复用同一订单和支付流水模型接入。

第一版需要具备支付安全底线：

- 支付订单
- 支付状态机
- 支付回调验签
- 支付回调幂等
- 金额校验
- 第三方支付单号记录
- 支付日志
- 人工补偿入口

第一版不做：

- 发票
- 合同
- 风控审核
- 自动退款
- 渠道分佣
- 优惠码
- 自动续费

### 4.4 购买周期

SaaS Plan 第一版支持：

- 月付
- 年付
- 年付可配置折扣或年付价格

第一版不做：

- 季付
- 半年付
- 自定义周期
- 套餐升级补差价
- 套餐降级退款
- 自动续费

### 4.5 套餐额度超限和到期

额度超限：

- 不删除已有数据
- 不影响已有课程、预约、员工、学员
- 禁止新增超出额度的资源
- 给品牌负责人提示升级套餐
- 平台管理员可手动调整套餐或额度

套餐到期：

- 到期前提醒
- 到期后进入宽限期
- 宽限期内继续可用，但提示续费
- 宽限期结束后品牌后台进入受限状态
- 平台管理员可手动延期、续费、冻结、解冻

受限状态建议：

- 允许登录
- 允许查看历史数据
- 禁止新增课程场次、员工、学员、权益
- 禁止学员新预约
- 允许处理已有预约：查看、取消、签到、爽约确认

## 5. 品牌层需求

### 5.1 品牌初始化向导

品牌支付成功后进入初始化向导。

初始化向导：

1. 完善品牌资料
2. 创建第一个 Location
3. 创建员工/授课员工
4. 创建课程分类
5. 创建课程
6. 创建学员权益模板
7. 创建第一节课程场次
8. 发布学员小程序入口/二维码

初始化向导的目标是让品牌跑通第一条业务链路：

```text
有网点
-> 有员工/老师
-> 有课程
-> 有权益
-> 有可预约场次
-> 学员能预约
```

### 5.2 Location 管理

Location 是品牌下的服务网点。

字段建议：

- 名称
- 地址
- 联系电话
- 营业状态
- 是否启用
- 备注

学员有一个主归属 Location，但允许跨 Location 预约。

### 5.3 Location Resource 管理

Location Resource 是 Location 下可被排课占用的资源。

字段建议：

- 所属 Location
- 名称
- 类型：教室、场地、线上、设备资源
- 容量
- 状态：启用、停用
- 备注

场次可选绑定 Location Resource。

绑定后：

- 同一资源同一时间不能被两个有效场次占用。
- 资源停用后不能继续新排课。
- 资源容量可作为场次容量默认值。
- 场次容量可以覆盖资源容量。

不绑定资源时：

- 只检查授课员工时间冲突。
- 不做资源冲突检查。

### 5.4 课程分类

第一版做品牌自定义课程分类，不做平台统一行业分类。

CourseCategory：

- Brand
- 名称
- 排序
- 状态
- 可选颜色/图标
- 是否显示在小程序筛选中

CourseTemplate 可绑定一个或多个分类。

行业模板后续可预置分类。

### 5.5 课程模板

CourseTemplate 是品牌级课程模板，不直接归属某个 Location。

字段建议：

- Brand
- 名称
- 简介
- 封面
- 分类
- 难度/级别
- 默认时长
- 默认容量
- 状态

### 5.6 课程可用 Location

第一版必须支持 CourseLocationAvailability。

作用：

- 控制某个 CourseTemplate 可在哪些 Location 排课。
- 排课时课程选择器只展示当前 Location 可用课程。
- 后端再次校验，避免绕过前端。

规则：

- 创建课程时选择适用 Location。
- 默认全选当前启用 Location。
- 编辑课程时可调整适用 Location。
- 移除某 Location 后，不影响该 Location 已经生成的历史/未来场次。
- 新增 Location 后，不自动继承旧课程，需要手动启用课程。

### 5.7 学员权益

第一版将“课包、体验包、会员卡”统一为 Learner Entitlement。

第一版支持：

1. 次数/课时包
2. 单次体验包
3. 基础会员卡

基础会员卡只做：

- 有效期
- 每日预约上限
- 每周预约上限
- 每月预约上限
- 同时未完成预约上限
- 适用课程范围
- 适用 Location 范围

复杂会员卡后续做：

- 冻结/解冻
- 请假延期
- 转卡
- 共享卡/家庭卡
- 分时段会员
- 自动续费
- 会员等级权益

多个可用权益时：

- 系统自动选择。
- 品牌后台可人工调整。
- 学员端第一版不让学员自己选择权益。

自动选择优先级：

1. 指定课程/指定活动权益
2. 最早过期权益
3. 次数/课时包优先于会员卡
4. 体验包只在符合体验条件时使用

### 5.8 员工代预约

学员自助预约必须有可用权益。

品牌后台/员工代预约可以无权益占位。

默认可代预约角色：

- 品牌负责人
- 品牌管理员
- 店长
- 前台

默认不可代预约：

- Instructor
- 课程运营
- 财务/售后

权限体系保留操作点：

```text
booking.create_assisted
```

无权益占位限制：

- 只能由有权限员工创建。
- 默认不允许超过场次容量。
- 需要填写原因。
- 可标记为待补权益。
- 签到时如果仍无权益，系统提醒员工处理。

## 6. 员工和权限需求

### 6.1 员工体系

员工统一为 Staff。

Staff 支持：

- 跨多个 Location
- 品牌级角色
- Location 级角色
- 数据权限绑定具体分配关系
- 操作权限来自角色模板

### 6.2 角色模型

第一版采用：

```text
角色模板 + 数据范围 + 操作点
```

不做完全自定义 ACL。

品牌级角色示例：

- 品牌负责人
- 品牌管理员
- 课程运营
- 财务/售后

Location 级角色示例：

- 店长
- 前台
- Instructor
- 门店协助

一个员工可以：

- 没有品牌级角色，只是某个 Location 的 Instructor。
- 有品牌级角色，同时兼任某个 Location 店长。
- 在 A Location 是 Instructor，在 B Location 是店长。

### 6.3 授课员工档案

Instructor 是 Staff 的角色，但需要独立 InstructorProfile。

InstructorProfile：

- 关联 Staff
- 展示昵称
- 头像
- 简介
- 擅长方向
- 可授课程
- 证书/资质
- 学员端是否展示
- 排课可用状态

前台、店长、课程运营、财务/售后不需要独立业务档案。

### 6.4 数据权限

数据权限绑定员工的具体分配关系，而不是写死在角色模板上。

示例：

```text
员工 A 在 Location 1：Instructor，只看自己授课场次
员工 A 在 Location 2：店长，看 Location 2 全部场次
员工 B 是品牌课程运营：看全品牌课程和排课
```

数据范围建议：

- 全品牌
- 指定 Location
- 自己负责的课程/场次
- 自己创建或处理的数据

### 6.5 操作权限

第一版操作点示例：

- 查看
- 新增
- 编辑
- 删除
- 导出
- 审核
- 代预约
- 取消预约
- 手动签到
- 调课
- 爽约确认
- 权益人工调整

## 7. 学员小程序需求

### 7.1 小程序入口

第一版学员端以微信小程序为主要形态。

入口方式：

- 品牌专属小程序二维码
- 带 `brand_id` 或 `brand_code` 的小程序路径
- 手动输入机构码

流程：

```text
打开小程序
-> 识别品牌参数
-> 微信登录
-> 手机号绑定/授权
-> 创建或关联该品牌下 BrandLearnerProfile
-> 进入该品牌学员首页
```

无品牌参数时：

- 显示输入机构码。
- 或显示最近访问的品牌列表。

### 7.2 学员端菜单

学员小程序第一版菜单：

- 首页
- 课程表
- 我的预约
- 我的权益
- 上课记录
- 我的

### 7.3 学员预约

学员自助预约必须通过权益校验。

预约成功后：

- 创建 Booking。
- 创建 EntitlementHold。
- 学员端显示已预约。
- 品牌后台可查看本次预约使用的权益。

不可预约原因应明确展示：

- 无可用权益
- 场次已满
- 已超过预约截止时间
- 取消后不可重复预约
- 频次上限
- 同时预约上限
- 学员状态不可用

### 7.4 学员取消预约

取消规则由品牌默认规则和场次覆盖规则共同决定。

取消成功后：

- Booking 变更为已取消。
- EntitlementHold 释放或按规则转为消耗。
- 如有候补，触发候补处理。

### 7.5 微信订阅消息

员工/品牌后台使用站内通知和后台消息中心。

学员小程序使用微信订阅消息触达。

第一版学员订阅消息：

- 预约成功
- 预约取消
- 上课提醒
- 场次取消
- 候补转正

实现前需要按微信当前订阅消息规则核实授权和模板限制。

## 8. 核心业务对象

### 8.1 平台层对象

- SaaSPlan
- SaaSPlanOrder
- BrandSubscription
- PaymentTransaction
- PlatformAdmin

### 8.2 品牌组织对象

- Brand
- Location
- LocationResource
- Staff
- StaffLocationAssignment
- RoleTemplate
- Permission
- InstructorProfile

### 8.3 学员对象

- LearnerIdentity
- BrandLearnerProfile
- LearnerStaffAssignment
- LearnerEntitlement

### 8.4 课程预约对象

- CourseCategory
- CourseTemplate
- CourseLocationAvailability
- ClassSession
- RecurringSchedule
- Booking
- WaitlistEntry
- Attendance
- EntitlementHold
- EntitlementConsumption
- SessionRecord

### 8.5 通知和审计对象

- Notification
- OperationLog
- SystemSetting

### 8.6 对象拆分原则

Booking、Attendance、EntitlementHold、EntitlementConsumption 不合并成一个大表。

拆分理由：

- Booking 表达学员占了一个名额。
- Attendance 表达到课事实。
- EntitlementHold 表达临时占用权益。
- EntitlementConsumption 表达最终消耗权益。
- 拆分后取消、爽约、补扣、补退、人工调整更清晰。

UI 可以简化，但对象语义不能混。

## 9. 预约与权益状态流

### 9.1 场次状态

第一版场次不做审批，只做状态流转。

建议状态：

```text
Draft 草稿
-> Scheduled 已发布/已排课，可预约
-> InProgress 进行中
-> Completed 已完成
-> Cancelled 已取消
```

派生状态：

- Full：已预约人数达到容量
- Closed：已超过预约截止时间

### 9.2 Booking 状态

```text
Booked 已预约
-> Cancelled 已取消
-> Attended 已到课
-> PendingNoShow 待确认爽约
-> NoShow 已爽约
```

场次结束后，未签到预约进入 PendingNoShow，由员工确认后处理。

第一版不建议自动爽约扣课，除非品牌规则开启。

### 9.3 Entitlement 状态

预约成功：

```text
LearnerEntitlement
-> EntitlementHold
```

到课后：

```text
EntitlementHold
-> EntitlementConsumption
```

取消时：

```text
EntitlementHold
-> Released
```

爽约时：

```text
EntitlementHold
-> Released 或 Consumption
```

具体按品牌默认规则和场次覆盖规则决定。

## 10. 排课规则

### 10.1 单次排课

排课必须包含：

- CourseTemplate
- Location
- Instructor
- 开始时间
- 结束时间或时长
- 容量

可选包含：

- LocationResource
- 场次预约规则覆盖
- 备注

### 10.2 简单循环排课

第一版做简单循环排课。

支持：

- 每周重复
- 选择周几
- 开始日期
- 结束日期或重复周数
- 开始时间
- 结束时间或时长
- 批量生成场次
- 逐个做冲突检查
- 冲突场次跳过并返回清单

不做：

- 复杂 RRULE
- 节假日跳过
- 隔周课
- 批量调课
- 整批取消
- 模板化课表

### 10.3 排课冲突检查

第一版做较完整冲突检查，但不考虑跨校区通勤时间。

必须检查：

- 同一 Instructor 同一时间不能被安排到两个场次。
- 同一 LocationResource 同一时间不能被两个有效场次占用。
- Instructor 不可用时间不能排课。
- Instructor 请假/停用期间不能排课。
- Resource 停用后不能排课。
- CourseTemplate 必须允许在目标 Location 排课。
- 场次开始时间必须早于结束时间。

暂不检查：

- 跨校区通勤时间。
- 员工每日最大工时。
- 员工每周最大课时。
- 课间休息强制间隔。
- 不同课程类型准备时间。

### 10.4 员工可用性

第一版做轻量 Staff Availability。

支持：

- 默认可排课。
- 固定不可用时间段。
- 临时请假/不可排课时间段。

不做复杂班表轮班。

## 11. 预约规则

### 11.1 规则层级

取消和预约规则由：

```text
品牌默认规则 + 场次覆盖规则
```

共同决定。

### 11.2 品牌默认规则

建议配置：

- 最多提前预约时间
- 最少提前预约时间
- 最晚取消时间
- 取消后是否释放权益
- 爽约是否扣课
- 单学员每日预约上限
- 单学员每周预约上限
- 同时未完成预约上限
- 候补默认开关
- 候补上限

### 11.3 场次覆盖规则

建议配置：

- 是否允许取消
- 最晚取消时间
- 取消是否退回权益
- 爽约是否扣课
- 是否允许候补
- 候补上限

## 12. 候补机制

第一版做简单候补，完整候补后续做。

第一版支持：

- 场次满员后允许学员加入候补。
- 候补按加入时间排序。
- 品牌/员工可查看候补名单。
- 有人取消后，可按顺序转正。
- 转正时再锁定学员权益。

第一版不做：

- 自动通知候补学员确认。
- 限时确认。
- 超时自动跳过。
- 候补时提前锁定权益。
- 多渠道通知编排。

后续完整候补机制可吸收参考分支中的 notified、confirm_deadline、expired、定时扫描等设计。

## 13. 签到和爽约

### 13.1 签到方式

第一版员工手动签到为主。

签到权限：

- 前台可签到本 Location 所有场次。
- 店长可签到自己管理 Location 所有场次。
- Instructor 可签到自己授课的场次。
- 品牌管理员/品牌负责人可签到全品牌场次。

第一版不做：

- 学员扫码自助签到
- 地理围栏签到
- 人脸识别签到
- 签到码动态刷新

### 13.2 签到后动作

签到后：

```text
Booking -> Attended
EntitlementHold -> EntitlementConsumption
SessionRecord -> 生成上课/履约记录
```

无权益占位预约签到时：

- 系统提示该预约无可用权益。
- 有权限员工可人工确认签到。
- 记录异常原因。
- 后续进入补权益/补扣处理。

### 13.3 爽约处理

场次结束后，未签到预约进入 PendingNoShow。

员工确认爽约后：

- Booking -> NoShow
- EntitlementHold 按规则释放或消耗
- 记录处理人和处理原因

## 14. 菜单架构

### 14.1 平台后台菜单

- 平台概览
- 品牌管理
- 套餐管理
- 订单与支付
- 订阅与额度
- 平台管理员
- 行业/术语模板
- 系统配置

### 14.2 品牌后台菜单

- 品牌概览
- 初始化向导
- Location 管理
- 资源/教室管理
- 员工管理
- 角色权限
- 学员管理
- 权益/会员卡
- 课程分类
- 课程管理
- 课表排课
- 预约管理
- 签到与爽约
- 通知消息
- 报表
- 品牌设置

### 14.3 员工视图菜单

员工视图在 Brand Backoffice 内按权限显示：

- 今日工作台
- 我的课表
- 我的场次
- 签到处理
- 预约处理
- 我的学员
- 消息通知

### 14.4 学员小程序菜单

- 首页
- 课程表
- 我的预约
- 我的权益
- 上课记录
- 我的

## 15. 基础报表

第一版只做基础运营看板，不做深财务和绩效报表。

平台看板：

- 品牌总数
- 活跃品牌数
- 付费品牌数
- 即将到期品牌数
- 本月订单收入
- 套餐分布
- Location 用量
- 员工席位用量
- 学员用量

品牌看板：

- 预约数
- 到课数
- 取消数
- 爽约数
- 上座率
- 热门课程
- Location 场次/预约分布
- Instructor 授课场次数
- 权益锁定/消耗次数
- 待处理爽约数
- 候补人数

第一版不做：

- 收入分析
- 支付对账
- 退款报表
- 教练绩效薪酬
- 课程转化率
- 学员留存
- 渠道来源
- LTV
- 经营驾驶舱
- 导出中心

## 16. 第一版范围

第一版核心范围：

- 通用课程预约 SaaS 术语和模型
- 品牌自助注册
- 微信支付/支付宝购买平台套餐
- SaaS Plan、订单、订阅、额度
- 品牌初始化向导
- Location 和 LocationResource
- Staff、InstructorProfile、角色模板、数据范围、操作点
- CourseCategory、CourseTemplate、CourseLocationAvailability
- LearnerIdentity、BrandLearnerProfile
- LearnerEntitlement：次数/课时包、单次体验包、基础会员卡
- ClassSession 单次排课
- 简单循环排课
- 基础排课冲突检查
- Booking、WaitlistEntry、Attendance、EntitlementHold、EntitlementConsumption、SessionRecord
- 品牌默认预约规则 + 场次规则覆盖
- 简单候补
- 员工手动签到
- 待确认爽约
- 微信小程序学员端
- 微信订阅消息
- 员工/品牌后台站内通知
- 基础运营看板

## 17. 后续需求池

后续需求池统一存放“需要实现但不进入第一版核心”的能力。

### 17.1 预约能力

- 一对一预约
- 授课员工可预约时间
- 学员选择老师/员工
- 私教/一对一课时包
- 时间冲突检测扩展
- 改约/请假/补课规则
- 完整候补机制
- 候补通知、限时确认、过期顺延
- 学员自助签到
- 二维码签到
- 地理围栏签到
- 人脸识别签到

### 17.2 会员权益

- 复杂会员卡
- 冻结/解冻
- 请假延期
- 转卡
- 共享卡/家庭卡
- 分时段会员
- 黑名单课程
- 自动续费
- 会员等级权益
- 储值余额
- 优惠券

### 17.3 平台商业化

- 发票
- 合同
- 风控审核
- 支付宝支付通道接入
- 已入驻品牌自助续费
- 自动退款
- 渠道分佣
- 优惠码
- 季付/半年付
- 套餐升级补差价
- 套餐降级退款
- 自动续费

### 17.4 权限和组织

- 完全自定义 ACL
- 字段级权限
- 审批流
- 权限审计
- JWT 黑名单
- PostgreSQL RLS policy

### 17.5 行业和终端

- 完整行业模板体系
- 健身行业模板
- 英语培训行业模板
- 艺术课行业模板
- 学员 H5
- 学员 Web
- 学员 App
- 抖音小程序

### 17.6 通知和报表

- 短信
- 邮件
- 企业微信/飞书通知
- 通知模板管理
- 多渠道通知编排
- 自动营销提醒
- 收入分析
- 支付对账
- 退款报表
- 教练绩效薪酬
- 学员留存
- LTV
- 导出中心

### 17.7 排班增强

- 复杂 RRULE
- 节假日跳过
- 隔周课
- 批量调课
- 整批取消
- 模板化课表
- 跨校区通勤时间
- 员工每日最大工时
- 员工每周最大课时
- 课间休息强制间隔

## 18. 参考分支吸收与不采纳

参考分支：

```text
remotes/origin/claude/read-background-md-3G8pb
```

### 18.1 吸收

- CourseTemplate 与 ClassSession 拆分。
- 课程可用网点关系，改为 CourseLocationAvailability。
- LocationResource 参与排课冲突检查。
- 简单循环排课的批量生成和冲突清单。
- 预约规则中的提前预约、最晚取消、候补上限。
- RBAC 权限码思想。
- 会员权益的 Location 范围和课程范围。

### 18.2 调整后吸收

- Store/Room/Coach 调整为 Location/LocationResource/Instructor。
- CheckIn 调整为 Attendance。
- MembershipProduct 调整到 LearnerEntitlement 体系中。
- 预约扣减从直接扣减改为 Hold -> Consumption。
- 自定义角色不进入第一版完整 ACL，只保留角色模板、数据范围和操作点。
- 候补完整状态机后置，第一版只做简单候补。

### 18.3 不采纳

- Phase 1 免费预约、不校验权益。
- Phase 1 同期做一对一预约。
- Phase 1 同期接入抖音小程序通知。
- Learner 只能属于一个 Brand。
- 会员卡冻结/解冻第一版。
- 在线购卡后置到 Phase 3。
- 以健身行业 Store/Coach 作为底层术语。

## 19. 已确认决策

| 决策点 | 结论 |
|---|---|
| 产品定位 | 通用课程预约 SaaS，健身/英语培训为行业模板 |
| 员工跨 Location | 支持 |
| 员工角色 | 品牌级角色 + Location 级角色并存 |
| Instructor | Staff 角色 + 独立 InstructorProfile |
| 员工入口 | 第一版共用 Brand Backoffice |
| 权限 | 角色模板 + 数据范围 + 操作点 |
| 数据权限 | 绑定员工具体分配关系 |
| 学员服务关系 | 可选绑定员工 |
| 学员 Location | 主归属 Location + 可跨 Location 预约 |
| 课程类型 | 班课/团课场次为主，一对一后续 |
| 平台套餐 | SaaS Plan 限制 Location、员工席位、学员数、功能模块 |
| 品牌入驻 | 自助注册 + 微信/支付宝支付 |
| SaaS 购买周期 | 月付 + 年付 |
| 品牌初始化 | 第一版加入初始化向导 |
| 学员端 | 微信小程序优先 |
| 多品牌学员身份 | 底层支持同一微信身份绑定多个 Brand |
| 权益 | 次数/课时包 + 单次体验包 + 基础会员卡 |
| 权益消耗 | 预约锁定，到课正式消耗 |
| 多权益选择 | 系统自动选择，后台可人工调整 |
| 后台代预约 | 可无权益占位 |
| 签到 | 员工手动签到 |
| 爽约 | 场次结束后待确认爽约 |
| 通知 | 员工站内通知，学员微信订阅消息 |
| 循环排课 | 第一版做简单周重复 |
| CourseLocationAvailability | 第一版核心 |
| CourseCategory | 品牌自定义分类 |
| 容量 | 总容量 + 候补上限 |
| 报表 | 基础运营看板 |

## 20. 详细功能域规格

本节把前文蓝图拆成可落地的功能域。第一版应优先完成闭环，不追求一次性覆盖所有行业差异。

### 20.1 平台公开注册与套餐购买

品牌自助购买入口面向品牌负责人。

第一阶段公开购买入口只做功能型购买页，不做完整营销官网。

页面范围：

- 套餐列表：展示套餐名称、价格、额度和功能开关。
- 注册表单：品牌名称、行业类型、联系人、手机号、短信验证码和密码。
- 购买周期选择：月付或年付。
- 微信支付二维码页：展示订单金额、二维码、倒计时和支付状态。
- 支付成功页：引导进入 Brand Backoffice 或 Brand Onboarding Wizard。

第一阶段不做：

- 大型营销首页。
- 客户案例。
- SEO 内容页。
- 复杂定价对比动画。
- 优惠活动页。

主流程：

1. 品牌负责人进入注册入口。
2. 填写负责人手机号、短信验证码和登录密码。第一阶段不做微信授权登录。
3. 填写品牌名称、行业类型、联系人信息。
4. 系统创建 Pending Brand 和 Pending Staff。Pending Brand 不可运营，Pending Staff 不可登录 Brand Backoffice。
5. 选择 SaaS Plan 和购买周期：月付或年付。
6. 系统按 SaaS Plan 周期价格生成 SaaSPlanOrder，订单金额必须等于套餐表价格。
7. 系统调用微信支付 Native 下单并返回二维码 code_url。
8. 支付成功回调通过验签、幂等、金额校验和订单状态校验后，系统开通 BrandSubscription。
9. 系统激活 Brand，激活品牌负责人 Staff，并授予品牌负责人角色。
10. 用户进入 Brand Onboarding Wizard。

异常流程：

- 支付未完成：订单保持待支付，Pending Brand 不进入正式可用状态，可回到订单继续支付。
- 支付失败：订单记录失败原因，允许重新发起支付。
- 支付成功但回调延迟：前端显示支付处理中，后台通过支付查询或回调完成订阅开通。
- 回调重复：系统按订单号和支付单号做幂等，不重复开通订阅。
- 支付金额不一致：订单进入异常状态，不开通订阅，平台管理员人工处理。
- 短信验证码：开发环境允许固定验证码或 Mock Provider；生产环境必须配置短信 Provider，未配置时不得允许公网真实注册。

微信支付回调开通规则：

- 回调验签失败：拒绝处理，不修改订单和订阅。
- 找不到订单：记录 PaymentCallbackLog 为失败或忽略，不开通订阅。
- 订单已 Paid：按幂等成功返回，不重复开通订阅。
- 支付金额不一致：订单进入异常状态，不开通订阅。
- 金额一致且订单为 PendingPayment：在同一个数据库事务内完成订单支付确认、PaymentTransaction 成功记录、BrandSubscription 创建或激活、Pending Brand 激活、Pending Staff 激活为 Brand Owner、OperationLog 写入。
- 事务内任一步失败：整体回滚，记录 callback 错误，平台管理员可通过人工补偿入口处理。
- 第一阶段不引入异步队列延迟开通；后续可在支付量增大后拆为可靠消息或任务队列。

第一阶段订单来源：

- `public_signup_first_purchase`：新品牌公开注册首购。
- `admin_manual_compensation`：平台管理员人工补偿开通、续期或额度调整。

后续版本订单来源：

- `brand_self_service_renewal`：已入驻品牌自助续费。
- `brand_self_service_upgrade`：已入驻品牌自助升级。
- `brand_self_service_downgrade`：已入驻品牌自助降级。

订阅有效期规则：

- 新品牌首购从支付成功时间开始计算：月付为 `paid_at + 1 month`，年付为 `paid_at + 1 year`。
- 平台人工续期时，如果当前订阅未过期，从当前 `expires_at` 顺延；如果当前订阅已过期，从操作时间重新开始。
- 平台默认宽限期为 7 天，可作为系统配置调整。
- 订阅到期后进入 `GracePeriod`，宽限期结束后进入 `Restricted`。
- 平台冻结 `Frozen` 不自动延长订阅有效期；如需补偿延期，由平台管理员手动续期或调整有效期。

### 20.2 平台套餐管理

平台管理员可管理 SaaS Plan。

SaaS Plan 字段：

- 套餐名称
- 套餐描述
- 月付价格
- 年付价格或年付折扣
- 最大 Location 数
- 最大员工席位数
- 最大学员数
- 功能模块开关
- 是否启用
- 排序

功能要求：

- 新建套餐。
- 编辑未下架套餐。
- 启用/停用套餐。
- 查看套餐关联品牌数量。
- 停用套餐不影响已购买品牌的当前订阅，但不能新购。
- 第一阶段不提供套餐物理删除；已被订单或订阅引用的 SaaSPlan 只能停用，不能删除。

订阅快照规则：

- BrandSubscription 创建时，从 SaaSPlan 和 SaaSPlanFeature 复制额度与功能开关快照。
- SaaSPlan 调价、调额度、停用后，不影响既有 BrandSubscription。
- 已成交订单和当前订阅展示使用 BrandSubscription 快照，不直接读取 SaaSPlan 当前值。
- 平台管理员可对单个 BrandSubscription 手动调整额度或功能开关，必须记录 OperationLog。
- 新购或后续续费时，才按当时 SaaSPlan 的最新价格、额度和功能开关生成新的订单与订阅快照。

### 20.3 平台订单与订阅管理

SaaSPlanOrder 状态：

```text
PendingPayment 待支付
-> Paid 已支付
-> Closed 已关闭
-> Failed 支付失败
-> Refunding 退款中
-> Refunded 已退款
```

BrandSubscription 状态：

```text
Active 正常
-> GracePeriod 宽限期
-> Restricted 受限
-> Frozen 平台冻结
-> Expired 已过期
```

平台管理员能力：

- 查看订单列表和支付流水。
- 查看品牌当前订阅和额度使用。
- 手动续期。
- 手动调整额度。
- 冻结/解冻品牌。
- 处理支付异常订单。

额度硬限制执行规则：

- 额度限制按 BrandSubscription 快照执行。
- Location、Staff、Learner 超过额度时，禁止新增对应资源。
- 已有 Location、Staff、Learner 不删除、不自动停用，不影响历史课程、预约和履约记录。
- 平台管理员可手动把额度调低到低于当前使用量；系统应标记超限并继续禁止新增对应资源。
- 订阅进入 Restricted 后，禁止新增课程场次、员工、学员、权益和学员新预约。
- Restricted 状态仍允许查看历史数据、取消预约、签到和爽约确认，以保证已有业务善后。

订阅手动调整审计规则：

- 第一阶段不单独建设 SubscriptionAdjustment 专表。
- 平台管理员手动续期、调额度、冻结、解冻、支付异常补偿开通时，必须写入 OperationLog。
- OperationLog 需记录操作人、动作类型、调整前值、调整后值、原因和关联订单/支付流水。
- 不允许静默直接修改 BrandSubscription 商业关键字段。

### 20.4 品牌初始化向导

初始化向导是第一版核心体验。

完成条件：

- 品牌资料已完善。
- 至少 1 个启用 Location。
- 至少 1 个启用 Staff。
- 至少 1 个 InstructorProfile。
- 至少 1 个 CourseCategory。
- 至少 1 个 CourseTemplate。
- 至少 1 个 LearnerEntitlement 模板。
- 至少 1 个 Scheduled ClassSession。
- 已生成品牌小程序入口二维码或机构码。

允许跳过：

- 可跳过部分步骤进入后台，但后台首页继续显示开通进度。
- 未完成关键步骤前，小程序课程表为空，需要明确展示配置提示。

### 20.5 Location 与 Resource 管理

Location 管理能力：

- 创建 Location。
- 编辑 Location。
- 启用/停用 Location。
- 查看 Location 下资源、员工、课程场次和学员数量。

停用规则：

- 停用 Location 后，不允许新建该 Location 的场次。
- 已存在的未来场次不自动取消，由有权限员工处理。
- 学员主归属 Location 可被停用，但后台需提示迁移学员归属。

LocationResource 管理能力：

- 创建资源。
- 编辑资源容量和基础信息。
- 启用/停用资源。
- 查看资源占用日历。

停用规则：

- 停用资源后，不允许新排课使用该资源。
- 已排未来场次不自动取消，但编辑场次时需要提示资源已停用。

### 20.6 员工管理

员工管理能力：

- 创建 Staff。
- 编辑 Staff 基础信息。
- 启用/停用 Staff。
- 分配品牌级角色。
- 分配 Location 级角色。
- 配置数据范围。
- 创建或编辑 InstructorProfile。
- 设置员工不可用时间和临时请假。

停用规则：

- 停用 Staff 后不能登录。
- 停用 Staff 后不能被新排课。
- 如果 Staff 已绑定未来场次，停用时必须提示，并要求处理未来场次。

### 20.7 学员管理

学员管理能力：

- 查看 BrandLearnerProfile 列表。
- 创建或导入学员。
- 编辑学员基础信息、主归属 Location、备注和标签。
- 绑定服务关系：服务顾问、销售顾问、主 Instructor、跟进人。
- 查看学员预约、候补、权益、上课记录。
- 冻结/解冻学员。

冻结规则：

- 冻结后学员不能自助预约。
- 冻结不自动取消已有预约。
- 冻结前若存在未来预约，系统提示操作人处理。

### 20.8 课程与分类管理

课程分类能力：

- 新建分类。
- 编辑分类。
- 启用/停用分类。
- 配置是否在小程序筛选中展示。

课程模板能力：

- 新建课程模板。
- 编辑课程基础信息。
- 配置分类、默认时长、默认容量、难度/级别。
- 配置适用 Location。
- 发布/下架课程。

课程状态：

```text
Draft 草稿
-> Published 已发布
-> Archived 已归档
```

规则：

- 只有 Published 课程可用于新排课。
- Archived 课程不影响历史场次。
- CourseLocationAvailability 控制课程是否可在目标 Location 排课。

### 20.9 权益管理

LearnerEntitlement 拆为两个层次：

```text
EntitlementProduct 权益模板
LearnerEntitlement 学员持有权益
```

EntitlementProduct 能力：

- 创建次数/课时包。
- 创建单次体验包。
- 创建基础会员卡。
- 配置适用 Location。
- 配置适用 CourseTemplate。
- 配置有效期。
- 配置频次上限。
- 启用/停用。

LearnerEntitlement 能力：

- 给学员开通权益。
- 查看剩余次数、锁定次数、消耗次数、有效期。
- 人工调整权益。
- 查看权益流水。

权益流水类型：

- Grant 开通
- Hold 预约锁定
- Release 取消释放
- Consume 到课消耗
- NoShowConsume 爽约消耗
- ManualAdjust 人工调整

人工调整要求：

- 必须填写原因。
- 必须记录操作人。
- 必须进入 OperationLog。

### 20.10 排课管理

排课能力：

- 单次排课。
- 简单循环排课。
- 编辑未开始场次。
- 取消场次。
- 查看按日期、Location、Instructor、课程筛选的课表。

编辑限制：

- 已有预约的场次，修改时间、Instructor、Resource、容量时必须提示影响。
- 容量不能小于当前有效预约数。
- 已完成或已取消场次不可编辑。

取消场次：

- 场次状态变为 Cancelled。
- 未到课 Booking 变为 Cancelled。
- EntitlementHold 释放。
- WaitlistEntry 失效。
- 触发学员微信订阅消息和后台站内通知。

### 20.11 预约管理

预约管理能力：

- 查看预约列表。
- 按场次、Location、学员、状态筛选。
- 员工代预约。
- 员工代取消。
- 查看预约绑定权益。
- 处理无权益占位预约。

Booking 来源：

- LearnerSelfService 学员自助预约。
- StaffAssisted 员工代预约。

员工代预约规则：

- 可绑定权益并锁定。
- 可无权益占位。
- 无权益占位必须填写原因。
- 无权益占位不可绕过容量。

### 20.12 签到、到课和履约记录

签到能力：

- 按场次查看预约学员。
- 标记到课。
- 撤销误签到应进入后续需求池，第一版通过人工权益调整处理。
- 标记待确认爽约。
- 确认爽约。

到课后生成：

- Attendance
- EntitlementConsumption
- SessionRecord

SessionRecord 第一版只记录基础履约信息，不承载复杂课程评价或训练数据。

### 20.13 通知消息

后台站内通知对象：

- 品牌负责人
- 品牌管理员
- 店长
- 前台
- Instructor

后台站内通知事件：

- 新预约
- 取消预约
- 候补变化
- 待确认爽约
- 场次取消
- 支付/订阅异常
- 套餐额度即将超限

学员微信订阅消息事件：

- 预约成功
- 预约取消
- 上课提醒
- 场次取消
- 候补转正

失败处理：

- 微信订阅消息发送失败不影响主流程。
- 发送失败需要记录日志。
- 学员端“消息/通知记录”可展示关键业务消息。

## 21. 默认权限矩阵

第一版不做完全自定义 ACL，但需要明确默认角色模板。

### 21.1 品牌级角色

| 权限域 | 品牌负责人 | 品牌管理员 | 课程运营 | 财务/售后 |
|---|---|---|---|---|
| 品牌设置 | 全部 | 编辑 | 查看 | 查看 |
| Location | 全部 | 全部 | 查看 | 查看 |
| 员工与权限 | 全部 | 全部 | 查看 | 不可见 |
| 学员 | 全部 | 全部 | 查看 | 查看 |
| 权益/会员卡 | 全部 | 全部 | 查看 | 全部 |
| 课程分类/课程 | 全部 | 全部 | 全部 | 查看 |
| 排课 | 全部 | 全部 | 全部 | 查看 |
| 预约 | 全部 | 全部 | 查看 | 查看 |
| 签到/爽约 | 全部 | 全部 | 查看 | 不可见 |
| 报表 | 全部 | 全部 | 查看 | 财务相关查看 |

### 21.2 Location 级角色

| 权限域 | 店长 | 前台 | Instructor | 门店协助 |
|---|---|---|---|---|
| Location 数据 | 指定 Location | 指定 Location | 指定 Location | 指定 Location |
| 员工查看 | 本 Location | 不可见 | 不可见 | 不可见 |
| 学员查看 | 本 Location | 本 Location | 自己相关学员 | 按配置 |
| 课程查看 | 本 Location 可用课程 | 查看 | 查看 | 查看 |
| 排课 | 本 Location | 不可操作 | 查看自己场次 | 按配置 |
| 预约 | 查看/代预约/代取消 | 查看/代预约/代取消 | 查看自己场次预约 | 按配置 |
| 签到 | 本 Location | 本 Location | 自己场次 | 按配置 |
| 爽约确认 | 本 Location | 本 Location | 自己场次 | 按配置 |
| 权益调整 | 不默认开放 | 不默认开放 | 不可见 | 不可见 |
| 报表 | 本 Location 基础报表 | 不可见 | 自己场次数据 | 不可见 |

### 21.3 权限计算规则

员工最终权限：

```text
品牌级角色权限
+ Location 级角色权限
+ 具体分配关系的数据范围
= 实际可见菜单和可执行操作
```

规则：

- 多角色权限取并集。
- 数据范围不因角色模板固定，按具体分配关系计算。
- 任何写操作都必须同时满足操作权限和数据范围。
- 前端隐藏无权限菜单，后端仍必须做权限校验。

## 22. 关键流程规格

### 22.1 学员自助预约流程

```text
学员进入课程场次详情
-> 系统检查场次状态、预约窗口、容量、学员状态
-> 系统检查可用权益
-> 系统按优先级选择权益
-> 创建 Booking
-> 创建 EntitlementHold
-> 返回预约成功
-> 发送微信订阅消息
```

失败场景：

- 无权益：返回无可用权益，引导联系机构或购买权益。
- 满员：引导加入候补。
- 超过预约窗口：不可预约。
- 频次超限：提示会员卡频次超限。
- 同一时间已有预约：提示时间冲突。

### 22.2 员工代预约流程

```text
员工选择学员和场次
-> 系统检查员工操作权限和数据范围
-> 系统检查场次容量
-> 员工选择绑定权益或无权益占位
-> 创建 Booking
-> 绑定权益时创建 EntitlementHold
-> 无权益时记录占位原因
```

无权益占位进入异常提示队列，签到时继续提示。

### 22.3 取消预约流程

```text
发起取消
-> 校验取消权限
-> 校验品牌默认规则和场次覆盖规则
-> Booking -> Cancelled
-> EntitlementHold 按规则 Release 或 Consumption
-> 如存在候补，生成候补待转正提示
-> 发送通知
```

取消来源：

- 学员自助取消。
- 员工代取消。
- 场次取消触发批量取消。

### 22.4 简单候补流程

第一版候补不做自动限时确认。

```text
场次满员
-> 学员加入候补
-> WaitlistEntry 按时间排序
-> 有人取消后，系统标记候补队首为可转正
-> 有权限员工手动转正候补
-> 系统重新校验容量和权益
-> 创建 Booking 和 EntitlementHold
-> WaitlistEntry -> Promoted
-> 发送候补转正微信订阅消息
```

规则：

- 候补时不锁定权益。
- 转正时才锁定权益。
- 如果队首学员转正时无可用权益，员工可跳过或改为无权益占位，需记录原因。
- 自动通知、限时确认、过期顺延进入后续需求池。

### 22.5 签到流程

```text
员工进入场次签到页
-> 查看有效 Booking 列表
-> 标记学员到课
-> 创建 Attendance
-> EntitlementHold -> EntitlementConsumption
-> 创建 SessionRecord
```

异常：

- 无权益占位：提示员工人工确认或补权益。
- 重复签到：系统阻止重复创建 Attendance。
- 场次已取消：不可签到。

### 22.6 爽约流程

```text
场次结束
-> 未签到 Booking -> PendingNoShow
-> 员工查看待处理爽约
-> 员工确认爽约
-> Booking -> NoShow
-> EntitlementHold 按规则 Release 或 Consumption
-> 记录操作日志
```

第一版避免自动扣课造成争议，默认需要员工确认。

## 23. 验收标准和测试场景

### 23.1 平台商业化验收

- 平台管理员可以创建启用状态的 SaaS Plan。
- 品牌负责人可以自助选择月付或年付套餐。
- 支付成功后自动创建 BrandSubscription。
- 支付回调重复不会重复开通订阅。
- 套餐额度超限后，系统禁止新增对应资源但不影响存量数据。
- 套餐到期进入宽限期，宽限期结束后禁止新增预约和资源。

### 23.2 品牌初始化验收

- 新品牌支付成功后进入初始化向导。
- 完成 Location、Instructor、课程、权益、场次后，开通进度显示完成。
- 未创建场次时，学员小程序课程表显示空状态和配置提示。

### 23.3 排课验收

- 同一 Instructor 同一时间不能被排入两个场次。
- 同一 LocationResource 同一时间不能被两个场次占用。
- CourseTemplate 未启用目标 Location 时，不允许排课。
- 简单循环排课遇到冲突时跳过冲突场次，并返回冲突清单。
- 已有预约的场次，容量不能改小到低于有效预约数。

### 23.4 预约验收

- 学员有有效权益时可以预约可用场次。
- 学员无权益时不能自助预约。
- 预约成功后生成 Booking 和 EntitlementHold。
- 场次满员后学员可加入候补。
- 员工代预约可创建无权益占位，并记录原因。
- 学员取消预约后释放或消耗权益，按规则处理。

### 23.5 签到和爽约验收

- 前台可签到本 Location 场次。
- Instructor 只能签到自己授课的场次。
- 签到后生成 Attendance、EntitlementConsumption、SessionRecord。
- 场次结束后未签到 Booking 进入 PendingNoShow。
- 员工确认爽约后按规则释放或消耗权益。

### 23.6 权限验收

- 无权限菜单在前端不展示。
- 无权限 API 调用返回拒绝。
- 员工只能访问自己数据范围内的 Location、场次、预约和学员。
- 同一员工在不同 Location 的角色不同，数据范围按分配关系计算。

### 23.7 学员小程序验收

- 学员通过品牌二维码进入后绑定正确 Brand。
- 同一微信身份可绑定多个 Brand，数据互不串。
- 学员可以查看课程表、我的预约、我的权益、上课记录。
- 微信订阅消息发送失败不影响预约主流程。

## 24. 非功能需求

### 24.1 权限与数据隔离

- 所有品牌数据必须带 Brand 维度隔离。
- 所有 Location 数据必须受员工数据权限过滤。
- 前端权限隐藏只作为体验优化，后端必须强校验。
- 学员端接口不得允许客户端传入 learner_id 查询他人数据。

### 24.2 并发与一致性

- 预约创建必须防止超卖。
- EntitlementHold 创建必须与 Booking 创建保持一致。
- 支付回调必须幂等。
- 签到不能重复消费同一 EntitlementHold。
- 候补转正必须重新校验容量和权益。

### 24.3 审计

以下操作必须记录 OperationLog：

- 支付异常人工处理。
- 品牌订阅和额度人工调整。
- 员工角色和数据范围变更。
- 学员权益人工调整。
- 无权益代预约。
- 场次取消。
- 爽约确认。

### 24.4 可观测性

第一版至少记录：

- 支付成功率和回调失败日志。
- 预约成功/失败原因。
- 排课冲突原因。
- 微信订阅消息发送失败原因。
- 权限拒绝日志。

### 24.5 兼容性

- 学员端第一版以微信小程序为准。
- Brand Backoffice 和 Platform Backoffice 继续按现有 Web 后台方向设计。
- 其他终端进入后续需求池。

## 25. 需求优先级

### 25.1 Must

- 品牌自助注册和套餐购买。
- 微信支付/支付宝支付。
- BrandSubscription 和额度限制。
- Location、LocationResource。
- Staff、InstructorProfile、角色模板、数据范围。
- LearnerIdentity、BrandLearnerProfile。
- CourseCategory、CourseTemplate、CourseLocationAvailability。
- LearnerEntitlement 基础模型。
- ClassSession 单次和简单循环排课。
- Booking、EntitlementHold、Attendance、EntitlementConsumption。
- 学员微信小程序预约。
- 员工手动签到。
- 待确认爽约。

### 25.2 Should

- 简单候补。
- 基础运营看板。
- 后台站内通知。
- 微信订阅消息。
- 品牌初始化向导。
- 员工不可用时间。

### 25.3 Could

- 行业术语模板的基础配置。
- 学员最近访问品牌列表。
- 课程分类颜色/图标。
- Resource 占用日历。

### 25.4 Won't in V1

- 一对一预约。
- 复杂会员卡。
- 完整候补机制。
- 学员自助签到。
- 发票、合同、风控。
- 完全自定义 ACL。
- 深度财务报表和教练薪酬。

## 26. 用户角色矩阵与用户故事

### 26.1 用户角色矩阵

| 角色 | 核心诉求 | 主要痛点 | 第一版满足方式 |
|---|---|---|---|
| Platform Administrator | 管理品牌增长、套餐收入和订阅状态 | 手工开通品牌、续费和额度管理容易出错 | 自助购买、订单支付、订阅与额度后台 |
| Brand Owner | 快速开通业务并掌握经营状态 | 新系统配置路径长，不知道先做什么 | 初始化向导、品牌看板、套餐/权益/排课闭环 |
| Brand Administrator | 管理品牌日常运营 | 多 Location、员工权限、课程和预约容易混乱 | Location、员工权限、课程、预约、权益统一管理 |
| Location Manager | 管理本 Location 的课表和服务质量 | 只能靠人工表格安排课程和统计到课 | Location 数据权限、排课、签到、基础报表 |
| Receptionist | 处理学员咨询、代预约、签到和异常 | 电话/到店预约难以和线上预约统一 | 员工代预约、无权益占位、签到处理 |
| Instructor | 查看自己的课程和学员，完成到课确认 | 不希望看到无关后台菜单，也不能被重复排课 | 我的课表、自己场次预约、签到权限、冲突检查 |
| Learner | 在微信小程序完成预约、取消、查看权益 | 不清楚自己能约什么课、预约是否成功、权益还剩多少 | 品牌空间、课程表、我的预约、我的权益、订阅消息 |

### 26.2 核心用户故事

| 角色 | 用户故事 | 验收标准 |
|---|---|---|
| Platform Administrator | 作为平台管理员，我想配置 SaaS Plan，以便品牌可以自助购买合适套餐 | 套餐可设置月付/年付、额度和功能模块；停用套餐不影响已订阅品牌 |
| Brand Owner | 作为品牌负责人，我想购买套餐后按向导完成初始化，以便尽快上线课程预约 | 支付成功后进入向导；完成关键步骤后可以发布第一个可预约场次 |
| Brand Administrator | 作为品牌管理员，我想配置员工角色和数据范围，以便不同员工只能操作自己的职责范围 | 员工菜单和数据按角色、Location、操作点控制；越权 API 被拒绝 |
| Location Manager | 作为 Location 负责人，我想给本 Location 排课并避免资源冲突，以便课表真实可执行 | 同一 Instructor 或 Resource 时间冲突时创建失败或跳过冲突场次 |
| Receptionist | 作为前台，我想帮学员代预约并处理无权益占位，以便线下咨询和电话预约能进入系统 | 代预约可选择权益或无权益占位；无权益占位必须记录原因 |
| Instructor | 作为授课员工，我想只看到自己的课程和学员，以便快速完成签到 | Instructor 只能查看自己场次和对应预约，并可完成签到 |
| Learner | 作为学员，我想在微信小程序查看可约课程并预约，以便不用联系前台也能完成上课安排 | 有可用权益时可预约；预约成功生成 Booking 和 EntitlementHold；无权益时给出明确原因 |

## 27. 指标与埋点

### 27.1 北极星指标

第一版北极星指标建议：

```text
月成功履约人次 = 每月完成 Attendance 的 Booking 数
```

选择理由：

- 能同时反映品牌是否在排课、学员是否在预约、员工是否在履约。
- 比单纯注册品牌数或创建课程数更接近真实业务价值。
- 对健身、英语培训等行业都适用。

### 27.2 输入指标

平台侧：

- 新注册品牌数
- 支付成功品牌数
- 活跃品牌数
- 套餐续费率
- 套餐到期未续费数

品牌侧：

- 已发布场次数
- 预约数
- 到课数
- 取消数
- 爽约数
- 上座率
- 权益消耗次数
- 候补人数

学员侧：

- 小程序访问人数
- 课程详情访问数
- 预约成功率
- 取消率
- 订阅消息授权率

### 27.3 核心埋点事件

| 事件 | 触发时机 | 关键字段 |
|---|---|---|
| `brand_signup_started` | 品牌开始注册 | source、industry |
| `saas_plan_selected` | 选择平台套餐 | plan_id、billing_cycle |
| `saas_order_paid` | 平台套餐支付成功 | order_id、plan_id、amount、payment_channel |
| `brand_onboarding_step_completed` | 完成初始化步骤 | brand_id、step_key |
| `location_created` | 创建 Location | brand_id、location_id |
| `staff_created` | 创建员工 | brand_id、staff_id、role_type |
| `course_template_created` | 创建课程模板 | brand_id、course_id、category_id |
| `class_session_created` | 创建课程场次 | brand_id、location_id、course_id、instructor_id |
| `booking_created` | 创建预约 | source、brand_id、session_id、learner_id、entitlement_type |
| `booking_cancelled` | 取消预约 | source、reason、release_mode |
| `waitlist_joined` | 加入候补 | session_id、learner_id、position |
| `waitlist_promoted` | 候补转正 | session_id、learner_id、operator_id |
| `attendance_marked` | 标记到课 | session_id、learner_id、operator_id |
| `no_show_confirmed` | 确认爽约 | session_id、learner_id、operator_id、consume_mode |
| `entitlement_consumed` | 权益正式消耗 | entitlement_id、session_id、consume_type |

## 28. 风险评估

| 风险 | 影响 | 缓解措施 |
|---|---|---|
| 第一版范围过大：平台支付、品牌后台、小程序、权益、排课同时推进 | 研发周期拉长，交付风险高 | 按闭环拆里程碑：先平台购买和品牌初始化，再预约履约，再报表增强 |
| 微信订阅消息授权限制导致触达不稳定 | 学员可能收不到上课提醒或候补转正 | 业务主流程不依赖通知；小程序内保留消息记录；实现前核实微信当前规则 |
| 支付回调和订阅开通不一致 | 品牌付款后无法使用或重复开通 | 回调验签、幂等、金额校验、支付查询补偿、人工处理入口 |
| 预约并发导致超卖 | 场次名额错误，影响品牌信任 | Booking 创建和容量扣减必须事务化；候补转正重新校验容量 |
| 权益锁定和消耗不一致 | 学员余额错误，引发投诉 | Booking、Hold、Consumption 需要状态联动和操作日志；异常用人工调整补偿 |
| 员工多角色和数据范围复杂 | 容易出现越权或看不到数据 | 权限模型先限制为角色模板 + 数据范围 + 操作点；后端强校验 |
| 通用术语过抽象，品牌理解成本高 | 不同行业客户配置时困惑 | 底层通用，前端展示通过行业术语模板转换 |
| 简单候补不含自动限时确认 | 热门课程运营效率不够高 | 第一版由员工手动转正，完整候补机制进入后续需求池 |
| 无权益占位被滥用 | 品牌经营数据和权益消耗失真 | 只开放给指定角色；必须填写原因；后台看板显示异常占位 |

## 29. 接口域范围

本文不定义具体 API 路径，但后续接口设计应按以下接口域拆分：

第一阶段平台商业化接口必须拆为 Public API 和 Platform Admin API。

Public API 面向品牌负责人公开注册和支付：

- SaaS Plan 公开展示。
- Signup SMS Code。
- Signup Order 创建。
- Signup Order 状态查询。
- WeChat Pay Native 下单结果返回。
- WeChat Pay Callback / Notify。

Platform Admin API 面向平台管理员运营和补偿：

- SaaS Plan 管理。
- SaaS Plan Order 查询。
- PaymentTransaction 查询。
- PaymentCallbackLog 查询。
- BrandSubscription 查询。
- BrandSubscription 手动续期。
- BrandSubscription 额度调整。
- BrandSubscription 冻结/解冻。
- 支付异常订单人工补偿。
- OperationLog 查询。
- Platform Dashboard Summary。

Platform Dashboard Summary 第一阶段只统计商业化闭环：

- 品牌总数。
- Pending Brand 数。
- Active Brand 数。
- 当前有效订阅数。
- 7 天内到期订阅数。
- Restricted / Frozen 品牌数。
- 今日订单数。
- 今日支付成功金额。
- 待处理异常订单数。
- 待处理支付回调失败数。

后续再补课程运营统计：

- 月成功履约人次。
- 上座率。
- 课程发布数。
- Instructor 维度。
- 学员活跃。
- 行业分析。

平台侧：

- SaaS Plan
- Brand Signup
- SaaS Plan Order
- Payment Callback
- Brand Subscription
- Platform Dashboard

品牌侧：

- Brand Onboarding
- Location
- LocationResource
- Staff
- RoleTemplate / Permission
- InstructorProfile
- BrandLearnerProfile
- LearnerStaffAssignment
- EntitlementProduct
- LearnerEntitlement
- CourseCategory
- CourseTemplate
- CourseLocationAvailability
- ClassSession
- RecurringSchedule
- Booking
- Waitlist
- Attendance
- SessionRecord
- Notification
- Brand Dashboard

学员小程序侧：

- WeChat Login
- Brand Binding
- Course Calendar
- Session Detail
- Booking
- Waitlist
- My Bookings
- My Entitlements
- My Session Records
- WeChat Subscribe Message Authorization
