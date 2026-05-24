# Mini Schedule 背景资料

整理日期：2026-05-24

本文是 `pds` 需求仓库的背景资料，用于后续需求清单、版本计划和产品决策对齐。内容来自当前工作区根目录、`backend/`、`web/` 以及已有上下文文档。

## 1. 仓库边界

当前工作区路径：

```text
/Users/liushan/Documents/zkw/mini_schedule
```

当前工作区是多仓结构：

```text
mini_schedule/
├── backend/   # Golang 后端仓库，独立 .git
├── web/       # Next.js 前端 monorepo，独立 .git
├── pds/       # 需求与产品文档仓库，独立 .git
├── CONTEXT.md # 产品术语与领域关系
├── CLAUDE.md  # 项目部署、前后端通信、本地开发上下文
├── Agent.md
├── Agent-DB.md
├── Golang-SAAS.md
├── TS.md
└── PRD.md
```

整理前检查时，`backend/`、`web/`、`pds/` 三个 git 工作区均为干净状态。本文新增后，`pds/` 会出现本文件的未跟踪变更。

## 2. 根目录资料

### 2.1 `CONTEXT.md`

根目录 `CONTEXT.md` 是当前最权威的产品术语来源。应优先使用以下术语：

| 术语 | 含义 |
|---|---|
| Brand | 提供健身课程、计划和训练服务的租户组织 |
| Platform Administrator | 管理品牌入驻和平台管理员账号的平台级操作人员 |
| Platform Backoffice | 平台管理员使用的平台级运营后台 |
| Brand Administrator | 品牌侧工作人员，管理本品牌学员、课程和训练记录 |
| Brand Backoffice | 品牌管理员使用的品牌级运营后台 |
| Learner | 消费品牌课程并记录训练活动的终端用户 |

已确定关系：

- Platform Administrator 管理 0 个或多个 Brand。
- Brand 拥有 1 个或多个 Brand Administrator。
- Brand 拥有 0 个或多个 Learner。
- MVP 模型中，Learner 只属于 1 个 Brand。
- Platform Backoffice 与 Brand Backoffice 是不同操作界面。

用词注意：

- 避免把 Brand 称为 merchant、gym、account。
- 避免把 Learner 称为 customer、member、C 端用户。
- 避免单独使用 admin，必须说明是 Platform Administrator 还是 Brand Administrator。

### 2.2 `CLAUDE.md`

`CLAUDE.md` 记录了当前工程上下文：

- 产品是健身 SaaS 平台。
- 后端是 Golang 多服务结构，分为 `api-brand`、`api-app`、`api-admin`。
- 前端是 `web/` 下的 pnpm + Turborepo monorepo。
- 部署意向：后端 Railway，前端 Vercel。
- 前后端通信链路：浏览器到 Vercel，再通过 Next.js rewrite 到 Railway 上的 Golang 服务。
- 本地开发：后端从 `backend/` 执行 `go run ./cmd/api-*`，前端从 `web/` 执行 `pnpm dev`。

注意：`CLAUDE.md` 里写到“两个前端应用”，但当前代码实际有 `admin`、`brand`、`app` 三个前端应用，应以后者为准。

### 2.3 其他根目录文件

- `Agent.md`：项目协作角色说明，提到需求收集、后台开发、UI 设计、数据库设计专家。
- `Agent-DB.md`：数据库专家提示词和偏好，偏 PostgreSQL、Redis、阿里云。
- `Golang-SAAS.md`：后端开发提示词，强调 Gin、GORM、PostgreSQL、Redis、多租户隔离和可观测性。
- `TS.md`：前端开发提示词，强调 Next.js App Router、TypeScript、Tailwind CSS、shadcn/ui、Zustand。
- `PRD.md`：偏产品经理工作流示例，目前更像 PRD 生成方法模板，不是当前产品的正式 PRD。

## 3. 后端现状

后端路径：

```text
backend/
```

### 3.1 技术栈

| 方向 | 当前选择 |
|---|---|
| 语言 | Go 1.25 |
| Web 框架 | Gin |
| ORM | GORM v2 |
| 数据库 | PostgreSQL |
| 缓存 | Redis |
| 配置 | Viper |
| 参数验证 | validator |
| 认证 | JWT |
| 依赖注入 | Google Wire |
| API 文档 | Swagger |
| 数据库迁移 | golang-migrate SQL 文件 |

### 3.2 服务划分

后端按三端拆成独立 Gin 实例：

| 服务 | 路径 | 默认端口 | 路由前缀 | 使用者 |
|---|---|---:|---|---|
| api-admin | `cmd/api-admin` | 8083 | `/api/v1/admin` | Platform Administrator |
| api-brand | `cmd/api-brand` | 8081 | `/api/v1/brand` | Brand Administrator |
| api-app | `cmd/api-app` | 8082 | `/api/v1/app` | Learner |

### 3.3 分层结构

```text
backend/
├── cmd/
│   ├── api-admin/
│   ├── api-brand/
│   └── api-app/
├── internal/
│   ├── domain/
│   │   ├── brand/
│   │   ├── course/
│   │   ├── training/
│   │   └── user/
│   ├── application/
│   ├── infrastructure/
│   │   ├── cache/
│   │   ├── config/
│   │   └── persistence/
│   └── interfaces/
│       ├── admin/
│       ├── brand/
│       ├── app/
│       └── middleware/
├── migrations/
├── configs/
└── pkg/
```

### 3.4 主要领域对象

| 对象 | 当前字段概要 |
|---|---|
| Brand | name、logo_url、contact_name、contact_phone、status |
| BrandUser | brand_id、phone、password_hash、name、status |
| AppUser | brand_id、openid、phone、nickname、avatar_url、vip_level、status |
| AdminUser | username、password_hash、role、status |
| Course | brand_id、title、description、cover_url、difficulty、duration_min、type、status |
| TrainingRecord | user_id、brand_id、course_id、duration_min、calories、notes、completed_at |

### 3.5 数据模型

当前迁移文件包含：

- `brands`
- `brand_users`
- `app_users`
- `admin_users`
- `courses`
- `training_records`

当前数据隔离方式是单库单 schema，通过 `brand_id` 区分租户数据。`backend/Memory.md` 记录目标包含 PostgreSQL RLS，但当前迁移文件尚未创建 RLS policy。

### 3.6 API 概览

统一响应格式：

```json
{
  "code": "OK",
  "message": "success",
  "data": {}
}
```

分页响应格式：

```json
{
  "code": "OK",
  "message": "success",
  "data": {
    "items": [],
    "total": 100,
    "page": 1,
    "page_size": 20,
    "total_page": 5
  }
}
```

已实现或已规划的核心接口：

| 端 | 能力 |
|---|---|
| Platform Backoffice | 平台管理员登录、登出、品牌创建/列表/详情/更新/状态更新、平台管理员创建/列表 |
| Brand Backoffice | 品牌管理员登录、Learner 列表/详情/创建、课程创建/列表/详情/更新/删除/状态更新、训练记录列表 |
| Learner App | 微信登录、个人资料查看/更新、已发布课程列表/详情、创建训练记录、查看我的训练记录 |

### 3.7 已知后端注意事项

- `backend/API.md` 快速测试里登录密码写的是 `admin123`，但迁移种子 `000002_seed_admin.up.sql` 注释写初始化密码是 `Admin@123456`，需要校准。
- `backend/Memory.md` 写 MVP 权限模型是单品牌管理员，后期 RBAC；当前 `AdminUser` 已有 `super_admin/operator/support` 三种角色。
- `backend/Memory.md` 写目标数据隔离包含 PostgreSQL RLS；当前迁移里还没有 RLS policy。

## 4. 前端现状

前端路径：

```text
web/
```

### 4.1 Monorepo 结构

```text
web/
├── apps/
│   ├── admin/ # Platform Backoffice
│   ├── brand/ # Brand Backoffice
│   └── app/   # Learner App Web
├── packages/
│   ├── api/
│   ├── admin-system/
│   ├── config/
│   └── types/
├── package.json
├── pnpm-workspace.yaml
└── turbo.json
```

### 4.2 技术栈

| 方向 | 当前选择 |
|---|---|
| 框架 | Next.js 15 |
| UI 运行时 | React 19 |
| 语言 | TypeScript |
| 样式 | Tailwind CSS |
| UI 基础 | shadcn/ui、Radix UI、lucide-react |
| 服务端状态 | TanStack Query v5 |
| 客户端状态 | Zustand |
| 表单 | react-hook-form、Zod |
| Toast | Sonner |
| 构建编排 | Turborepo |
| 包管理器 | pnpm 9.15.0 |
| Node 要求 | >= 20 |

### 4.3 三端应用

| 应用 | 包名 | 默认端口 | 对接后端 |
|---|---|---:|---|
| admin | `@mini-schedule/admin` | 3001 | `api-admin` |
| brand | `@mini-schedule/brand` | 3002 | `api-brand` |
| app | `@mini-schedule/app` | 3003 | `api-app` |

### 4.4 当前页面

`apps/admin` 当前包含：

- 登录页
- 仪表盘
- 品牌管理
- 品牌详情
- 管理员管理
- 消息页

`apps/brand` 当前包含：

- 登录页
- 仪表盘
- Learner 管理
- Learner 详情
- 课程管理
- 课程详情
- 训练记录
- 消息页
- 未授权页

`apps/app` 当前包含：

- 登录页
- 仪表盘
- 课程列表
- 课程详情
- 训练记录
- 个人资料

### 4.5 共享包

| 包 | 作用 |
|---|---|
| `@mini-schedule/api` | 统一 HTTP client、认证 hooks、admin/brand/app API hooks |
| `@mini-schedule/types` | 共享 TypeScript 类型 |
| `@mini-schedule/admin-system` | 后台壳层、页面模板、表格、状态徽标、消息中心等后台 UI 组件 |
| `@mini-schedule/config` | 共享 Tailwind/ESLint 配置 |

### 4.6 认证与请求

- `@mini-schedule/api/auth` 使用 Zustand 持久化认证状态，localStorage key 为 `auth-storage`。
- `packages/api/src/client.ts` 统一解析后端 `{code, message, data}` 响应。
- 当前前端 client 以 `code === "OK"` 判断成功。
- Brand 和 App 端通过 localStorage token 添加 `Authorization: Bearer <token>`。
- Admin 端 client 对 `/api/v1/admin/` 使用 cookie auth 路径策略，具体登录态链路需要继续确认。

### 4.7 已知前端注意事项

- `web/Memory.md` 旧描述写 `code === null` 表示成功，但当前 `client.ts` 已按 `code === "OK"` 实现，应以后者为准。
- `web/packages/types/src/index.ts` 部分字段名与后端不同，例如 `duration_minutes`/`duration_min`、`calories_burned`/`calories`、`trained_at`/`completed_at`，需要在联调前校准。
- `BrandStatus` 类型写了 `active/suspended/pending`，后端 Brand 状态是 `active/inactive/pending`，需要统一。
- `AppUser` 类型里使用 `openid/unionid/role`，后端实体使用 `open_id` 数据库列、JSON 字段 `openid`、`vip_level`，需要统一前后端契约。

## 5. 产品信息

### 5.1 产品定位

Mini Schedule 是一个多品牌健身 SaaS 平台，帮助平台运营方管理品牌入驻，帮助品牌管理自己的学员、课程和训练记录，并让学员消费品牌发布的健身课程。

当前产品不是教练个人入驻平台，而是面向品牌组织的多租户 SaaS。

### 5.2 用户角色

| 角色 | 使用界面 | 核心目标 |
|---|---|---|
| Platform Administrator | Platform Backoffice | 管理品牌入驻、品牌状态、平台管理员账号 |
| Brand Administrator | Brand Backoffice | 管理本品牌 Learner、课程、训练记录 |
| Learner | Learner App Web | 登录、查看课程、完成训练、记录训练数据 |

### 5.3 MVP 范围

从当前后端、前端和项目记忆看，MVP 范围包括：

- 品牌入驻和品牌基础资料管理。
- 平台管理员账号管理。
- 品牌管理员登录。
- 品牌侧 Learner 管理。
- 品牌侧课程管理。
- 课程发布状态管理。
- Learner 微信登录。
- Learner 查看已发布课程。
- Learner 创建训练记录。
- 品牌侧查看训练记录。
- JWT 认证和基础权限隔离。
- 单库单 schema + `brand_id` 的租户数据隔离。

### 5.4 非 MVP 或后续能力

以下能力在文档中出现为后续方向，但当前代码没有形成完整实现：

- 完整 RBAC。
- PostgreSQL RLS policy。
- Prometheus 指标和完整可观测性。
- 异步任务 Worker Pool。
- Brand Backoffice 的 Electron 打包。
- 更复杂的消息中心。
- 学员训练计划、订阅计费、审计日志、租户限流与配额。
- AI 健身教练能力。根目录 `PRD.md` 提到智能健身教练 App 的 PRD 生成示例，但未成为当前代码的已实现产品范围。

### 5.5 关键产品边界

- Brand 是租户组织，不是单个教练。
- Learner 在 MVP 中只属于一个 Brand。
- Platform Backoffice 不直接管理 Learner 的日常训练细节，平台侧重点是品牌和平台运营。
- Brand Backoffice 管理品牌内的 Learner、课程、训练记录。
- Learner App 只展示当前 Brand 作用域内可见的课程与训练记录。

## 6. 需要校准的问题

这些问题会影响后续需求清单和接口联调，建议在进入具体 PRD 或迭代计划前确认：

1. 管理员初始密码以 `backend/API.md` 的 `admin123` 为准，还是以迁移文件的 `Admin@123456` 为准？
2. Brand 状态枚举统一使用 `inactive` 还是 `suspended`？
3. 训练记录字段统一使用后端命名 `duration_min/calories/completed_at`，还是前端命名 `duration_minutes/calories_burned/trained_at`？
4. Admin 端认证最终使用 Bearer token、cookie，还是二者并存？
5. MVP 是否需要真正启用 PostgreSQL RLS，还是先用应用层 `brand_id` 过滤？
6. `PRD.md` 中的“智能健身教练 App”是否只是示例，还是未来产品方向？

## 7. 信息来源

- `pds/README.md`
- `CONTEXT.md`
- `CLAUDE.md`
- `Agent.md`
- `Agent-DB.md`
- `Golang-SAAS.md`
- `TS.md`
- `PRD.md`
- `backend/API.md`
- `backend/Memory.md`
- `backend/go.mod`
- `backend/migrations/*.sql`
- `backend/internal/domain/*`
- `backend/internal/interfaces/*`
- `backend/pkg/response/response.go`
- `web/Memory.md`
- `web/package.json`
- `web/pnpm-workspace.yaml`
- `web/turbo.json`
- `web/apps/*/package.json`
- `web/packages/api/src/*`
- `web/packages/types/src/index.ts`
