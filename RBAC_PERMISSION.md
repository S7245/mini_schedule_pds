# 角色、菜单与权限体系 — 产品设计规格（PDS）

整理日期：2026-05-24
状态：Phase 1 草稿，待评审

---

## 目录

1. [背景与目标](#1-背景与目标)
2. [整体层级关系](#2-整体层级关系)
3. [平台层角色设计（Platform Backoffice）](#3-平台层角色设计platform-backoffice)
4. [品牌层角色体系（Brand Backoffice）](#4-品牌层角色体系brand-backoffice)
5. [权限码体系](#5-权限码体系)
6. [Brand Backoffice 菜单与权限矩阵](#6-brand-backoffice-菜单与权限矩阵)
7. [数据权限设计](#7-数据权限设计)
8. [数据库模型设计](#8-数据库模型设计)
9. [学员层功能入口（Learner App）](#9-学员层功能入口learner-app)
10. [JWT Payload 设计](#10-jwt-payload-设计)
11. [分期交付计划](#11-分期交付计划)
12. [已确认决策](#12-已确认决策)

---

## 1. 背景与目标

### 1.1 背景

Mini Schedule 当前品牌层（`brand_users` 表）没有角色字段，所有 Brand 操作人员默认等同于 Brand Administrator 全权限。随着 SCHEDULE_BOOKING.md 引入了 Coach、Receptionist 等新角色，需要系统性地建立角色、菜单、权限体系，明确：

- 哪些人能看哪些菜单；
- 哪些人能执行哪些操作；
- 哪些人只能访问哪些门店的数据。

### 1.2 目标

| 指标 | 目标 |
|---|---|
| 品牌层五类预设角色的权限清晰可执行 | Phase 1 交付 |
| 支持品牌管理员创建自定义角色 | Phase 1 交付 |
| 一个员工可持有多个角色（权限取并集） | Phase 1 交付 |
| 门店级数据权限隔离 | Phase 1 交付 |
| 会员服务菜单入口占位（功能体 Phase 2 实装） | Phase 1 占位 |

### 1.3 范围说明

本文档覆盖角色/菜单/权限的**产品定义**。实现细节（中间件代码、数据库迁移 SQL）在对应的工程文档中描述；完整会员服务体系见 Phase 2 规划。

---

## 2. 整体层级关系

### 2.1 三层架构

```
Platform（平台层）
│  管理品牌入驻和平台账号
│  界面：Platform Backoffice（admin 应用，端口 3001）
│  账号表：admin_users（已有 role 字段）
│  角色：super_admin / operator / support
│
└── Brand（品牌/租户层）
      │  管理本品牌门店、员工、课程、学员、预约、报表
      │  界面：Brand Backoffice（brand 应用，端口 3002）
      │  账号表：brand_users
      │  角色：通过 brand_user_roles 多对多关联，支持预设 + 自定义
      │
      ├── Store（门店）→ Room（场地）
      │
      ├── BrandStaff（品牌员工）
      │     可持有多个角色，可被分配到多个门店
      │     最终权限 = 所有持有角色的权限并集
      │
      └── Learner（学员层，属于本品牌）
            界面：Learner App（app 应用，端口 3003）
            账号表：app_users
            无角色区分，vip_level 标识会员等级（Phase 2 扩展）
```

### 2.2 层级规则

| 规则 | 说明 |
|---|---|
| 平台不直接管理学员 | Platform Administrator 只能看到品牌层面的数据（品牌信息、品牌状态），不接触学员数据 |
| 品牌数据隔离 | Brand 之间数据完全隔离，所有查询通过 brand_id 过滤 |
| 门店数据隔离（品牌内） | store_manager / coach / receptionist 只能看到被分配门店的数据；brand_admin / operations 可看全品牌数据 |
| 学员归属单一品牌 | MVP 阶段每个 Learner 只属于一个 Brand，不能跨品牌访问 |
| 平台账号独立 | admin_users 与 brand_users 是完全独立的账号体系，JWT payload 中的 user_type 区分 |
| 多角色叠加 | 员工可持有多个角色，最终权限取所有角色权限码的并集 |
| 数据权限取并集 | 若员工任一角色为 brand_admin 或 operations，则视为全品牌数据权限 |

---

## 3. 平台层角色设计（Platform Backoffice）

### 3.1 角色定义

| 角色标识 | 角色名称 | 职责描述 |
|---|---|---|
| super_admin | 超级管理员 | 平台最高权限，可执行所有操作，含创建其他 Platform Administrator 账号、删除品牌。通常只有 1–2 个账号。 |
| operator | 平台运营 | 日常品牌运营人员，可处理品牌入驻申请、编辑品牌信息、查看品牌数据，不能创建/删除 Platform Administrator 账号。 |
| support | 客服支持 | 只读型角色，用于排查问题，只能查看品牌列表和详情，不能执行写操作。 |

### 3.2 Platform Backoffice 菜单

| 菜单模块 | 说明 |
|---|---|
| 仪表盘（Dashboard） | 平台级统计：品牌总数、活跃品牌数、本月新入驻数 |
| 品牌管理（Brands） | 品牌列表、详情、审批入驻、暂停/恢复 |
| 平台账号管理（AdminUsers） | Platform Administrator 账号增删改查 |
| 平台设置（Settings） | 平台级配置（邮件模板、全局参数等） |

### 3.3 平台层操作权限矩阵

| 操作 | super_admin | operator | support |
|---|---|---|---|
| 查看仪表盘 | ✓ | ✓ | ✓ |
| 查看品牌列表/详情 | ✓ | ✓ | 只读 |
| 创建品牌 | ✓ | ✓ | — |
| 编辑品牌信息 | ✓ | ✓ | — |
| 审批入驻（pending → active） | ✓ | ✓ | — |
| 暂停品牌（active → inactive） | ✓ | ✓ | — |
| 恢复品牌（inactive → active） | ✓ | ✓ | — |
| 删除品牌 | ✓ | — | — |
| 查看平台账号列表 | ✓ | — | — |
| 创建/修改/禁用平台账号 | ✓ | — | — |
| 查看/修改平台设置 | ✓ | 只读 | — |

**权限说明：**
- ✓ = 可访问页面并执行对应操作
- 只读 = 可访问页面，操作按钮隐藏或禁用
- — = 菜单隐藏，API 调用返回 403

### 3.4 实现方式

`admin_users.role` 字段已有 `super_admin / operator / support` 枚举，Phase 1 **无需修改表结构**，只需在 `api-admin` 中间件层实现 `RequireAdminRole(roles ...string)` 守卫函数。

---

## 4. 品牌层角色体系（Brand Backoffice）

### 4.1 系统预设角色（5 个）

系统预设角色在品牌初始化时自动创建，`is_system = true`，不可删除，权限码固定。

| 角色标识 | 角色名称 | 数据权限 | 职责描述 |
|---|---|---|---|
| brand_admin | 品牌管理员 | 全品牌 | 品牌内最高权限，管理所有门店、员工、课程、学员、报表；可配置角色和系统设置 |
| store_manager | 门店店长 | 指定门店 | 管理被分配门店的日常运营：排课、本店学员、本店报表；不可管理员工账号 |
| coach | 教练 | 指定门店 + 仅自己课次 | 查看自己的课次列表，确认签到，标记爽约；不可查看其他教练课次或修改排课 |
| receptionist | 前台 | 指定门店 | 处理到店签到、代客预约/取消、学员基础信息查询；不可修改课程/排课配置，不可查看财务报表 |
| operations | 运营 | 全品牌（只读） | 全品牌只读报表查询，用于运营分析；不可修改任何配置或数据 |

### 4.2 自定义角色

- **谁可以创建**：brand_admin
- **入口**：Brand Backoffice → 系统设置 → 角色管理
- **创建方式**：从权限码列表中勾选权限，或以系统预设角色为模板复制后修改
- **存储**：存入 `brand_roles` 表，`is_system = false`
- **删除限制**：有员工持有的角色不可直接删除，需先解除关联

### 4.3 多角色叠加规则

- 一个员工通过 `brand_user_roles` 表持有多个角色
- 最终菜单权限 = 所有持有角色权限码的并集
- 最终数据权限 = 所有持有角色对应门店的并集；若任一角色为 brand_admin 或 operations，则数据权限升级为全品牌

---

## 5. 权限码体系

权限码格式：`{resource}:{action}`

后端中间件以权限码为最小检查单位，`RequirePermission("session:write")` 表示当前用户权限码集合中必须包含该码。

### 5.1 权限码定义

| 权限码 | 说明 |
|---|---|
| `dashboard:read` | 查看仪表盘 |
| `store:read` | 查看门店信息 |
| `store:write` | 创建/编辑/停用门店 |
| `room:read` | 查看场地信息 |
| `room:write` | 创建/编辑/停用场地 |
| `staff:read` | 查看员工列表和详情 |
| `staff:write` | 创建员工账号、修改角色、分配门店、禁用账号 |
| `role:read` | 查看角色列表和权限配置 |
| `role:write` | 创建/编辑/删除自定义角色 |
| `course:read` | 查看课程模板 |
| `course:write` | 创建/编辑/发布/下架课程模板；设置课程的门店可用性（CourseStoreAvailability） |
| `course:delete` | 删除课程模板 |
| `session:read` | 查看全部课次（受数据权限过滤） |
| `session:own` | 仅查看自己作为教练的课次（Coach 专用） |
| `session:write` | 排课（单次/循环）、编辑课次 |
| `session:cancel` | 取消课次 |
| `booking:read` | 查看预约记录 |
| `booking:write` | 代客预约、代客取消预约 |
| `checkin:write` | 确认签到、标记爽约 |
| `learner:read` | 查看学员列表和详情 |
| `learner:write` | 编辑学员基础信息、添加/删除标签 |
| `learner:freeze` | 冻结/解冻学员账号 |
| `membership:read` | 查看会员服务信息（Phase 2 实装） |
| `membership:write` | 管理会员卡产品和学员持卡（Phase 2 实装） |
| `report:read` | 查看报表分析 |
| `settings:write` | 修改系统设置（预约规则、品牌信息、通知模板） |

### 5.2 系统预设角色权限码分配

| 权限码 | brand_admin | store_manager | coach | receptionist | operations |
|---|---|---|---|---|---|
| dashboard:read | ✓ | ✓ | ✓ | ✓ | ✓ |
| store:read | ✓ | ✓ | — | — | ✓ |
| store:write | ✓ | — | — | — | — |
| room:read | ✓ | ✓ | — | — | ✓ |
| room:write | ✓ | ✓ | — | — | — |
| staff:read | ✓ | ✓ | — | — | — |
| staff:write | ✓ | — | — | — | — |
| role:read | ✓ | — | — | — | — |
| role:write | ✓ | — | — | — | — |
| course:read | ✓ | ✓ | ✓ | — | ✓ |
| course:write | ✓ | ✓ | — | — | — |
| course:delete | ✓ | — | — | — | — |
| session:read | ✓ | ✓ | — | ✓ | ✓ |
| session:own | — | — | ✓ | — | — |
| session:write | ✓ | ✓ | — | — | — |
| session:cancel | ✓ | ✓ | — | — | — |
| booking:read | ✓ | ✓ | ✓ | ✓ | ✓ |
| booking:write | ✓ | ✓ | — | ✓ | — |
| checkin:write | ✓ | ✓ | ✓ | ✓ | — |
| learner:read | ✓ | ✓ | — | ✓ | ✓ |
| learner:write | ✓ | ✓ | — | — | — |
| learner:freeze | ✓ | — | — | — | — |
| learner_tag:write | ✓ | ✓ | — | — | — |
| membership:read | ✓ | ✓ | — | ✓ | ✓ |
| membership:write | ✓ | — | — | — | — |
| report:read | ✓ | ✓ | — | — | ✓ |
| settings:write | ✓ | — | — | — | — |

---

## 6. Brand Backoffice 菜单与权限矩阵

### 6.1 菜单模块列表

| 菜单 | 路由 | 显示所需权限码 |
|---|---|---|
| 仪表盘 | /dashboard | dashboard:read |
| 门店管理 | /stores | store:read |
| 场地管理 | /stores/:id/rooms | room:read |
| 员工管理 | /staff | staff:read |
| 角色管理 | /roles | role:read |
| 课程管理 | /courses | course:read |
| 排课管理 | /sessions | session:read 或 session:own（二选一即可见） |
| 预约管理 | /bookings | booking:read |
| 签到管理 | /checkins | checkin:write |
| 学员管理 | /learners | learner:read |
| 会员服务 | /memberships | membership:read（菜单占位，Phase 2 实装） |
| 报表分析 | /reports | report:read |
| 系统设置 | /settings | settings:write |

### 6.2 各模块操作权限

#### 门店管理

| 操作 | 所需权限码 | 备注 |
|---|---|---|
| 查看门店列表/详情 | store:read | 受数据权限过滤 |
| 创建门店 | store:write | |
| 编辑门店信息 | store:write | |
| 停用/启用门店 | store:write | |

#### 场地管理

| 操作 | 所需权限码 | 备注 |
|---|---|---|
| 查看场地列表/详情 | room:read | 受数据权限过滤 |
| 创建场地 | room:write | |
| 编辑场地信息/容量 | room:write | |
| 停用场地 | room:write | |

#### 员工管理

| 操作 | 所需权限码 | 备注 |
|---|---|---|
| 查看员工列表/详情 | staff:read | store_manager 只可见本店员工 |
| 创建员工账号 | staff:write | |
| 分配/修改角色（多角色） | staff:write | |
| 分配员工门店 | staff:write | |
| 禁用员工账号 | staff:write | |
| 重置员工密码 | staff:write | |

#### 角色管理

| 操作 | 所需权限码 | 备注 |
|---|---|---|
| 查看角色列表和权限配置 | role:read | 含系统预设角色（只读）和自定义角色 |
| 创建自定义角色 | role:write | 从权限码列表勾选权限 |
| 编辑自定义角色权限 | role:write | 系统角色不可编辑 |
| 复制角色为模板 | role:write | |
| 删除自定义角色 | role:write | 有员工持有时不可删除 |

#### 课程管理

| 操作 | 所需权限码 |
|---|---|
| 查看课程列表/详情 | course:read |
| 创建课程模板 | course:write |
| 编辑课程信息 | course:write |
| 发布/下架课程 | course:write |
| 设置课程可用门店（CourseStoreAvailability） | course:write | 仅 brand_admin；store_manager 排课时只能看到本门店已启用的课程 |
| 删除课程模板 | course:delete |

#### 排课管理

| 操作 | 所需权限码 | 备注 |
|---|---|---|
| 查看课次列表 | session:read 或 session:own | session:own 只能看自己的课次 |
| 单次排课 | session:write | |
| 循环排课 | session:write | |
| 编辑课次 | session:write | |
| 取消课次 | session:cancel | |

#### 预约管理

| 操作 | 所需权限码 | 备注 |
|---|---|---|
| 查看预约列表 | booking:read | coach 只能看自己课次的预约 |
| 代客预约 | booking:write | |
| 代客取消预约 | booking:write | |

#### 签到管理

| 操作 | 所需权限码 | 备注 |
|---|---|---|
| 查看签到列表 | checkin:write | 包含查看能力 |
| 确认学员到课 | checkin:write | coach 只能操作自己课次 |
| 标记爽约 | checkin:write | coach 只能操作自己课次 |

#### 学员管理

| 操作 | 所需权限码 | 备注 |
|---|---|---|
| 查看学员列表/详情 | learner:read | 受数据权限过滤 |
| 查看学员预约历史 | learner:read + booking:read | |
| 查看学员训练记录 | learner:read | |
| 编辑学员基础信息/备注 | learner:write | |
| 添加/删除学员标签 | learner_tag:write | |
| 冻结学员账号 | learner:freeze | 仅 brand_admin |
| 解冻学员账号 | learner:freeze | 仅 brand_admin |

**冻结规则：**
- 冻结前需检查该学员是否有未来 24 小时内状态为 confirmed 的 Booking，若有则弹出提示询问是否同时取消预约
- 冻结本身不自动取消预约，由操作人员决定

#### 报表分析

| 操作 | 所需权限码 | 备注 |
|---|---|---|
| 预约/上座率报表 | report:read | 受数据权限过滤 |
| 收入流水报表 | report:read | 受数据权限过滤 |
| 学员活跃度报表 | report:read | 受数据权限过滤 |
| 教练课时报表 | report:read | 受数据权限过滤 |

#### 系统设置

| 操作 | 所需权限码 |
|---|---|
| 预约规则配置 | settings:write |
| 品牌基础信息修改 | settings:write |
| 通知模板设置 | settings:write |

---

## 7. 数据权限设计

### 7.1 数据权限类型

| 角色 | 门店数据范围 | 课次数据范围 |
|---|---|---|
| brand_admin | 全品牌（不过滤 store_id） | 全品牌 |
| store_manager | 仅分配的门店 | 仅分配门店的课次 |
| coach | 仅分配的门店 | 仅分配门店中自己（coach_id = self）的课次 |
| receptionist | 仅分配的门店 | 仅分配门店的课次（只读） |
| operations | 全品牌（不过滤 store_id） | 全品牌 |

### 7.2 多角色叠加时数据权限

- 取所有持有角色对应门店集合的并集
- 若任一角色为 brand_admin 或 operations，数据权限升级为全品牌（`store_ids` 置空表示不过滤）

### 7.3 实现伪代码

```
func getAccessibleStoreIDs(user BrandUser) ([]int64, globalAccess bool) {
    roles := user.Roles  // 从 JWT 取
    for _, role := range roles {
        if role == "brand_admin" || role == "operations" {
            return nil, true  // 全品牌，不过滤
        }
    }
    return user.StoreIDs, false  // 从 JWT 取
}

func buildSessionFilter(user BrandUser) QueryFilter {
    storeIDs, global := getAccessibleStoreIDs(user)
    filter := QueryFilter{Global: global, StoreIDs: storeIDs}
    if hasPermission(user, "session:own") && !hasPermission(user, "session:read") {
        filter.CoachID = user.ID  // 只有 session:own 时额外限制
    }
    return filter
}
```

---

## 8. 数据库模型设计

### 8.1 新增表：`brand_roles`（品牌角色表）

```sql
CREATE TABLE brand_roles (
    id          bigserial PRIMARY KEY,
    brand_id    bigint NOT NULL REFERENCES brands(id),
    name        varchar(50) NOT NULL,          -- 角色显示名称
    code        varchar(30) NOT NULL,          -- 角色标识（系统预设固定值，自定义随机生成）
    is_system   boolean NOT NULL DEFAULT false, -- 是否系统预设角色
    description text,
    status      varchar(20) NOT NULL DEFAULT 'active',
    created_at  timestamptz NOT NULL DEFAULT now(),
    updated_at  timestamptz NOT NULL DEFAULT now(),

    UNIQUE (brand_id, code)
);

CREATE INDEX idx_brand_roles_brand_id ON brand_roles(brand_id);
```

品牌初始化时自动插入 5 条 is_system=true 记录：
`brand_admin / store_manager / coach / receptionist / operations`

### 8.2 新增表：`brand_role_permissions`（角色权限码关联表）

```sql
CREATE TABLE brand_role_permissions (
    id              bigserial PRIMARY KEY,
    role_id         bigint NOT NULL REFERENCES brand_roles(id) ON DELETE CASCADE,
    permission_code varchar(50) NOT NULL,
    brand_id        bigint NOT NULL REFERENCES brands(id),  -- 冗余，便于按品牌查询

    UNIQUE (role_id, permission_code)
);

CREATE INDEX idx_brand_role_perms_role_id ON brand_role_permissions(role_id);
CREATE INDEX idx_brand_role_perms_brand_id ON brand_role_permissions(brand_id);
```

### 8.3 新增表：`brand_user_roles`（员工角色多对多）

```sql
CREATE TABLE brand_user_roles (
    id              bigserial PRIMARY KEY,
    brand_user_id   bigint NOT NULL REFERENCES brand_users(id) ON DELETE CASCADE,
    role_id         bigint NOT NULL REFERENCES brand_roles(id) ON DELETE CASCADE,
    brand_id        bigint NOT NULL REFERENCES brands(id),  -- 冗余
    created_at      timestamptz NOT NULL DEFAULT now(),

    UNIQUE (brand_user_id, role_id)
);

CREATE INDEX idx_brand_user_roles_user_id ON brand_user_roles(brand_user_id);
CREATE INDEX idx_brand_user_roles_role_id ON brand_user_roles(role_id);
CREATE INDEX idx_brand_user_roles_brand_id ON brand_user_roles(brand_id);
```

### 8.4 新增表：`brand_user_stores`（员工门店权限关联）

```sql
CREATE TABLE brand_user_stores (
    id              bigserial PRIMARY KEY,
    brand_user_id   bigint NOT NULL REFERENCES brand_users(id) ON DELETE CASCADE,
    store_id        bigint NOT NULL REFERENCES stores(id) ON DELETE CASCADE,
    brand_id        bigint NOT NULL REFERENCES brands(id),  -- 冗余
    created_at      timestamptz NOT NULL DEFAULT now(),

    UNIQUE (brand_user_id, store_id)
);

CREATE INDEX idx_brand_user_stores_user_id ON brand_user_stores(brand_user_id);
CREATE INDEX idx_brand_user_stores_store_id ON brand_user_stores(store_id);
CREATE INDEX idx_brand_user_stores_brand_id ON brand_user_stores(brand_id);
```

说明：
- brand_admin / operations 全品牌权限，无需在此表插入记录（无记录 = 全品牌）
- store_manager / coach / receptionist 需要在此表关联 1 个或多个门店

### 8.5 新增表：`learner_tags`（学员标签）

```sql
CREATE TABLE learner_tags (
    id          bigserial PRIMARY KEY,
    brand_id    bigint NOT NULL REFERENCES brands(id),
    learner_id  bigint NOT NULL REFERENCES app_users(id),
    tag_name    varchar(50) NOT NULL,
    created_by  bigint NOT NULL REFERENCES brand_users(id),
    created_at  timestamptz NOT NULL DEFAULT now(),

    UNIQUE (brand_id, learner_id, tag_name)
);

CREATE INDEX idx_learner_tags_learner_id ON learner_tags(learner_id);
CREATE INDEX idx_learner_tags_brand_id ON learner_tags(brand_id);
```

### 8.6 `brand_users` 表变更

移除单 role 字段方案（改为多角色中间表），新增：

| 字段 | 类型 | 说明 |
|---|---|---|
| is_owner | boolean | 是否品牌创始人/最高管理员，不可被禁用或降权，默认 false |

无需在 `brand_users` 上保留 role 字段。

### 8.7 `admin_users` 表（无需变更）

`admin_users.role` 字段已有 `super_admin / operator / support` 枚举，Phase 1 不修改表结构。

### 8.8 迁移文件清单（Phase 1）

| 迁移文件 | 内容 |
|---|---|
| `000XXX_add_is_owner_to_brand_users.up.sql` | brand_users 新增 is_owner 字段 |
| `000XXX_create_brand_roles.up.sql` | 创建 brand_roles 表及索引 |
| `000XXX_create_brand_role_permissions.up.sql` | 创建 brand_role_permissions 表及索引 |
| `000XXX_create_brand_user_roles.up.sql` | 创建 brand_user_roles 表及索引 |
| `000XXX_create_brand_user_stores.up.sql` | 创建 brand_user_stores 表及索引 |
| `000XXX_create_learner_tags.up.sql` | 创建 learner_tags 表及索引 |
| `000XXX_seed_system_roles.up.sql` | 为已有品牌插入 5 个系统预设角色及权限码记录 |

---

## 9. 学员层功能入口（Learner App）

### 9.1 Learner 无角色区分

Learner 是单一身份，无角色差异。`app_users.vip_level` 字段用于 Phase 2 会员等级特权，Phase 1 所有 Learner 权限一致。

### 9.2 Learner App 功能 Tab

| Tab / 页面 | 路由 | 功能说明 |
|---|---|---|
| 首页（Home） | /home | 品牌公告、推荐课次、快捷入口 |
| 课程（Sessions） | /sessions | 可预约课次列表，支持按日期/类型筛选 |
| 课次详情 | /sessions/:id | 课次详情、预约/加入候补按钮 |
| 我的预约 | /bookings | 即将上课 / 历史记录 / 已取消 三个 Tab |
| 候补列表 | /waitlists | 当前候补中的记录及状态 |
| 个人中心（Profile） | /profile | 个人信息、头像、联系方式 |
| 训练记录 | /training | 历史训练记录列表 |

### 9.3 Learner 数据可见范围

| 数据类型 | 可见范围 | 说明 |
|---|---|---|
| 课次（Session） | 仅本品牌 + 状态为 scheduled + 在预约窗口内 | 已结束或已取消课次不展示 |
| 自己的预约（Booking） | 仅自己（learner_id = self） | 不能看其他学员的预约 |
| 自己的候补（Waitlist） | 仅自己 | 候补队列位置可见 |
| 学员档案 | 仅自己 | 不能查看其他学员信息 |
| 训练记录 | 仅自己 | |
| 课次剩余名额 | 可见（聚合数字，非名单） | 仅展示剩余名额数，不暴露已预约学员名单 |

### 9.4 Learner 端 API 权限规则

- 所有 `api-app` 接口验证 JWT 中 `brand_id` 与请求资源归属一致
- `learner_id` 从 JWT 取，不允许客户端传递 learner_id 参数查询他人数据

---

## 10. JWT Payload 设计

```json
// api-admin 颁发（Platform Administrator）
{
  "user_type": "admin",
  "user_id": 1,
  "role": "super_admin",
  "exp": 1234567890
}

// api-brand 颁发（BrandStaff）
// permissions 在登录时计算并缓存在 JWT，避免每次请求查库
{
  "user_type": "brand",
  "user_id": 42,
  "brand_id": 7,
  "is_owner": false,
  "permissions": ["session:own", "checkin:write", "booking:read", "dashboard:read"],
  "store_ids": [3, 5],
  "exp": 1234567890
}
// store_ids 为空数组表示全品牌（brand_admin / operations 的情况）

// api-app 颁发（Learner）
{
  "user_type": "app",
  "user_id": 101,
  "brand_id": 7,
  "exp": 1234567890
}
```

**安全规则：**
- `brand_id` 和 `permissions` 均从 JWT 取，不信任请求参数
- `store_ids` 从 JWT 取，注入所有数据查询的过滤条件
- 修改员工角色或门店分配后，需强制使旧 JWT 失效（黑名单或缩短 TTL）

---

## 11. 分期交付计划

### Phase 1（本文档范围，必须做）

| 序号 | 任务 | 说明 |
|---|---|---|
| 1 | 数据库迁移 | 创建 brand_roles / brand_role_permissions / brand_user_roles / brand_user_stores / learner_tags 五张表；brand_users 新增 is_owner 字段 |
| 2 | 品牌初始化种子 | 每个品牌自动创建 5 个系统预设角色及权限码记录 |
| 3 | api-admin 权限守卫 | RequireAdminRole() 中间件，从 JWT role 字段拦截未授权 API 调用 |
| 4 | api-brand 权限中间件 | RequirePermission() 守卫，从 JWT permissions 集合检查权限码 |
| 5 | api-brand 数据权限注入 | 从 JWT store_ids 注入所有 Store/Session/Booking/Learner 查询的过滤条件 |
| 6 | Brand Backoffice 动态菜单 | 根据当前用户 permissions 动态渲染侧边栏，隐藏无权限菜单项 |
| 7 | Brand Backoffice 操作按钮权限 | 页面内按钮/操作项根据 permissions 显示/隐藏（前端防御层） |
| 8 | 员工管理页（BrandStaff CRUD） | brand_admin 创建/编辑员工，支持多角色分配和门店分配 |
| 9 | 角色管理页（Role CRUD） | brand_admin 查看预设角色（只读）；创建/编辑/删除自定义角色（权限码勾选 UI） |
| 10 | 学员管理：冻结/解冻 | brand_admin 专属，含预约冲突提示 |
| 11 | 学员标签功能 | brand_admin / store_manager 可操作 |

**Phase 1 不包含：**
- 完整会员卡体系（仅菜单占位）
- PostgreSQL RLS policy（继续用应用层 brand_id 过滤）
- 权限审计日志

### Phase 2（增强）

| 序号 | 任务 | 说明 |
|---|---|---|
| 1 | 会员卡产品体系 | membership_products、learner_memberships、membership_transactions 三张表及 API |
| 2 | 次卡/时间卡消耗逻辑 | 预约时扣减，爽约扣减，取消退还 |
| 3 | 学员会员卡页面（Learner App） | 我的会员卡、使用记录、到期提醒 |
| 4 | 代客预约前台完整操作流 | 含卡次校验的 Receptionist 代客预约 UI |
| 5 | 学员分组 | 批量操作、分组营销推送 |
| 6 | 权限审计日志 | 记录敏感操作：冻结学员、修改角色、删除课次等 |
| 7 | JWT 黑名单机制 | 角色变更后立即使旧 Token 失效（当前 Phase 1 依赖 TTL 自然过期） |

### Phase 3（商业化）

| 序号 | 任务 | 说明 |
|---|---|---|
| 1 | 在线购卡（微信/抖音小程序支付） | 学员 App 内购买会员卡 |
| 2 | PostgreSQL RLS policy | 数据库层隔离，提升安全性 |
| 3 | 多品牌 Learner | 修改 app_users 结构，支持一个手机号绑定多个品牌 |

---

## 12. 已确认决策

| 编号 | 问题 | 决策 |
|---|---|---|
| D-01 | 是否支持自定义角色 | 是，brand_admin 可在角色管理页创建自定义角色并勾选权限码 |
| D-02 | 是否支持多角色叠加 | 是，一人可持有多个角色，权限取并集，数据权限取并集 |
| D-03 | 会员服务本期范围 | 仅菜单入口占位，功能体放 Phase 2 详细规划 |

---

## 附录 A：枚举值汇总

| 表 | 字段 | 允许值 |
|---|---|---|
| admin_users | role | super_admin / operator / support |
| brand_users | status | active / inactive |
| brand_roles | status | active / inactive |
| brand_roles | code（系统预设） | brand_admin / store_manager / coach / receptionist / operations |
| app_users | status | active / inactive |
| app_users | vip_level | 0（普通）/ 1（VIP，Phase 2 扩展） |

## 附录 B：会员服务 Phase 2 数据模型（详见 MEMBERSHIP.md）

会员卡模块完整 PDS（含数据模型、业务规则、API 端点、状态流转）已迁移至独立文档 **`MEMBERSHIP.md`**，本附录不再维护细节，以主文档为准。

**核心表概览（5张新表）：**

| 表名 | 说明 |
|---|---|
| `membership_products` | 会员卡产品模板；含 `store_scope`（brand / store_specific）和 `course_scope`（all / course_specific）两个范围字段 |
| `membership_product_stores` | 产品可用门店关联表（store_scope=store_specific 时生效） |
| `membership_product_courses` | 产品可用课程关联表（course_scope=course_specific 时生效） |
| `learner_memberships` | 学员持有的卡实例；含 remaining_credits、expires_at、status |
| `membership_transactions` | 消费/退还/爽约扣减等全量流水 |

**关键设计决策：**
- 不在卡层级绑定教练（coach），通过 course_specific 绑定课程模板实现「教练专属课包」
- Phase 2 起预约必须持有匹配卡，无卡不可约
- 购卡 Phase 2 为后台手动开卡（线下收款），在线支付为 Phase 3
