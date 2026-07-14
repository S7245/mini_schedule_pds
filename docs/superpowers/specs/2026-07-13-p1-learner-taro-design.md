# P1 学员端 Taro（微信小程序 + 公众号 H5）设计

> 日期：2026-07-13 · 状态：设计已确认（4 决策点会话内拍板）· 上游蓝图：`2026-07-13-multi-client-framework-design.md`
> 拆 P1a/b/c 三小批交付，每批独立验收、零回归（稳 > 快）。

## 1. 范围与决策（grill 拍板）

| 决策点 | 结论 |
|---|---|
| HttpClient 错误语义（蓝图 P1 首决策，全端统一） | **throw 类型化 `CoreApiError`**（含 code/message/details）+ **`Notifier` 端口**承担 toast。与现有 web client + react-query 惯例一致 |
| 小程序 AppID | **尚无** → P1 用微信开发者工具测试号 + 现有 **mock 登录**（后端 `wechat-login` 的 `dev_+code`）跑通全链路；真 `code2session` + 公众号网页授权 = 独立后端 batch，等 AppID（处置同商户号） |
| 页面范围 | **仅预约主链路 8 页**：登录 / 首页 / 课程表(class-sessions) / 我的预约(bookings) / 候补 / 权益(entitlements) / 上课记录(records) / 我的(profile)。legacy 双轨页 courses·trainings **不迁**，随旧 app 退役一并砍 |
| 拆批 | **P1a 基建+登录 → P1b 课程表/预约/候补 → P1c 我的+H5 验收+退役旧 app** |

**P1 全程零后端改动**（mock 登录已存在）。真实登录（code2session + snsapi 网页授权）为后置 batch，卡 AppID。

## 2. 架构

### 2.1 新增 `apps/learner-taro`（Taro 4.x + React + TS）

- 编译目标：`weapp`（微信小程序）+ `h5`（公众号链接），一套代码。
- 消费 `@mini-schedule/core`（领域函数 + ports）与 `@mini-schedule/types`；**不消费** `@mini-schedule/api`（其 sonner/localStorage web 耦合，见蓝图 §6）。
- 版本锚定 Taro 4.x，锁 minor（蓝图风险表）。monorepo 接入需配 Taro 的 workspace/sourceRoot（一次性成本）。

### 2.2 core 的 P1 增量（api 包演进路径的第一步）

```
packages/core/src/
├── ports.ts          (已有) + Notifier 端口（showToast(message, type)）
├── errors.ts         新：CoreApiError { code, message, httpStatus?, details? }
├── client.ts         新：createApiClient(http: HttpClient, notifier?: Notifier)
│                        → code!=='OK' 时 throw CoreApiError（notifier 由调用端决定何时提示）
└── app/              新：学员域纯函数（注入 client）
    ├── auth.ts       wechatLogin(code, brandId, nickname) → token/profile
    ├── sessions.ts   listSessions / getSession / usableEntitlements
    ├── bookings.ts   listMy / create / cancel
    ├── waitlist.ts   listMy / join / cancel
    └── learner.ts    entitlements / records / profile(get/update)
```

- 领域函数**纯 async 函数**（React 无关），单测用 fake HttpClient 全覆盖（vitest，延续 P0 模式）。
- react-query hooks **不进 core**（保持 headless）：learner-taro 内薄封一层本地 hooks（@tanstack/react-query 在 Taro React 可用）。
- 现有 `packages/api` 不动（web 端继续用）；待 core/app 稳定后，web 端 app.ts 的重复实现在旧 `apps/app` 退役时自然消亡。

### 2.3 端口的 Taro 实现（P1a 交付）

| 端口 | Taro 实现（weapp + h5 同一份） |
|---|---|
| `HttpClient` | `Taro.request` 封装（weapp 走原生请求域名白名单；h5 走 XHR，同源或直连+CORS） |
| `KeyValueStorage`/`AuthTokenStore` | `Taro.getStorage/setStorage` 异步封装 |
| `Notifier` | `Taro.showToast` |

**H5 API 基址**：公众号 H5 无 Next rewrites → 直连后端域名，需后端 `MINI_SCHEDULE_CORS_ALLOWED_ORIGINS` 放行 H5 域（注意 keysToBind 已知坑）；小程序侧在 mp 后台配 request 合法域名。开发期都指 `http://localhost:8082`（api-app）。

### 2.4 品牌定位

沿用现状：构建时默认 brandId（对应 `NEXT_PUBLIC_DEFAULT_BRAND_ID`）+ 启动参数覆盖（小程序 scene/query、H5 URL query）。多品牌「品牌空间」完整版后置。

## 3. 分批交付

| 批 | 交付物 | 验收 |
|---|---|---|
| **P1a 基建+登录** | learner-taro 接入 monorepo（weapp+h5 双目标可构建）；core：errors/client/Notifier/app/auth + 单测；Taro 三端口实现；登录页 + 首页壳（登录态展示 profile 昵称） | `pnpm --filter core test` 绿；weapp 构建产物在微信开发者工具（测试号）登录成功并展示昵称；h5 构建可本地打开完成同流程；web 零回归 |
| **P1b 预约主链路** | core/app：sessions/bookings/waitlist + 单测；页面：课程表、场次详情+预约（含可用权益选择）、我的预约+取消、候补加入/取消 | 开发者工具跑通 预约→我的预约→取消→候补 全链路（连本地 api-app + dev DB）；core 单测绿；brand 端可见学员操作产生的通知（Batch 18 链路侧证） |
| **P1c 我的+收尾** | core/app：learner(权益/记录/profile 更新) + 单测；页面：权益、上课记录、我的（含资料编辑）；H5 目标全页面验收；**退役 `apps/app`**（git rm + 根 CLAUDE.md 端口表更新 + turbo/vercel 摘除说明） | 8 页在 weapp+h5 双目标全部可用；`pnpm build` 全仓绿（旧 app 移除后）；文档同步 |

## 4. 风险与兜底

| 风险 | 兜底 |
|---|---|
| Taro 4 + pnpm monorepo 接入坑（hoisting/sourceRoot） | P1a 第一个 task 就是"空工程双目标构建通过"，先验证再写业务；折不动降级为 Taro 独立 lockfile 目录（蓝图逃生舱同理） |
| `toLocaleString` 在小程序引擎的 Intl 差异（P0 已留标记） | P1a 在真机/工具里验证 `formatNotificationTime`；异常则 core 改手写补零（有单测护栏） |
| react-query 在 Taro 的兼容性 | P1a 冒烟先验证；不行则退化为 core 函数 + 本地 useState/useEffect 薄封装（领域函数不受影响） |
| mock 登录与真登录切换 | `wechatLogin` 领域函数签名以 code 为入参，后端换真 code2session 时前端零改动 |
| 旧 app 退役时机 | P1c 且 H5 全页验收通过后才 git rm；期间双存不冲突（Taro dev 端口错开 3003） |

## 5. 不做什么

- 不做抖音/支付宝目标（P4）；不做微信支付/订阅消息（卡商户号/模板）；不做真 code2session（卡 AppID，独立后置 batch）；不做多品牌空间完整版；不迁 legacy courses/trainings 页；react-query hooks 不进 core。

## 6. 成功标准

- 一套 Taro 代码产出 weapp + h5 双目标，8 页预约主链路全部可用（mock 登录）。
- 学员域业务逻辑 100% 在 `core/app`（纯函数 + 单测），页面零业务规则。
- 全程 web/brand/admin 零回归；旧 `apps/app` 退役后全仓构建绿。
