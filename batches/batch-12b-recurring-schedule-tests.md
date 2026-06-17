# Batch 12b 测试场景 — RecurringSchedule 循环排课

测试品牌：brand 21｜owner `18816820405/admin123`｜只读 `13900139777/admin123`｜门店「讯美广场」id=1｜教练「张三」instructor_profile id=1｜需至少一个已发布课程模板（可用门店含讯美广场）。
后端端口 8081（api-brand）。铁律：测前重建/重启后端 + 前端 `rm -rf .next` 重启 + curl 预热；前端 filter 用包名 `@mini-schedule/brand`。

## Happy Path

| # | 步骤 | 预期结果 |
|---|---|---|
| H1 | /schedule「循环排课」Tab → 循环排课：门店讯美广场→已发布课程→教练张三→勾周一+周三→开始日期=下周一→重复 4 周→09:00→60 分钟→提交 | 结果「成功 8 节 / 跳过 0」；列表新增该循环排课（已生成数=8，状态 active）|
| H2 | 切到「单场次」Tab | 可见这 8 节 scheduled 场次（教练张三、讯美广场）|
| H3 | 循环排课列表点「详情」 | 显示模板信息（周一·周三 / 起止 / 09:00·60）+ 已生成 8 节场次列表 |
| H4 | 绑资源「1号教室」重复 H1（错开时段避冲突）→ 提交 | 成功生成，每节绑该资源，容量默认资源容量 |
| H5 | 取消某循环排课 | status=已取消；单场次 Tab 那 8 节仍 scheduled（未被取消）|

## Edge Cases

| # | 场景 | 预期结果 |
|---|---|---|
| E1 | 新循环与已有场次同教练同时段重叠（部分日期）| 部分成功：结果「成功 X / 跳过 Y」+ 跳过清单列出日期·时间·原因「教练冲突」|
| E2 | 全部 occurrence 都与已有场次同教练冲突 | RECURRING_ALL_CONFLICT 409 → toast「全部时段冲突，未生成」，列表不新增（不落空壳）|
| E3 | 绑资源的循环与已占该资源时段冲突（换教练避教练冲突）| 跳过清单原因「资源冲突」|
| E4 | 不勾任何周几提交 | 400「请至少选择一个星期几」|
| E5 | 同时填结束日期和重复周数（或都不填）| 400「结束日期与重复周数二选一」|
| E6 | 开始日期早于今天 | 400「开始日期不能早于今天」|
| E7 | 跨度>26 周 或 预计生成>200 节 | 400「排课区间过长 / 生成场次过多」|
| E8 | 绑已停用资源 | RESOURCE_NOT_AVAILABLE（下拉不出现 / 直连 API 409）|
| E9 | 取消已是 cancelled 的循环排课 | RECURRING_CANCEL_NOT_ALLOWED 409 |
| E10 | 只读 13900139777 | 「循环排课」按钮 + 取消全 disabled；直连 POST 403 |
| E11 | 有 active 循环排课时删讯美广场门店 | LOCATION_IN_USE 409（active recurring 计入 CountActiveReferences）|

## 后端单测覆盖（DB 集成 + service）

- repo Generate：happy（周一+周三×4周=8 节、recurring+weekdays 落库、session.recurring_schedule_id 回填）；部分冲突（预置冲突场次→部分 skip + created>0，reason 分流 instructor/resource）；全冲突→RECURRING_ALL_CONFLICT 且 recurring 行回滚不留；绑资源容量默认；批级校验失败（course 未发布/门店停用/教练不可排/资源停用）整批不生成。
- repo List/GetByID：反范式名 + weekdays 聚合 + session_count；detail 带 sessions；跨租户/软删隔离。
- repo Cancel：active→cancelled 非级联（已生成场次状态不变）；非 active→RECURRING_CANCEL_NOT_ALLOWED。
- service：入参校验（weekday 空/越界、结束条件 XOR、start_date<今天、上限 26 周/200 节、时区生成日期正确）；data_scope（assigned 越权 404）；容量优先级。
- location CountActiveReferences：纳入 active recurring（含/取消后不含）。

## 执行方式

后端 `go test ./...`；前端 `pnpm --filter @mini-schedule/brand lint && build`。e2e 用户另开 session 跑真实栈（UI 建循环 → page.request 校验生成场次 → 取消 → teardown 取消生成的场次 + cancel 循环）。
