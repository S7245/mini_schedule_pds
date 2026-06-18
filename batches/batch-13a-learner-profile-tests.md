# Batch 13a 测试场景 — 学员档案管理

> 契约：[batch-13a-learner-profile.md](batch-13a-learner-profile.md)。账号见 [[brand21_test_accounts]]：owner `18816820405/admin123`（李四，user_id=16）；只读 `13900139777/admin123`（staff.view+location.view）。门店「讯美广场」id=1。后端 :8081，前端 :3002。

## Happy Path

| # | 步骤 | 预期结果 |
|---|---|---|
| H1 | 登录 owner → 点导航「学员管理」 | 进 `/learners`，空数据显示空态；「新增学员」按钮可点 |
| H2 | 新增学员：手机号 `13700001111` + 昵称「测试学员A」+ 主门店「讯美广场」+ 学号 `S001` + 选 1 个标签 | 列表新增一行，显示昵称/手机号/主门店名/标签 chip/状态 active |
| H3 | 编辑该学员：改昵称「学员A改」+ 备注 + 加/减标签 | 列表与详情页实时反映新昵称/备注/标签 |
| H4 | 行操作「冻结」→ ConfirmDialog 确认 | status badge 变 frozen；再「解冻」→ active |
| H5 | 行操作「删除」（无引用）→ 确认 | 行消失（软删 deleted_at）；列表 total -1 |
| H6 | 标签页 `/learner-tags`：建标签「VIP」(颜色) → 回学员表单可选 → 停用「VIP」 | 标签列表新增；学员表单标签下拉含 VIP；停用后状态 inactive |
| H7 | 列表搜索 `q=13700`、状态筛选 frozen、主门店筛选「讯美广场」 | 各自 server-side 过滤返回正确子集（直连 `GET /brand/learners?q=` 同结果） |
| H8 | 进 `/learners/[id]` 详情 | 基础信息卡（昵称/手机号/学号/主门店/备注/状态）+ 标签区；权益/预约/履约 Tab 显示空态占位 |
| H9 | 登出，登录只读 `13900139777` → 进 `/learners` | 列表可见；新增/编辑/冻结/删除按钮全 disabled（Hint tooltip） |

## Edge Cases

| # | 场景 | 预期结果 |
|---|---|---|
| E1 | 同品牌再建手机号 `13700001111` 的学员 | `LEARNER_ALREADY_EXISTS`(409)；弹窗保持打开 + inline/toast 错误 |
| E2 | 新建学员学号填已存在的 `S001` | `LEARNER_NO_DUPLICATED`(409) |
| E3 | 标签页建重名标签 | `LEARNER_TAG_NAME_DUPLICATED`(409) |
| E4 | API 直连 `POST /learners` 传不存在的 `tag_ids:[999999]` | `LEARNER_TAG_NOT_FOUND`(404)，事务回滚不创建学员 |
| E5 | API 直连：把品牌订阅 `MaxLearners` 调到当前学员数，再建 | `QUOTA_EXCEEDED`(**409**，复用 SubscriptionGuard 约定)；前端 toast + inline |
| E6 | 店长账号（仅辖某门店）看 `/learners` + 访问他门店学员 detail | 列表只含主门店∈辖区的学员；越权 detail 返 `LEARNER_NOT_FOUND`(404) 不泄漏 |
| E7 | （13b/13c 落地后）删除有 active 权益/未来预约的学员 | `LEARNER_IN_USE`(409)；本批无数据恒不触发，guard 已就位 |
| E8 | 手机号格式非法（如 `abc`） | 前端校验拦截「请输入合法手机号」；后端兜底 400 |
| E9 | 必填缺失（phone 空）提交 | 前端 RHF 校验拦截「请输入合法手机号」（min 长度兜空），不发请求 |
| E10 | API 直连 `GET /learners/:id` 传别的 brand / 已软删 id | `LEARNER_NOT_FOUND`(404) |
| E11 | 冻结的学员仍可被编辑基础信息 | 编辑成功（冻结只挡自助预约，不挡后台编辑） |

## 执行方式

镜像 Batch 12b 回归：Happy Path（导航/列表/建改删/筛选/权限门可视化部分）走 chrome-devtools UI；额度门(E5)/标签无效 id(E4)/data_scope 越权(E6/E10)/IN_USE(E7) 走后端 API 直连验证（前端校验已强制或本批无数据时）。e2e 由用户另开 session 跑真实栈，自带 API teardown（软删测试学员 + 停用测试标签）。
