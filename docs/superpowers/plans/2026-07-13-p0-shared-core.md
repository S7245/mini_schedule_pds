# P0 共享内核 `packages/core` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 `web/` monorepo 建立无头共享内核 `@mini-schedule/core`（含 Vitest 单测基建），迁移第一块纯业务逻辑（通知事件标签/时间格式化）作为模式验证，并定义 Storage/HttpClient/AuthTokenStore 依赖注入接口，为后续 Taro/RN/Tauri 各端复用打地基。

**Architecture:** `core` 是纯 TS、无 UI、不 import 任何平台 API 的包，通过依赖注入隔离平台差异（存储/网络/鉴权）。各端消费 `core`，业务逻辑写一遍。P0 只做地基 + 一块代表性迁移（notification-format），后续纯逻辑迁移按同一模式在独立 plan 中迭代。

**Tech Stack:** TypeScript、pnpm workspace + Turborepo、Vitest（新引入的单测框架）。`core` 沿用 `packages/api` 的「raw `.ts` via `exports`」消费模式，无需额外 Next 配置。

**约束（贯穿全程）:** 稳 > 快；每个 Task 独立可验收；对现有 brand/admin **零回归**（每次改动后 `pnpm --filter @mini-schedule/brand build` 必须通过）。

**参考:** 蓝图 `pds/docs/superpowers/specs/2026-07-13-multi-client-framework-design.md`（第 3 节内核边界、第 5 节 P0 定义）。

**工作目录:** 所有相对路径基于 `/Users/liushan/Documents/zkw/mini_schedule/web`。

---

### Task 1: 搭建 `packages/core` 包 + Vitest 基建

**Files:**
- Create: `packages/core/package.json`
- Create: `packages/core/tsconfig.json`
- Create: `packages/core/vitest.config.ts`
- Create: `packages/core/src/smoke.test.ts`

- [ ] **Step 1: 写包定义 `packages/core/package.json`**

```json
{
  "name": "@mini-schedule/core",
  "version": "0.0.1",
  "private": true,
  "exports": {
    "./notifications": "./src/notifications.ts",
    "./ports": "./src/ports.ts"
  },
  "scripts": {
    "test": "vitest run"
  },
  "devDependencies": {
    "typescript": "^5.7.0",
    "vitest": "^2.1.9"
  }
}
```

- [ ] **Step 2: 写 `packages/core/tsconfig.json`（沿用 packages/api 范式）**

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

- [ ] **Step 3: 写 `packages/core/vitest.config.ts`（node 环境，纯逻辑无需 DOM）**

```ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    environment: 'node',
    include: ['src/**/*.test.ts'],
  },
})
```

- [ ] **Step 4: 写冒烟测试 `packages/core/src/smoke.test.ts`（先证明测试跑得起来）**

```ts
import { describe, it, expect } from 'vitest'

describe('core smoke', () => {
  it('vitest runs', () => {
    expect(1 + 1).toBe(2)
  })
})
```

- [ ] **Step 5: 安装依赖（引入 vitest）**

Run: `pnpm install`
Expected: 安装成功，`@mini-schedule/core` 作为 workspace 包被识别；vitest 出现在 core 的依赖树。

- [ ] **Step 6: 跑冒烟测试验证基建**

Run: `pnpm --filter @mini-schedule/core test`
Expected: PASS，1 个测试通过（`core smoke > vitest runs`）。

- [ ] **Step 7: 提交**

```bash
cd /Users/liushan/Documents/zkw/mini_schedule/web
git add packages/core/package.json packages/core/tsconfig.json packages/core/vitest.config.ts packages/core/src/smoke.test.ts pnpm-lock.yaml
git commit -m "chore(core): scaffold @mini-schedule/core package + vitest (P0 task 1)

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 2: 迁移 notification-format 到 core（TDD）

将 `apps/brand/lib/notification-format.ts` 的纯逻辑迁入 core。**先写测试**，再迁代码。行为与现有完全一致（零回归）。

**Files:**
- Create: `packages/core/src/notifications.test.ts`
- Create: `packages/core/src/notifications.ts`

- [ ] **Step 1: 写失败测试 `packages/core/src/notifications.test.ts`**

```ts
import { describe, it, expect } from 'vitest'
import {
  NOTIFICATION_EVENT_LABELS,
  notificationEventLabel,
  formatNotificationTime,
} from './notifications'

describe('notificationEventLabel', () => {
  it('maps known event types to Chinese labels', () => {
    expect(notificationEventLabel('booking_created')).toBe('新预约')
    expect(notificationEventLabel('subscription_expiring')).toBe('订阅即将到期')
  })
  it('falls back to the raw event type when unknown', () => {
    expect(notificationEventLabel('mystery_event')).toBe('mystery_event')
  })
  it('exposes all 8 current event labels', () => {
    expect(Object.keys(NOTIFICATION_EVENT_LABELS)).toHaveLength(8)
  })
})

describe('formatNotificationTime', () => {
  it('passes through an unparseable string unchanged', () => {
    expect(formatNotificationTime('not-a-date')).toBe('not-a-date')
  })
  it('renders a non-empty string for a valid ISO time', () => {
    const out = formatNotificationTime('2026-07-13T12:00:00Z')
    expect(out.length).toBeGreaterThan(0)
    expect(out).toContain('13') // 日 = 13，跨运行时稳定的不变量
  })
})
```

- [ ] **Step 2: 跑测试确认失败**

Run: `pnpm --filter @mini-schedule/core test`
Expected: FAIL，报 `Cannot find module './notifications'` 或导出未定义。

- [ ] **Step 3: 写实现 `packages/core/src/notifications.ts`（逻辑与现有一字不差，保证零回归）**

```ts
// 站内通知事件类型 → 中文标签 + 时间格式化。无 UI/平台依赖，供全端复用。
// 注：formatNotificationTime 目前用 toLocaleString，与现有 Web 行为一致（零回归）。
// 小程序引擎 Intl 支持弱 —— 若 P1 (Taro) 发现格式异常，再改为手写补零格式（见蓝图风险表）。

export const NOTIFICATION_EVENT_LABELS: Record<string, string> = {
  booking_created: '新预约',
  booking_cancelled: '预约取消',
  waitlist_changed: '候补变化',
  attendance_pending_noshow: '待确认爽约',
  session_cancelled: '场次取消',
  quota_near_limit: '额度已达上限',
  subscription_abnormal: '订阅受限',
  subscription_expiring: '订阅即将到期',
}

export function notificationEventLabel(eventType: string): string {
  return NOTIFICATION_EVENT_LABELS[eventType] ?? eventType
}

export function formatNotificationTime(iso: string): string {
  const d = new Date(iso)
  if (Number.isNaN(d.getTime())) return iso
  return d.toLocaleString('zh-CN', {
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
  })
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `pnpm --filter @mini-schedule/core test`
Expected: PASS，全部测试通过（smoke + notifications）。

- [ ] **Step 5: 提交**

```bash
cd /Users/liushan/Documents/zkw/mini_schedule/web
git add packages/core/src/notifications.ts packages/core/src/notifications.test.ts
git commit -m "feat(core): migrate notification labels + time formatting into core (P0 task 2)

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 3: 定义依赖注入接口 (ports) + 一致性测试

定义平台差异的注入契约。**异步接口**（Promise）取 RN AsyncStorage / 小程序 Storage 的最低公约数；Web 的同步 localStorage 用 `Promise.resolve` 包一层即可满足。P0 只定义接口 + 用内存实现证明可实现（未来各端注入真实现）。

**Files:**
- Create: `packages/core/src/ports.ts`
- Create: `packages/core/src/ports.test.ts`

- [ ] **Step 1: 写失败测试 `packages/core/src/ports.test.ts`（内存适配器证明接口可实现）**

```ts
import { describe, it, expect } from 'vitest'
import type { KeyValueStorage, AuthTokenStore, HttpClient } from './ports'

// 内存实现：证明接口可被真实实现（未来 Web/RN/小程序各注入自己的）。
class InMemoryStorage implements KeyValueStorage {
  private m = new Map<string, string>()
  async get(k: string) {
    return this.m.has(k) ? (this.m.get(k) as string) : null
  }
  async set(k: string, v: string) {
    this.m.set(k, v)
  }
  async remove(k: string) {
    this.m.delete(k)
  }
}

// AuthTokenStore 可由 KeyValueStorage 组合实现（各端复用同一套逻辑）。
function makeAuthTokenStore(storage: KeyValueStorage, key = 'access-token'): AuthTokenStore {
  return {
    getAccessToken: () => storage.get(key),
    setAccessToken: async (token) => {
      if (token === null) await storage.remove(key)
      else await storage.set(key, token)
    },
  }
}

describe('ports', () => {
  it('KeyValueStorage round-trips a value', async () => {
    const s = new InMemoryStorage()
    await s.set('k', 'v')
    expect(await s.get('k')).toBe('v')
    await s.remove('k')
    expect(await s.get('k')).toBeNull()
  })

  it('AuthTokenStore stores and clears a token over KeyValueStorage', async () => {
    const auth = makeAuthTokenStore(new InMemoryStorage())
    expect(await auth.getAccessToken()).toBeNull()
    await auth.setAccessToken('jwt-abc')
    expect(await auth.getAccessToken()).toBe('jwt-abc')
    await auth.setAccessToken(null)
    expect(await auth.getAccessToken()).toBeNull()
  })

  it('HttpClient interface is structurally usable', async () => {
    const fake: HttpClient = {
      request: async <T>() => ({ code: 'OK', message: 'ok', data: null as T }),
    }
    const res = await fake.request<{ id: number } | null>({ method: 'GET', path: '/x' })
    expect(res.code).toBe('OK')
  })
})
```

- [ ] **Step 2: 跑测试确认失败**

Run: `pnpm --filter @mini-schedule/core test`
Expected: FAIL，报 `Cannot find module './ports'`。

- [ ] **Step 3: 写接口 `packages/core/src/ports.ts`**

```ts
// 平台差异注入契约（Batch 21 蓝图 P0）。core 不 import 任何平台 API；
// 各端（Web/RN/小程序/桌面）启动时注入自己的实现。异步取 RN/小程序 Storage 的最低公约数。

export interface KeyValueStorage {
  get(key: string): Promise<string | null>
  set(key: string, value: string): Promise<void>
  remove(key: string): Promise<void>
}

// 鉴权 token 存取。可用 KeyValueStorage 组合实现（见 ports.test.ts）。
export interface AuthTokenStore {
  getAccessToken(): Promise<string | null>
  setAccessToken(token: string | null): Promise<void>
}

// 统一响应封装，对齐后端 { code, message, data }。
export interface ApiEnvelope<T> {
  code: string
  message: string
  data: T
}

export interface HttpRequest {
  method: 'GET' | 'POST' | 'PUT' | 'DELETE'
  path: string
  body?: unknown
  headers?: Record<string, string>
}

// 网络请求注入契约。Web 用 fetch、RN 用 fetch、小程序用 wx.request，各端各实现。
export interface HttpClient {
  request<T>(req: HttpRequest): Promise<ApiEnvelope<T>>
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `pnpm --filter @mini-schedule/core test`
Expected: PASS，全部通过（smoke + notifications + ports）。

- [ ] **Step 5: 提交**

```bash
cd /Users/liushan/Documents/zkw/mini_schedule/web
git add packages/core/src/ports.ts packages/core/src/ports.test.ts
git commit -m "feat(core): define Storage/Auth/HttpClient DI ports + conformance test (P0 task 3)

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 4: brand 改为消费 core，删除本地副本（零回归）

把 brand 的两个消费者从本地 `@/lib/notification-format` 切到 `@mini-schedule/core/notifications`，删除本地文件。行为不变。

**Files:**
- Modify: `apps/brand/app/(protected)/notifications/page.tsx:24`
- Modify: `apps/brand/app/(protected)/dashboard/page.tsx:16`
- Modify: `apps/brand/package.json`（加 core 依赖）
- Delete: `apps/brand/lib/notification-format.ts`

- [ ] **Step 1: 给 brand 加 core 依赖**

在 `apps/brand/package.json` 的 `dependencies` 中加一行（与 `@mini-schedule/api` 同风格）：

```json
"@mini-schedule/core": "workspace:*",
```

- [ ] **Step 2: 改 `apps/brand/app/(protected)/notifications/page.tsx` 第 21-24 行的 import**

将：
```ts
import {
  notificationEventLabel,
  formatNotificationTime,
} from '@/lib/notification-format'
```
改为：
```ts
import {
  notificationEventLabel,
  formatNotificationTime,
} from '@mini-schedule/core/notifications'
```

- [ ] **Step 3: 改 `apps/brand/app/(protected)/dashboard/page.tsx` 第 13-16 行的 import**

将：
```ts
import {
  notificationEventLabel,
  formatNotificationTime,
} from '@/lib/notification-format'
```
改为：
```ts
import {
  notificationEventLabel,
  formatNotificationTime,
} from '@mini-schedule/core/notifications'
```

- [ ] **Step 4: 删除本地副本**

Run: `git rm "apps/brand/lib/notification-format.ts"`
Expected: 文件被删除并 staged。

- [ ] **Step 5: 安装（让 workspace 依赖生效）**

Run: `pnpm install`
Expected: brand 的 node_modules 出现 `@mini-schedule/core` 软链。

- [ ] **Step 6: 确认无残留引用**

Run: `grep -rn "lib/notification-format" apps/brand/app apps/brand/components`
Expected: 无输出（无残留引用）。

- [ ] **Step 7: 提交**

```bash
cd /Users/liushan/Documents/zkw/mini_schedule/web
git add "apps/brand/package.json" "apps/brand/app/(protected)/notifications/page.tsx" "apps/brand/app/(protected)/dashboard/page.tsx" pnpm-lock.yaml
git commit -m "refactor(brand): consume notification helpers from @mini-schedule/core (P0 task 4)

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 5: 零回归验证 + 收尾

- [ ] **Step 1: 跑 core 全部单测**

Run: `pnpm --filter @mini-schedule/core test`
Expected: PASS，3 个测试文件全绿（smoke + notifications + ports）。

- [ ] **Step 2: 构建 brand 证明零回归**

Run: `pnpm --filter @mini-schedule/brand build`
Expected: 构建成功；`/notifications` 与 `/dashboard` 路由正常编译（无 module-not-found、无类型错误）。

- [ ] **Step 3: 构建 admin 证明未波及**

Run: `pnpm --filter @mini-schedule/admin build`
Expected: 构建成功（admin 不消费该逻辑，应完全不受影响）。

- [ ] **Step 4: （若前两步已各自 commit，无新增改动则跳过）最终确认工作树干净**

Run: `git status --short`
Expected: 无未提交改动（除 `.next` 等已忽略产物）。

---

## Self-Review（写完计划后的自查结果）

**Spec 覆盖**（对照蓝图第 5 节 P0 定义「抽 packages/core + 定义 Storage/Http/Auth 注入接口」）：
- 抽 `packages/core` → Task 1（含 Vitest 基建）。
- 收编纯业务逻辑（首块）→ Task 2（notification-format）。
- 定义 Storage/Http/Auth 注入接口 → Task 3。
- 改现有消费者用 core → Task 4。
- 零回归验收 → Task 5。
- **范围说明**：蓝图 P0 提到「收编分散在 brand/app 的业务逻辑（金额时间、zod、权限码常量…）」。本 plan 只迁 notification-format 作**模式验证**（稳 > 快，控制单次爆炸半径）；其余纯逻辑迁移按同一 TDD 模式在后续独立 plan 迭代，不在本 plan 一次搬完（降低回归风险）。

**Placeholder 扫描**：无 TBD/TODO；每个 code step 均含完整代码与确切命令/预期。

**类型一致性**：`KeyValueStorage`/`AuthTokenStore`/`HttpClient`/`ApiEnvelope`/`HttpRequest` 在 ports.ts 定义、在 ports.test.ts 使用，签名一致；`notificationEventLabel`/`formatNotificationTime`/`NOTIFICATION_EVENT_LABELS` 在 notifications.ts 与其测试、及 brand 两个消费者中名称一致。

**已知取舍（非缺陷，已在代码注释与蓝图风险表记录）**：`formatNotificationTime` 沿用 `toLocaleString` 以保证 Web 零回归；小程序 Intl 支持弱，留待 P1 (Taro) 真实验证时再决定是否改手写格式。

---

## 执行记录（2026-07-13，Subagent-Driven，每 task 独立 subagent + 主线程复核）

| Task | Commit (web dev) | 验证 |
|---|---|---|
| 1 scaffold + Vitest | `9e64833` | smoke 1 test PASS |
| 2 notifications 迁移 | `1239e4e` | 6 tests PASS（红→绿确认） |
| 3 DI ports | `e9dc469` | 9 tests PASS + `tsc --noEmit` 干净 |
| 4 brand 切换消费 | `6f930de` | grep 无残留引用；本地副本已删 |
| 5 零回归验收 | —（纯验证） | core 9 tests 绿；brand build ✓(30 页含 /notifications /dashboard)；admin build ✓(13 页)；工作树干净 |
| 审查修正 | 见下节 | ports 补 PATCH 后全绿 + tsc 干净 |

## Fable 5 复盘审查（2026-07-13）

**已修正（代码缺陷）**：`ports.ts` 的 `HttpRequest.method` 缺 `'PATCH'`——后端/web 客户端已大量使用 PATCH（订阅状态、套餐启停等），未来各端实现无法覆盖既有 API 面。已补一行并复验全绿。

**对后续 plan 的方法论修正（沿用时注意）**：
1. **type-only 模块的 TDD 判据**：Task 3 的「红」在运行时不成立——`import type` 被 esbuild 擦除、vitest 不做类型检查。纯接口任务的红/绿应以 `tsc --noEmit` 为判据（本次执行 agent 已正确处置；后续含纯类型的 task 直接把 tsc 写进步骤）。
2. **时区脆弱断言**：`formatNotificationTime` 的 `toContain('13')` 依赖运行机时区（Asia/Shanghai 下稳定）。未来上 CI 时在 `vitest.config.ts` 加 `test.env.TZ = 'Asia/Shanghai'` 固定。
3. **`smoke.test.ts` 已完成使命**（基建验证），下一个 core plan 顺手删除，不值得单独提交。
4. **zod 未随 P0 引入**是正确的 YAGNI——本批无 schema 迁移；首个 schema 迁移的 plan 再加依赖。
5. **根 `pnpm test` 已自动覆盖 core**：turbo 对无 build script 的包把 `dependsOn: build` 视为 no-op，core 的 vitest 正常纳入，无需额外接线。

**升级到蓝图的事项**（已写入 spec §6 风险表 + §10 修订记录）：Tauri 加载模式定为远程 URL、`packages/api` web 耦合（sonner/localStorage）的演进路径、小程序/公众号登录后端前置、`HttpClient` 错误语义为 P1 spec 首个决策点、Taro/Expo 版本锚定、P2 收窄为 brand-only + 归档 electron 仓。
