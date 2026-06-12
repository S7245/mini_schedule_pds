# Batch 8 契约 — 品牌门店管理页 `/locations` + location.view 门 + Playwright 回归

> 状态：**契约已 approve**（2026-06-12，会话内确认）。
> 背景：Batch 7 验收暴露 `location.view` 前端无可见门、`/locations` 管理页未建（Batch 4 FR 4.2 遗留）。本批补齐。

---

## 0. 关键事实（survey 已确认）

**后端已完整，无需改动**：`location_handler.go` 已有 6 个路由（list/get/create/update/status/delete），含订阅额度 guard、audit、软删、data_scope（assigned_locations）过滤、`(brand_id,name)` 唯一约束。permission gate：list/get=`location.view`、create=`location.create`、update=`location.edit`、status=`location.edit`、delete=`location.delete`。

**前端已有可复用件**：`packages/api/locations.ts`（6 hooks 全有）、`packages/types` Location 类型全有、`components/locations/location-form-dialog.tsx`（create/edit + 错误映射 LOCATION_NAME_DUPLICATED/QUOTA_EXCEEDED/LOCATION_NOT_FOUND）、`components/locations/location-status-toggle.tsx`（gate location.edit）、`lib/permissions.tsx` 已有 `LOCATION_VIEW/CREATE/EDIT/DELETE`。onboarding 内已有 `/onboarding/locations` 页用着这些件。

**缺的就是**：独立 `/locations` 管理页 + 导航入口（location.view 门）+ Playwright 回归脚本。

---

## 0.1 grill 决策（已与用户确认 2026-06-12）

| 决策 | 定论 |
|---|---|
| Playwright 模式 | **真实跑通整个栈**：真登录(owner 18816820405)+真 CRUD 调 :8081+真 DB，脚本自带 setup/teardown（唯一名创建、跑完软删清理） |
| 删除门店引用保护 | **本批不动，转 FR**（场次表未落地，引用保护现在加不完整；软删现状保留） |
| UpdateStatus 权限门 | **保持 `location.edit`**（不改后端；seed 的 location.toggle_status 暂闲置，记 FR） |

→ **本批无后端代码改动**。纯前端页面 + 导航 + e2e 脚本。

---

## 1. 契约

### API 接口
**无新增/改动**。复用现有：

| 方法 | 路径 | gate | 用途 |
|---|---|---|---|
| GET | `/api/v1/brand/locations?page&page_size&status` | location.view | 列表（分页+状态筛选，data_scope 自动过滤） |
| GET | `/api/v1/brand/locations/:id` | location.view | 详情 |
| POST | `/api/v1/brand/locations` | location.create | 新建（额度 guard） |
| PATCH | `/api/v1/brand/locations/:id` | location.edit | 编辑 |
| PATCH | `/api/v1/brand/locations/:id/status` | location.edit | 启用/停用 |
| DELETE | `/api/v1/brand/locations/:id` | location.delete | 软删 |

### 前端页面模块

| 页面/模块 | 类型 | 关键字段/操作 |
|---|---|---|
| `/locations` 门店管理页 | 页面（新建） | 表格列：name / address / phone / status badge；顶部「新建门店」按钮 gate `location.create`（disabled+Hint）；状态筛选 Select（全部/启用/停用）+ 分页；空状态。行操作：编辑(gate location.edit)/启用·停用(`LocationStatusToggle`, location.edit)/删除(gate location.delete, `ConfirmDialog`)。 |
| 工作台导航 | 导航 | `config/nav.tsx` 加「门店管理」(lucide `Store` 图标，置于「员工管理」附近)；`layout.tsx` `NAV_HREF_PERMISSIONS['/locations'] = PERMISSIONS.LOCATION_VIEW` |

### 前端实现约束
- **复用**：`LocationFormDialog`（create/edit）、`LocationStatusToggle`、`ConfirmDialog`、`useBrandLocations`/`useCreateBrandLocation`/`useUpdateBrandLocation`/`useUpdateBrandLocationStatus`/`useDeleteBrandLocation`、`usePermissions`/`Hint`/`PERMISSIONS.LOCATION_*`。**不新增 UI 库、不新写 API hook/类型**。
- **镜像** `/staff` 页结构（header+筛选+分页+空状态）+ `/roles` 页操作按钮/确认弹窗模式 + `/onboarding/locations` 页的列与 badge 样式。
- 删除错误映射：`LOCATION_NOT_FOUND` → toast「门店不存在或已删除」+ 关弹窗；其余沿用 `LocationFormDialog` 内既有映射。
- data_scope：列表/详情后端已按 assigned_locations 过滤，前端无需特殊处理（location_manager 只见任职门店）。
- 不引入服务端 name 搜索（后端 list 不支持 q）；如需可加**当前页客户端名称过滤**，非必须。
- 守 Batch 6 跨用户缓存泄漏铁律：不新增 user-scoped query（locations query 是 brand-scoped）。

### Playwright 回归脚本（真实栈）

`web/e2e/batch-08-locations.spec.ts`，沿用 `e2e/batch-01-signup.spec.ts` 约定（describe + H/E 前缀，baseURL localhost:3002）。

- **前置**：需后端 :8081 + brand 前端 :3002 在跑、DB 可用。新增登录 helper（owner `18816820405` / `admin123`，写 auth-storage 或走登录表单）。
- **覆盖**：
  - H1 owner 登录后导航见「门店管理」入口，进 `/locations` 列表渲染
  - H2 新建门店（name=`e2e-loc-<时间戳>` 保证唯一）→ 列表出现该行、status=启用
  - H3 编辑该门店地址/电话 → 列表反映更新
  - H4 启用/停用切换 → badge 变化（经 ConfirmDialog）
  - H5 软删该门店 → 行从列表消失
  - E1 重名新建 → toast `LOCATION_NAME_DUPLICATED`「门店名称已存在」
  - E2（可选，若易实现）无 location.create 的账号「新建门店」按钮 disabled + Hint
- **teardown**：脚本创建的门店跑完全部软删（按唯一名前缀清理），避免污染 + 占额度。
- **交付**：脚本 + 顶部注释写清运行前置（起后端/前端、测试账号、`pnpm exec playwright test e2e/batch-08-locations.spec.ts`），让另一 session 可独立执行。

---

## 2. 验收闭环（手动 + 脚本）

owner 登录 → 工作台见「门店管理」→ 进 `/locations` → 新建/编辑/停用/软删跑通 → 重名报错 → location_manager（只 location.view）登录：见入口、能看列表、但「新建/编辑/删除」按 disabled+Hint（无 create/edit/delete）→ 跑 `batch-08-locations.spec.ts` 全绿。

---

## 3. 任务拆分（TDD 逐 task commit）

**前端**（仅 `apps/brand` + 复用 packages）
- T01 `/locations` 页：列表表格 + 状态筛选 + 分页 + 空状态 + 「新建门店」gate
- T02 行操作接线：编辑(LocationFormDialog)/停用切换(LocationStatusToggle)/删除(ConfirmDialog + 错误映射)
- T03 导航入口 + `NAV_HREF_PERMISSIONS['/locations']=location.view`

**e2e**
- T04 登录 helper + `batch-08-locations.spec.ts`（H1–H5 + E1，真实栈，含 teardown）

**FR（不在本批做，记录）**
- 软删门店的引用保护（LOCATION_IN_USE）；UpdateStatus 改 location.toggle_status；后端 list 加 name 搜索。
