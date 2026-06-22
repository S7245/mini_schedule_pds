# Batch 13b 测试场景 — 权益产品 + 发放

> 契约：[batch-13b-entitlement.md](batch-13b-entitlement.md)。账号见 [[brand21_test_accounts]]：owner `18816820405/admin123`（全权限 fast-path）；只读 `13900139777/admin123`（有 entitlement.view、无 manage/adjust）。门店「讯美广场」id=1，已有已发布课程模板。需先有测试学员（13a 已能建）。后端 :8081，前端 :3002。

## Happy Path

| # | 步骤 | 预期结果 |
|---|---|---|
| H1 | 登录 owner → 导航「权益产品」→ `/entitlement-products` | 空态 + 「新增产品」可点 |
| H2 | 建课包：类型 class_pack + 次数 10 + 有效期 90 天 + 适用范围 all | 列表新增，显示 10 次 / 90 天 / 全部门店·课程 |
| H3 | 建会员卡：membership_card + 有效期 30 天 + 每日上限 1 + scope specific 选「讯美广场」+ 某课程 | 列表显示「不限次」+ 适用范围具体；total 列空/不限 |
| H4 | 编辑课包（改名 + 改每周上限）→ 保存；再「停用」会员卡 | 列表反映改动；会员卡状态 inactive |
| H5 | 进学员详情「权益」Tab → 发放课包（选产品 + 默认今天）| 权益列表新增：剩余 10/10、到期=今天+90、状态 active；流水有 1 条「开通」 |
| H6 | 给同学员发放会员卡（active 的那张）| 显示「不限次」、到期=今天+30、状态 active |
| H7 | 对课包权益「调整」：+5（原因）→ 再 -3（原因）| 剩余 12 → 流水抽屉显示 开通(+10,bal10) / 调整(+5,bal15) / 调整(-3,bal12) |
| H8 | 权益「冻结」→「恢复」→「作废」| 状态 badge 已冻结 → active → 已作废 |
| H9 | 登出登录只读 13900139777 → /entitlement-products + 学员权益 Tab | 产品列表/权益列表可见；新增/编辑/启停/发放/调整/状态 按钮全 disabled（Hint） |

## Edge Cases

| # | 场景 | 预期结果 |
|---|---|---|
| E1 | 建与现有 active 产品同名 | `ENTITLEMENT_PRODUCT_NAME_DUPLICATED`(409)，弹窗保持打开 |
| E2 | class_pack 不填次数 / 次数 0 | 前端 zod 拦截「次数必填且>0」；后端兜底 400 |
| E3 | 有效期天数 ≤ 0 | 前端校验「有效期必须>0」 |
| E4 | API 直连 POST 产品 scope=specific + location_ids 含非本品牌/停用门店 | `ENTITLEMENT_SCOPE_INVALID`(400)，事务回滚 |
| E5 | 从已停用产品发放 | `ENTITLEMENT_PRODUCT_INACTIVE`(409) |
| E6 | API 直连：调整课包 delta=-999（超剩余）| `ENTITLEMENT_INSUFFICIENT`(409)，remaining 不变、无流水 |
| E7 | 对会员卡（不限次）调整额度 | 前端禁用额度调整；API 直连 → `ENTITLEMENT_INSUFFICIENT`(409，不限次不可调) |
| E8 | 对已作废权益 adjust 或改状态 | `ENTITLEMENT_NOT_ADJUSTABLE`(409) |
| E9 | settle-expired 落库：API 直连发放后把 expires_at 改到昨天 → 再 `GET /learners/:id/entitlements` | 该权益 status 落库为 `expired`（DB 实查确认非仅前端显示） |
| E10 | settle-depleted 落库：调整课包剩余到 0 | 该权益 status 落库为 `depleted`（adjust 后 settle） |
| E11 | API 直连 GET 跨品牌 / 不存在的 entitlement id | `ENTITLEMENT_NOT_FOUND`(404) |
| E12 | 权限 backfill（§21.1）：course_operator 仅 view、finance_support 全部 | course_operator 有 view、调整 **403**；finance_support / brand_admin 有 view+manage+**adjust**（财务/售后=全部，可调，勿误判 403）。注：验收时把只读账号 13900139777 重指派为 course_operator（only-view），保留此状态 |

## 执行方式

混合（镜像 12b/13a）：H1-H9 + E1/E2/E3/E5/E8 可视化走 chrome-devtools UI；E4/E6/E7/E9/E10/E11（scope/insufficient/settle 落库/越权）走后端 API 直连 + DB 实查验证 settle。e2e 另开 session 跑真实栈，自带 teardown（作废测试权益 + 停用测试产品；产品无硬删）。settle 落库须 `psql` 实查 learner_entitlements.status 确认，不能只看前端派生显示。
