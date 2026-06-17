# Batch 12b 循环排课 — 回归结论（2026-06-17）

环境：brand 21｜owner 18816820405/admin123（user_id=16, 显示名「李四」）｜只读 13900139777/admin123（显示名「B7测试员工」）｜后端重建重启 :8081｜前端 `rm -rf .next` 重启 :3002｜curl 预热 OK。
执行方式：UI 经 chrome-devtools 驱动（happy path + 冲突/校验/权限的可视化部分）+ 后端 API 直连校验（XOR/越界/资源/权限/级联）。

## 结论：全部通过 ✅（H1–H5 + E1–E11，17/17）

| # | 结果 | 关键证据 |
|---|---|---|
| H1 | ✅ | 张三/讯美广场/晨间瑜伽/周一+周三/起 2026-06-22/4 周/09:00/60min → 「已生成 8 节，跳过 0 节」；列表行：已生成 8、状态 进行中 |
| H2 | ✅ | 单场次 Tab 出现这 8 节（6/22,6/24,6/29,7/1,7/6,7/8,7/13,7/15 @09:00）；API 确认 8 节 scheduled、cap 8 |
| H3 | ✅ | 详情头「晨间瑜伽·讯美广场·周一、周三·09:00」+ 8 节 scheduled 列表 |
| H4 | ✅ | 绑 1号教室、错峰 11:00 → 8 节生成；选资源后容量字段自动 8→10；API 确认每节 res=1号教室、cap=10 |
| H5 | ✅ | 取消确认弹窗文案正确（仅停模板不取消场次）；id=1 → 已取消，操作列变「—」；其 8 节仍 scheduled（非级联） |
| E1 | ✅ | 张三 09:00 周一+周二 → 「成功 4 / 跳过 4」；跳过清单 4 个周一 reason「教练时段冲突」 |
| E2 | ✅ | 张三 09:00 周一+周三（全撞 id=1）→ 弹窗内联错误「所选时段全部冲突，未生成任何场次」+ 8 行清单；列表 total 不增（不落空壳，已核对 RECURRING_ALL_CONFLICT） |
| E3 | ✅ | B7 11:00 绑 1号教室 周一+周二（周一撞 id=2 资源）→「成功 4 / 跳过 4」；跳过 reason「资源时段冲突」 |
| E4 | ✅ | 不勾周几 → 内联校验「请至少选择一个星期几」，弹窗不关、不提交 |
| E5 | ✅ | API：同填 end_date+repeat_weeks → 400「结束日期与重复周数二选一」；都不填 → 同 400（前端 radio 已强制二选一，故走 API 验证）|
| E6 | ✅ | API：start_date=2026-06-10 → 400「开始日期不能早于今天」 |
| E7 | ✅ | API：repeat_weeks=30 → 400「排课区间过长（最长 26 周）」（见下「观察」：>200 节守卫被 26 周上限遮蔽，结构上不可独立触发）|
| E8 | ✅ | UI 资源下拉只列 active「1号教室」，停用资源不出现；API 直绑停用资源 id=7 → 409 RESOURCE_NOT_AVAILABLE「所选资源已停用或不属于该门店」 |
| E9 | ✅ | API：对已 cancelled 的 id=1 再取消 → 409 RECURRING_CANCEL_NOT_ALLOWED「仅可取消进行中的循环排课」 |
| E10 | ✅ | 只读账号：循环排课创建按钮 disabled、全部取消按钮 disabled；API 直连 POST → 403 PERMISSION_DENIED、cancel → 403 |
| E11 | ✅ | 有 active 循环时 DELETE /locations/1 → 409 LOCATION_IN_USE（见「观察」：该门店同时有员工/场次引用，未隔离仅 recurring 这一因子，由后端单测覆盖隔离）|

## 观察 / 备注（非阻断）
1. **>200 节守卫被 26 周上限遮蔽**：service 先查 26 周（line 156），再查 >200（line 169）。26 周 × 7 天 = 182 < 200，无论 repeat_weeks 还是 end_date 模式都先撞 26 周上限，故 E7 的「>200 节」分支结构上不可独立触发，只是纵深防御。E7 实测覆盖的是 26 周分支。
2. **E11 未隔离 recurring 因子**：门店 1 同时被员工任职/已排场次引用，E2E 的 409 不能证明「active recurring 单独计入 CountActiveReferences」。该隔离由后端单测「location CountActiveReferences: 纳入 active recurring」覆盖。
3. **class-sessions 列表 DTO 不暴露 recurring_schedule_id**（list 里恒为 null），但 GET /recurring-schedules/:id 能正确聚合回该循环的场次，证明 FK 回填生效。属 DTO 字段取舍，非缺陷。
4. **E2 的全冲突在弹窗内联呈现**（api-error 区 + 跳过清单），非顶部 toast；文案为 mapApiError 的「所选时段全部冲突，未生成任何场次」。与契约语义一致。

## Teardown 已完成（环境已还原至基线）
- 取消 24 节生成场次（H1×8, H4×8, E1×4, E3×4）全部 → cancelled。
- 取消循环排课 id=1（H5 UI 已取消）、id=2/3/5（API 取消）→ 全 cancelled。
- 删除本轮新建的测试资源 1号教室(id=6)、停用教室(id=7)→ 资源列表回到空。
- **讯美广场(id=1) 未删，仍 active**。
- 残留：4 条 cancelled 循环排课 + 24 节 cancelled 场次（按 teardown 约定保留，不影响后续）。

## 复跑要点
- 后端 `go build -o /tmp/api-brand ./cmd/api-brand` 重启（旧二进制掩盖改动）；前端 `pnpm --filter @mini-schedule/brand dev`（:3002）+ `rm -rf .next`。
- 今天 2026-06-17（周三）→ 下周一 = 2026-06-22；周一+周三×4 周 = 6/22,6/24,6/29,7/1,7/6,7/8,7/13,7/15。
- 周几复选框是 `sr-only` 隐藏 input，chrome-devtools 的 click/fill 点不到，需 `el.click()` 经脚本触发；select 用原生 value setter + dispatch change 才能更新 React state。
- 时间存 UTC，09:00 本地 = 01:00Z，11:00 = 03:00Z（按 starts_at 子串过滤会漏，需换算）。
