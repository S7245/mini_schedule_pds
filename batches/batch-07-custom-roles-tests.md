# Batch 7 测试场景 — 品牌自定义角色 CRUD

契约见 `batch-07-custom-roles.md`（已 approve）。前后端实现按下表逐 task TDD。

---

## Happy Path

| # | 步骤 | 预期结果 | 结论 |
|---|---|---|---|
| 1 | owner 登录 brand 后台，工作台导航 | 出现「角色管理」入口（gate=`role.manage`，owner fast-path 有权）| 通过 |
| 2 | 进入 `/roles` | 列表展示 8 个系统角色（`is_system` badge）+ 各自权限数；系统角色行仅「查看 / 复制为自定义」 | 通过 |
| 3 | 点「新建角色」，填 name=「前台兼职」，scope=location，勾 `staff.view`+`location.view`，保存 | POST /roles 201，列表新增一行带「自定义」badge，code 形如 `custom_xxx` | 通过 |
| 4 | `GET /api/v1/brand/permissions` | 返回按 domain 分组的全量细粒度权限（brand/location/staff/instructor/role）| 通过 |
| 5 | `GET /api/v1/brand/roles/custom_xxx` | 返回该角色详情，permissions 含 staff.view + location.view（原始勾选，未展开）| 通过 |
| 6 | 在某员工详情页分配「前台兼职」+ 某 location，data_scope=assigned_locations | 分配成功，员工持有该角色 |  通过 |
| 7 | 该员工登录 | 看到 staff/location 对应菜单与按钮 | 通过（员工 13900139777 仅持 custom_5f0b9bc8=staff.view+location.view：登录后侧栏显示「员工管理」、可进 /staff；无「角色管理」入口。注：location.view 当前前端无独立菜单/按钮门，仅 staff.view 驱动 /staff 菜单——见报告 §UI 说明）|
| 8 | owner 编辑「前台兼职」去掉 `location.view`，保存（PUT 全量替换） | 200；**该员工刷新页面后** location 入口立即消失（验证 C1 主动失效，不等 60s）| 通过（C1 已在 API+Redis 层证实：编辑前 `/me/permissions` 含 location.view 且 Redis `rbac:perms:23` 存在；owner PUT 去掉 location.view 返回 200 后，**该 key 立即被 DEL**（reverse-lookup 主动失效），员工 60s 内重查 `/me/permissions` location.view 即消失。前端因 location.view 无菜单门，UI 上无可见入口消失，但底层主动失效成立）|
| 9 | owner 移除该员工的「前台兼职」任职后，删除该角色 | DELETE 204，列表移除该行 | 通过 |
| 10 | PATCH `/roles/custom_yyy/status` status=inactive | 200，角色变   停用态，不可再新分配 | 通过（PATCH inactive→200 且 status=inactive；尝试把该停用角色分配给员工 → 400 INVALID_PARAM「角色已停用」，不可再新分配）|

## Edge Cases

| # | 场景 | 预期结果 | 结论 |
|---|---|---|---|
| E1 | 删除仍有 active 任职引用的自定义角色 | 409 `ROLE_IN_USE`，前端 toast「该角色仍有员工任职，请先移除」| 通过（UI：owner 删 custom_5f0b9bc8 → DELETE 返回 **409 ROLE_IN_USE**，toast「该角色仍有员工任职，请先在员工详情移除任职」，确认弹窗保持打开）|
| E2 | PUT/DELETE/PATCH 作用于系统角色（is_system=TRUE）| 409 `ROLE_IS_SYSTEM`，前端系统角色行根本不渲染编辑/删除按钮 | 通过（PUT/DELETE/PATCH course_operator 全部 **409 ROLE_IS_SYSTEM**；UI：8 个系统角色行仅渲染「查看 / 复制为自定义」，无编辑/删除）|
| E3 | 删除或降级 brand_owner 系统角色 | 409 `OWNER_PROTECTED`（D2 防线）| 通过（DELETE/PATCH inactive/PUT brand_owner 全部 **409 OWNER_PROTECTED**；owner 保护优先于 is_system，符合 D2）|
| E4 | 非 owner 管理员新建角色，勾选超出自身有效权限的 permission | 403 `ROLE_PERMISSION_EXCEEDS_ACTOR`；前端超出项 disable + Hint | 通过（brand_admin 18816820408 POST 勾 staff.delete（其缺）→ **403 ROLE_PERMISSION_EXCEEDS_ACTOR**，data.exceeded=staff.delete；对照：只勾自有权限 →201）|
| E4-编辑变体 | 非 owner 编辑「含其缺失权限」的角色：只改名（保留该缺失权限）应成功；新增缺失权限才拦 | 改名成功 200；新增越权项 403（B1 增量语义）| 通过（**需后端 rebuild 后才成立**，详见报告 §关键发现 1）。brand_admin 对含 staff.delete 的角色只改名（permission_codes 不变）→ **200**；同一请求新增 subscription.manage（其缺）→ **403 EXCEEDS, exceeded=subscription.manage**。增量语义正确：仅校验本次新增项 |
| E5 | owner 新建角色勾任意权限 | 通过（owner 例外）| 通过（owner POST 勾 staff.delete+subscription.manage+location.delete → **201**，owner fast-path 跳过 B1）|
| E6 | POST /roles permission_codes=[]（空数组）| 201，创建无权限角色（不应被 omitempty 吞成 null）| 通过（POST permission_codes=[] → **201**；GET 回查角色存在且 0 权限，未被吞成 null/报错）|
| E7 | PUT /roles 试图改 scope_type | scope_type 被忽略/拒绝，角色 scope 不变（A3）| 通过（location 角色 PUT 带 scope_type=brand → 200 但 **scope_type 仍为 location**，仅 name 生效；GET 复核 scope 未变）|
| E8 | 无 `role.manage` 权限的用户访问写接口 | 403 `PERMISSION_DENIED`；前端「新建」按钮 disabled + Hint | 通过（location_manager 13900139001 POST /roles → **403 PERMISSION_DENIED**，missing=[role.manage]；该账号侧栏无「角色管理」入口、写接口被 gate 拦）|
| E9 | 起空库顺序 migrate 至 000006 | 成功；permissions 含 role.manage；brand_owner/brand_admin 模板映射存在 |
| E10 | 存量 brand（000006 之前已 seed）执行 backfill | brand_admin 角色获得 role.manage；该 brand 的 admin 登录可见角色管理入口 |

## 执行方式

- 后端：`go test ./...`（含 migration up + service/handler 单测覆盖 E1–E10 的错误码路径）。
- 前端：`pnpm --filter @mini-schedule/brand lint && build`。
- 端到端：browser-automation / senior-qa skill 按 Happy Path 1–10 生成 Playwright 脚本（回收 Batch 6 延后的 T11）。
