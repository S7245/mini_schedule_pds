# Batch 11 测试场景

账号：owner=18816820405 / admin123（brand 21）。只读账号 13900139777（Batch 10 建）用于权限门断言。

## Happy Path

| # | 步骤 | 预期结果 |
|---|---|---|
| H1 | 登录 owner，进 /course-categories，新增分类「团课」(color 任选) | 列表新增一行，status=active |
| H2 | 编辑「团课」改 sort_order / 改 show_in_mini_program | 行更新，PATCH 落库 |
| H3 | 进 /courses，新增课程模板「晨间瑜伽」(分类=团课, level_label=初级, duration_min=60, default_capacity=8, 可用门店=默认全选) | 列表新增，status=draft，可用门店数=当前 active 门店数 |
| H4 | 对「晨间瑜伽」点发布 (status draft→published) | badge 变「已发布」，published_at 非空 |
| H5 | 进 /schedule，排单场次：门店1 + 课程「晨间瑜伽」+ 教练A + 明天 09:00 + 60min | 课程表新增 scheduled 场次，capacity 默认 8 |
| H6 | 进 /onboarding | 第 4(分类)/5(模板)/7(场次) 步均 completed |
| H7 | GET /courses/:id | 返回 category_ids + available_location_ids 正确 |

## Edge Cases

| # | 场景 | 预期结果 |
|---|---|---|
| E1 | 新增分类用已存在 name | `CATEGORY_NAME_DUPLICATED` 409，前端 inline+toast |
| E2 | 课程模板 category_ids 含非本 brand 分类 id | `CATEGORY_NOT_FOUND` 404 |
| E3 | 排课选未发布(draft)课程 | `COURSE_NOT_ACTIVE` 409 |
| E4 | 排课的门店不在课程可用门店列表 | `COURSE_LOCATION_UNAVAILABLE` 409 |
| E5 | 排课选 is_schedulable=false 或 inactive 教练 | `INSTRUCTOR_NOT_SCHEDULABLE` 409 |
| E6 | ends_at ≤ starts_at 或 starts_at 已过去 | `SESSION_TIME_INVALID` 400 |
| E7 | 同教练同时段重叠再排一节 | `SESSION_INSTRUCTOR_CONFLICT` 409（DB EXCLUDE 23P01，不裸 500） |
| E8 | 取消已 cancelled / completed 场次 | `SESSION_CANCEL_NOT_ALLOWED` 409 |
| E9 | 删除被 scheduled 场次引用的课程模板 | `COURSE_IN_USE` 409，弹窗保持打开 |
| E10 | 删除被 scheduled 场次引用的门店（/locations 删） | `LOCATION_IN_USE` 409（CountActiveReferences 已纳入 class_sessions） |
| E11 | 只读账号 13900139777 在 /courses /schedule 写按钮 | 全 disabled + Hint（course.create/session.create 无） |
| E12 | 无 session.create 角色直接 POST /class-sessions | `PERMISSION_DENIED` 403 |
| E13 | data_scope=assigned_locations 用户列场次 | 只见 scope 内门店场次；越权详情返 404 |

## 并发 / 事务

| # | 场景 | 预期结果 |
|---|---|---|
| C1 | 两请求同教练同时段并发排课 | 至多一条成功，另一条 409（DB EXCLUDE，非应用层先查后插的 race） |
| C2 | 课程 create 含 location_ids 写 availability 与 course 同事务 | 失败整体回滚，不留半条 availability |

## 执行方式

后端：`go test ./...`（含 repo DB 单测 + service fake-repo 单测）。
e2e：Playwright `web/e2e/batch-11-course-session.spec.ts` 真实栈跑 H1–H7 + E7 + E11，自带 teardown（软删课程/取消场次/删分类）。**e2e 前必重建并重启 :8081 后端**（旧二进制掩盖新逻辑——Batch 7 踩坑）。验收除 e2e 外必跑一次 prod `pnpm build`（Batch 8 踩坑）。
