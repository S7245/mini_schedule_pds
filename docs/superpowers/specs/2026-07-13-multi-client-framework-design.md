# 多端框架选型设计（学员端 + 商家端）

> 日期：2026-07-13 · 状态：设计已确认，待落地
> 这是**总蓝图**，不是单个 batch。每个阶段（P0…P4）落地时各自再走 grill→spec→plan→TDD→code-review。

## 1. 背景与目标

Mini Schedule 需要覆盖多端：

- **学员端**：微信小程序、抖音小程序、支付宝小程序、公众号 H5 链接。
- **商家端**：Web 网页、iOS（APP + 桌面）、Android（APP + 桌面）。

现状（2026-07-13 核查）：

- `web/apps/app` = 纯 **Next.js H5** 学员端，无任何小程序框架，无真实小程序。
- `web/apps/brand`、`web/apps/admin` = **Next.js Web** 商家端；admin 有少量 Electron 雏形；无移动原生端。
- 全项目**无** Taro / uni-app / React Native / Flutter / Capacitor —— 跨端为绿地。
- 技术栈：**React + TypeScript + Next.js**；后端 Go DDD 干净分层。
- `mini_schedule_web_electron` 是一个独立 Electron monorepo，但其 `DESIGN.md` 为无关占位内容（Claude 品牌分析模板），视为废弃/误建。

### 已确认的范围与约束（brainstorming 决策）

| 决策点 | 结论 |
|---|---|
| 学员端 v1 范围 | **仅微信小程序 + 公众号 H5**（抖音/支付宝后置） |
| 商家端 v1 范围 | **全都要**：Web + iOS/Android APP + 桌面 |
| 商家移动原生程度 | **手机要真原生**（→ React Native，不走 WebView 过渡） |
| 团队产能 | **1-2 人，主靠 AI** |
| 交付节奏偏好 | **稳 > 快**，可以慢但要扎实、分阶段、零回归 |

## 2. 核心架构：无头共享内核 + 各端只写 UI

1-2 人撑住多端的唯一办法是**业务逻辑写一遍，UI 各写各的**。

```
                    ┌─────────────────────────────────────┐
                    │         共享内核 (TS, 无 UI)          │
                    │  @mini-schedule/api    (已有,API 客户端)│
                    │  @mini-schedule/types  (已有,类型)     │
                    │  @mini-schedule/core   (新,业务逻辑)   │
                    └─────────────────────────────────────┘
                       ▲          ▲            ▲          ▲
   ┌────────────────┐ ┌──────────┐ ┌────────────┐ ┌─────────────┐
   │ 商家 Web        │ │ 商家桌面  │ │ 商家移动    │ │ 学员端       │
   │ Next.js         │ │ Tauri 包  │ │ React Native│ │ Taro         │
   │ brand/admin(现有)│ │ 壳 Next   │ │ (Expo)真原生│ │ 微信小程序+H5 │
   └─────────────────┘ └──────────┘ └────────────┘ └─────────────┘
```

### 端-框架映射

| 端 | 框架 | 现状 → 目标 |
|---|---|---|
| 商家 Web | **Next.js**（现有 brand/admin） | 已有，继续用 |
| 商家桌面(Win/macOS) | **Tauri** 包壳 Next Web | 绿地（替代废弃的 electron monorepo） |
| 商家移动(iOS/Android) | **React Native (Expo)** 真原生 | 绿地 |
| 学员小程序(微信) | **Taro**（React 系） | 绿地 |
| 学员 H5(公众号) | **Taro 编译 H5** | **替代现有 `apps/app` 的 Next H5** |
| 学员抖音/支付宝小程序 | **Taro 加编译目标** | 后置（P4） |

两处替代：

1. **`apps/app`（Next H5）→ Taro 版取代**：Taro 一套代码同时出微信小程序 + 公众号 H5，无需再单独维护 Next H5。迁移量主要是 UI（业务逻辑走内核）。
2. **桌面用 Tauri（非 Electron/非现有 electron monorepo）**：包体小 ~10 倍、内存省、Rust 壳 + 系统 WebView；"就是个大屏后台"。可逆决策，折不动可退回 Electron。

## 3. 共享内核边界

| 放进内核（写一遍，全端用） | 留在各端 UI |
|---|---|
| API 客户端（请求/鉴权 token 附加/`{code,message,data}` 解析） | 页面/组件/路由/导航 |
| 类型（后端 Swagger 生成 `api.d.ts`） | 端特有交互（小程序授权、RN 相机扫码、桌面托盘） |
| zod 校验 schema（表单规则） | 端特有存储（localStorage / RN AsyncStorage·SecureStore / 小程序 Storage） |
| 纯业务规则（金额时间格式化、预约窗口判断、权限码常量、事件标签如 notification-format） | UI 状态、样式、动效 |

**硬约束**：内核**不 import 任何 UI/平台 API**（不碰 `window`、不直接 `fetch`、不碰 RN/小程序 API）。平台差异用**依赖注入**：内核定义 `interface Storage / HttpClient / AuthTokenStore`，各端启动时注入自己的实现。这样内核能在 Next(Node/浏览器)、RN(Hermes)、Taro(小程序引擎) 三种运行时都跑。

## 4. 仓库结构（扩展现有 `web/` 单 monorepo）

选择**单 monorepo**（非多仓）。决定性理由：**原子改动**——改 `core/` + 同步改各端可一个 PR 完成、一次 CI、AI 一个上下文看全。拆多仓需把 core 发私有 registry / submodule，"改内核要发版各端再升级"对小团队是持续摩擦，废掉共享内核的最大好处。

```
web/  (pnpm + Turborepo)
├── apps/
│   ├── admin/            Next.js 平台后台 (现有)
│   ├── brand/            Next.js 商家 Web (现有)
│   ├── app/              → 迁移/退役（被 learner-taro 取代）
│   ├── merchant-mobile/  React Native (Expo)   ← 新
│   └── learner-taro/     Taro 微信小程序 + H5   ← 新
├── packages/
│   ├── api/              (现有)
│   ├── types/            (现有)
│   ├── core/             业务逻辑内核           ← 新
│   └── config/           (现有)
└── 桌面：brand/admin 各加 src-tauri/，Tauri 包壳，不单开 app
```

一次性接入成本：RN 的 Metro 要配 `watchFolders`、Taro 要配 `sourceRoot`/hoisting。**逃生舱**：若 RN 的 Metro 在 monorepo 里折腾不动，`merchant-mobile` 最独立，后期单独拆仓成本最低——但不要一开始就拆。

## 5. 分阶段路线图（贴合 1-2 人，稳 > 快）

不一次铺三套 UI。每阶段独立可交付、可验收（对齐后端 batch 流程）。

| 阶段 | 做什么 | 为什么这个顺序 |
|---|---|---|
| **P0 地基** | 抽 `packages/core`（收编分散在 brand/app 的业务逻辑：notification-format、金额时间、zod、权限码常量）+ 定义 Storage/Http/Auth 注入接口 | 后面每端都靠它，先立地基 |
| **P1 学员端** | `learner-taro`：Taro → 微信小程序 + 公众号 H5，迁移现有 `apps/app` 页面；旧 Next app 待 Taro H5 验收通过后退役 | 当前最大产品缺口（真小程序=0）；且**首次真实验证内核跨 Taro 运行时** |
| **P2 桌面** | brand/admin 加 Tauri 壳 → Win/macOS | 最便宜（包壳现有 Web），快速凑齐"桌面端" |
| **P3 移动** | `merchant-mobile`：React Native (Expo) 真原生，先做高频屏（登录、今日工作台、排课、现场签到扫码） | 最贵、体验要求最高，放最后；内核已被 Taro 验证，RN 复用顺 |
| **P4 扩端** | Taro 加编译目标：抖音 / 支付宝小程序 | 一套 Taro 代码，边际成本低，按业务开 |

## 6. 关键风险与兜底

| 风险 | 说明 | 兜底/决策 |
|---|---|---|
| 内核跨运行时 | 小程序引擎缺很多 Web API | 内核纯 TS + 依赖注入，不碰平台 API；P1 Taro 首验证 |
| 鉴权差异 | localStorage / `wx.login` code / RN SecureStore 各不同 | 内核定义 `AuthTokenStore` 接口各端注入；复用后端 app 端登录桥接（Batch 14） |
| Taro 取代 Next H5 迁移 | `apps/app` 页面搬到 Taro | 逻辑走 core 后迁移≈重画 UI；分屏迁移；旧 app 保留到验收通过再退役 |
| Tauri 学习成本 | Rust 壳（基本不写 Rust） | 桌面=包 Web + 少量原生(托盘/自动更新)；折不动退回 Electron（可逆） |
| RN 现场能力 | 扫码签到/拍照是移动真原生核心价值 | Expo 生态（expo-camera/barcode）成熟；正是移动值得上 RN 而非 WebView 的落点 |
| 微信支付/订阅消息 | 小程序支付、订阅消息模板 | 卡真实商户号 + per-brand 模板审核（已知 P0）；框架层先留接口，不阻塞 UI |

## 7. 不做什么（YAGNI / 明确排除）

- **不**把商家 Web 推倒重写成 React Native Universal（方案 B）——会废掉能跑的 Next brand/admin。
- **不**为省事把移动做成 WebView 包壳（方案 C）——与"手机真原生"冲突（用户已明确否决）。
- **不**一开始就为 RN/Taro 拆独立仓库——先单 monorepo，留逃生舱。
- v1 **不**做抖音/支付宝小程序（P4 再开）。
- **不**在框架层等微信支付真实接入——留接口，不阻塞。

## 8. 成功标准

- `packages/core` 能被 Next / Taro / RN 三种运行时消费且无平台 API 泄漏（P0+P1 验证）。
- 学员微信小程序 + 公众号 H5 从一套 Taro 代码产出，业务逻辑零重复。
- 商家五端（Web/桌面/iOS/Android）业务规则来自同一 `core`，无三处重复。
- 每阶段独立交付、可验收、零回归。

## 9. 下一步

本蓝图确认后，进入 **P0（抽 `packages/core`）** 的实施计划（writing-plans）。后续每阶段各自 grill→spec→plan→TDD→code-review。
