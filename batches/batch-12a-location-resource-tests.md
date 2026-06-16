# Batch 12a 测试场景 — Location Resource 资源管理

测试品牌：brand 21｜owner `18816820405/admin123`｜只读 `13900139777/admin123`｜门店「讯美广场」id=1｜教练「张三」instructor_profile id=1。
后端端口 8081（api-brand）。铁律：测前重建/重启后端（旧二进制掩盖改动）+ 前端 `rm -rf .next` 重启 + curl 预热；前端 filter 用包名 `@mini-schedule/brand`。

## Happy Path

| # | 步骤 | 预期结果 |
|---|---|---|
| H1 | owner 登录 → /resources，门店选「讯美广场」→ 新增资源「1 号教室」(type=classroom, capacity=10, remark) | 列表出现该资源，状态「启用」，容量 10 |
| H2 | /resources 编辑「1 号教室」容量→12、备注改写 | 保存成功，列表容量 12 |
| H3 | /schedule 排课：门店讯美广场 → 已发布课程 → 教练张三 → 资源选「1 号教室」 | 容量输入自动填 12（资源容量），可改 |
| H4 | 提交 H3（未来时间、时长 60） | 场次 A 创建成功（scheduled），/schedule 表「资源」列显示「1 号教室」 |
| H5 | /schedule 再排场次 B：同门店/课程/资源、与 A 不重叠时段、同教练张三 | 创建成功（资源+教练都不撞） |
| H6 | /resources 停用「1 号教室」（active→inactive） | 列表状态「停用」 |
| H7 | 取消场次 A、B 后，/resources 删除「1 号教室」 | 204，列表移除（软删） |

## Edge Cases

| # | 场景 | 预期结果 |
|---|---|---|
| E1 | /resources 在讯美广场再建同名「1 号教室」（未软删存在） | RESOURCE_NAME_DUPLICATED → toast「该门店已有同名资源」，弹窗保持打开 |
| E2 | 资源 type 传非法值（直连 API） | INVALID_PARAM 400 |
| E3 | /schedule 排课绑**已停用**资源（先停用再排，或下拉应不含 inactive） | 下拉不出现该资源；直连 API 绑 inactive → RESOURCE_NOT_AVAILABLE 409 |
| E4 | /schedule 排场次 B'：同资源、与 A **时段重叠**、**换教练**（避开教练冲突） | SESSION_RESOURCE_CONFLICT → toast「该资源在此时段已被占用」，场次不创建 |
| E5 | 删除被 scheduled 场次 A 引用的资源（A 未取消） | RESOURCE_IN_USE 409 → toast「该资源仍被未结束场次或循环排课占用」，弹窗保持打开 |
| E6 | 绑跨门店资源排课（资源属门店 X，场次门店 Y，直连 API） | RESOURCE_NOT_AVAILABLE 409 |
| E7 | 只读 13900139777 登录 /resources | 新增/编辑/停用/删除按钮全 disabled（Hint tooltip） |
| E8 | 只读 13900139777 直连 POST /location-resources | 403 权限不足 |
| E9 | 建 active 资源后删除其所属门店「讯美广场」 | LOCATION_IN_USE 409（active 资源计入 CountActiveReferences） |
| E10 | data_scope=assigned_locations 的员工访问非授权门店资源 list/detail | 被过滤/RESOURCE_NOT_FOUND 404 |

## 后端单测覆盖（DB 集成 + service fake）

- repo：Create（重名 23505→RESOURCE_NAME_DUPLICATED、capacity 默认 1、跨租户隔离）、Update（status 切换、跨租户）、Delete（被场次引用拒删、被 active recurring 引用拒删、无引用软删成功）、List（location_id 过滤 + status 过滤 + 反范式 location_name + 软删排除 + scope IN 过滤）、GetByID（软删/越权→not found）。
- class_session repo：Create 绑资源（资源校验 active/同门店/未软删；容量优先级 input>资源>课程；资源 EXCLUDE 23P01→SESSION_RESOURCE_CONFLICT vs 教练 23P01→SESSION_INSTRUCTOR_CONFLICT 按约束名分流）；list/detail 反范式 resource_name。
- location repo：CountActiveReferences 纳入 active location_resources（含/不含场景）。
- service（fake repo）：require 权限门、scopeFilterIDs/guardLocationInScope（all_brand 放行、assigned 过滤、越权 404）。

## 执行方式

后端：`go test ./...`（含新增 repo/service 单测）。前端：`pnpm --filter @mini-schedule/brand lint && pnpm --filter @mini-schedule/brand build`。
e2e：用户另开 session 跑真实栈（UI 登录拿 token → page.request 建资源/场次 → H/E 场景 → API teardown 取消场次+软删资源）。
