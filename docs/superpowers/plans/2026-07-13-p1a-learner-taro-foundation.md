# P1a 学员端 Taro 基建 + 登录 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 建立 `apps/learner-taro`（Taro 4 双目标 weapp+h5 可构建），core 长出 errors/client/Notifier/学员域 auth+profile（全 TDD），Taro 三端口适配器 + 登录页 + 首页（mock 登录跑通、展示昵称）。

**Architecture:** core 保持无头（纯函数 + 端口），`createApiClient` 在 `code !== 'OK'` 时 throw `CoreApiError`（P1 spec 决策 1）；Taro 端注入 `Taro.request/Storage/showToast` 三实现。weapp 直连 `http://localhost:8082`（devtools 关 urlCheck）；h5 dev 用 devServer proxy 保持同源（**零后端改动**）。

**Tech Stack:** Taro 4.x（webpack5 runner, React 18, TS）、@tanstack/react-query（仅 app 内薄封，不进 core）、Vitest（core 已有）。

**约束:** 稳 > 快；每 task 独立提交可验收；web/brand/admin 零回归。上游：spec `2026-07-13-p1-learner-taro-design.md`；后端契约已核实（`POST /api/v1/app/auth/wechat-login` body `{brand_id,code,nickname?}` → `{access_token,refresh_token,user{id,brand_id,nickname,avatar_url,vip_level},is_new_user}`；`GET /api/v1/app/profile`）。

**工作目录:** 所有相对路径基于 `/Users/liushan/Documents/zkw/mini_schedule/web`。提交在 `dev` 分支，不 push。

---

### Task 1: core errors + Notifier 端口 + createAuthTokenStore（TDD）

**Files:**
- Create: `packages/core/src/errors.ts`
- Create: `packages/core/src/errors.test.ts`
- Modify: `packages/core/src/ports.ts`（加 Notifier 接口 + createAuthTokenStore 工厂）
- Modify: `packages/core/src/ports.test.ts`（改用 core 工厂，删本地 makeAuthTokenStore）
- Modify: `packages/core/package.json`（exports 加 `./errors`）

- [ ] **Step 1: 写失败测试 `packages/core/src/errors.test.ts`**

```ts
import { describe, it, expect } from 'vitest'
import { CoreApiError, isCoreApiError } from './errors'

describe('CoreApiError', () => {
  it('carries code/message/details and is an Error', () => {
    const e = new CoreApiError('QUOTA_EXCEEDED', '已达套餐上限', { current: 3, max: 3 })
    expect(e).toBeInstanceOf(Error)
    expect(e.name).toBe('CoreApiError')
    expect(e.code).toBe('QUOTA_EXCEEDED')
    expect(e.message).toBe('已达套餐上限')
    expect(e.details).toEqual({ current: 3, max: 3 })
  })
  it('isCoreApiError narrows correctly', () => {
    expect(isCoreApiError(new CoreApiError('X', 'x'))).toBe(true)
    expect(isCoreApiError(new Error('x'))).toBe(false)
    expect(isCoreApiError(null)).toBe(false)
  })
})
```

- [ ] **Step 2: 跑测试确认失败**

Run: `pnpm --filter @mini-schedule/core test`
Expected: FAIL（Cannot find module './errors'）。

- [ ] **Step 3: 写 `packages/core/src/errors.ts`**

```ts
// 全端统一的类型化 API 错误（P1 决策 1：code!=='OK' 即 throw；toast 由各端 Notifier 决定）。

export class CoreApiError extends Error {
  readonly code: string
  readonly details?: unknown

  constructor(code: string, message: string, details?: unknown) {
    super(message)
    this.name = 'CoreApiError'
    this.code = code
    this.details = details
  }
}

export function isCoreApiError(e: unknown): e is CoreApiError {
  return e instanceof CoreApiError
}
```

- [ ] **Step 4: 在 `packages/core/src/ports.ts` 末尾追加 Notifier + createAuthTokenStore**

```ts
// 轻提示端口。Web=sonner、Taro=Taro.showToast、RN=Toast 库，各端各实现。
export interface Notifier {
  showToast(message: string, kind?: 'success' | 'error' | 'info'): void
}

// AuthTokenStore 的通用组合实现（各端只需提供 KeyValueStorage）。
export function createAuthTokenStore(storage: KeyValueStorage, key = 'ms-access-token'): AuthTokenStore {
  return {
    getAccessToken: () => storage.get(key),
    setAccessToken: async (token) => {
      if (token === null) await storage.remove(key)
      else await storage.set(key, token)
    },
  }
}
```

- [ ] **Step 5: 重构 `packages/core/src/ports.test.ts`**——删除本地 `makeAuthTokenStore` 函数，改为 `import { createAuthTokenStore } from './ports'`（连同 type import 合并），测试体内 `makeAuthTokenStore(` 两处调用改为 `createAuthTokenStore(`。断言不变。

- [ ] **Step 6: `packages/core/package.json` 的 exports 增加一行**

```json
"./errors": "./src/errors.ts",
```

- [ ] **Step 7: 跑测试 + 类型检查确认通过**（含纯类型改动，判据用 tsc——P0 复盘方法论 1）

Run: `pnpm --filter @mini-schedule/core test && npx tsc --noEmit -p packages/core/tsconfig.json`
Expected: 全部 PASS（4 文件 ≥11 tests）+ tsc 无输出。

- [ ] **Step 8: 提交**

```bash
git add packages/core/src/errors.ts packages/core/src/errors.test.ts packages/core/src/ports.ts packages/core/src/ports.test.ts packages/core/package.json
git commit -m "feat(core): CoreApiError + Notifier port + createAuthTokenStore (P1a task 1)

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 2: core createApiClient（TDD）

**Files:**
- Create: `packages/core/src/client.test.ts`
- Create: `packages/core/src/client.ts`
- Modify: `packages/core/package.json`（exports 加 `./client`）

- [ ] **Step 1: 写失败测试 `packages/core/src/client.test.ts`**

```ts
import { describe, it, expect } from 'vitest'
import { createApiClient } from './client'
import { isCoreApiError } from './errors'
import type { ApiEnvelope, HttpClient, HttpRequest } from './ports'

function fakeHttp(handler: (req: HttpRequest) => ApiEnvelope<unknown>): { http: HttpClient; calls: HttpRequest[] } {
  const calls: HttpRequest[] = []
  return {
    calls,
    http: {
      request: async <T>(req: HttpRequest) => {
        calls.push(req)
        return handler(req) as ApiEnvelope<T>
      },
    },
  }
}

describe('createApiClient', () => {
  it('returns data directly on code OK and maps method/path/body', async () => {
    const { http, calls } = fakeHttp(() => ({ code: 'OK', message: 'ok', data: { id: 7 } }))
    const api = createApiClient(http)
    const out = await api.post<{ id: number }>('/api/v1/app/x', { a: 1 })
    expect(out).toEqual({ id: 7 })
    expect(calls[0]).toMatchObject({ method: 'POST', path: '/api/v1/app/x', body: { a: 1 } })
  })

  it('throws CoreApiError with envelope code/message/details on business error', async () => {
    const { http } = fakeHttp(() => ({ code: 'UNAUTHORIZED', message: '请先登录', data: { hint: 1 } }))
    const api = createApiClient(http)
    const err = await api.get('/x').catch((e) => e)
    expect(isCoreApiError(err)).toBe(true)
    expect(err.code).toBe('UNAUTHORIZED')
    expect(err.message).toBe('请先登录')
    expect(err.details).toEqual({ hint: 1 })
  })

  it('throws MALFORMED_RESPONSE when envelope shape is broken', async () => {
    const { http } = fakeHttp(() => ({}) as ApiEnvelope<unknown>)
    const api = createApiClient(http)
    const err = await api.get('/x').catch((e) => e)
    expect(isCoreApiError(err)).toBe(true)
    expect(err.code).toBe('MALFORMED_RESPONSE')
  })

  it('propagates transport-layer throws untouched', async () => {
    const boom = new Error('socket hang up')
    const http: HttpClient = { request: async () => { throw boom } }
    const api = createApiClient(http)
    await expect(api.delete('/x')).rejects.toBe(boom)
  })

  it('exposes all five verbs incl. PATCH', async () => {
    const { http, calls } = fakeHttp(() => ({ code: 'OK', message: '', data: null }))
    const api = createApiClient(http)
    await api.get('/g'); await api.post('/p'); await api.put('/u'); await api.patch('/pa'); await api.delete('/d')
    expect(calls.map((c) => c.method)).toEqual(['GET', 'POST', 'PUT', 'PATCH', 'DELETE'])
  })
})
```

- [ ] **Step 2: 跑测试确认失败**

Run: `pnpm --filter @mini-schedule/core test`
Expected: FAIL（Cannot find module './client'）。

- [ ] **Step 3: 写 `packages/core/src/client.ts`**

```ts
// 无头 API 客户端：包一层注入的 HttpClient，统一 envelope 解析与错误语义。
// code==='OK' → 返回 data；否则 throw CoreApiError（details = envelope.data，
// 对齐后端把 AppError.Details 序列化进 Response.Data 的惯例）。

import { CoreApiError } from './errors'
import type { HttpClient, HttpRequest } from './ports'

export interface ApiClient {
  get<T>(path: string, headers?: Record<string, string>): Promise<T>
  post<T>(path: string, body?: unknown): Promise<T>
  put<T>(path: string, body?: unknown): Promise<T>
  patch<T>(path: string, body?: unknown): Promise<T>
  delete<T>(path: string): Promise<T>
}

export function createApiClient(http: HttpClient): ApiClient {
  async function run<T>(req: HttpRequest): Promise<T> {
    const envelope = await http.request<T>(req)
    if (!envelope || typeof envelope.code !== 'string') {
      throw new CoreApiError('MALFORMED_RESPONSE', '响应格式异常')
    }
    if (envelope.code !== 'OK') {
      throw new CoreApiError(envelope.code, envelope.message || '请求失败', envelope.data)
    }
    return envelope.data
  }

  return {
    get: (path, headers) => run({ method: 'GET', path, headers }),
    post: (path, body) => run({ method: 'POST', path, body }),
    put: (path, body) => run({ method: 'PUT', path, body }),
    patch: (path, body) => run({ method: 'PATCH', path, body }),
    delete: (path) => run({ method: 'DELETE', path }),
  }
}
```

- [ ] **Step 4: exports 增加 `"./client": "./src/client.ts",`；跑测试 + tsc**

Run: `pnpm --filter @mini-schedule/core test && npx tsc --noEmit -p packages/core/tsconfig.json`
Expected: 全 PASS + tsc 干净。

- [ ] **Step 5: 提交**

```bash
git add packages/core/src/client.ts packages/core/src/client.test.ts packages/core/package.json
git commit -m "feat(core): headless createApiClient with typed error semantics (P1a task 2)

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 3: core 学员域 auth + profile（TDD）

**Files:**
- Create: `packages/core/src/app/auth.ts`、`packages/core/src/app/learner.ts`、`packages/core/src/app/index.ts`
- Create: `packages/core/src/app/app.test.ts`
- Modify: `packages/core/package.json`（exports 加 `./app`）

- [ ] **Step 1: 写失败测试 `packages/core/src/app/app.test.ts`**

```ts
import { describe, it, expect } from 'vitest'
import { createApiClient } from '../client'
import type { ApiEnvelope, HttpClient, HttpRequest } from '../ports'
import { wechatLogin } from './auth'
import { getMyProfile } from './learner'

function clientWith(data: unknown) {
  const calls: HttpRequest[] = []
  const http: HttpClient = {
    request: async <T>(req: HttpRequest) => {
      calls.push(req)
      return { code: 'OK', message: 'ok', data } as ApiEnvelope<T>
    },
  }
  return { api: createApiClient(http), calls }
}

describe('app/auth', () => {
  it('wechatLogin posts snake_case body and returns login result', async () => {
    const result = {
      access_token: 'at', refresh_token: 'rt', is_new_user: true,
      user: { id: 1, brand_id: 21, nickname: '小明', avatar_url: '', vip_level: 'normal' },
    }
    const { api, calls } = clientWith(result)
    const out = await wechatLogin(api, { brandId: 21, code: 'dev-abc', nickname: '小明' })
    expect(out).toEqual(result)
    expect(calls[0]).toMatchObject({
      method: 'POST',
      path: '/api/v1/app/auth/wechat-login',
      body: { brand_id: 21, code: 'dev-abc', nickname: '小明' },
    })
  })
  it('wechatLogin omits nickname when absent', async () => {
    const { api, calls } = clientWith({ access_token: 'a', refresh_token: 'r', is_new_user: false, user: null })
    await wechatLogin(api, { brandId: 21, code: 'c' })
    expect(calls[0].body).toEqual({ brand_id: 21, code: 'c' })
  })
})

describe('app/learner', () => {
  it('getMyProfile GETs /api/v1/app/profile', async () => {
    const profile = { id: 9, nickname: '小明', avatar_url: null, vip_level: 'normal', phone: null }
    const { api, calls } = clientWith(profile)
    expect(await getMyProfile(api)).toEqual(profile)
    expect(calls[0]).toMatchObject({ method: 'GET', path: '/api/v1/app/profile' })
  })
})
```

- [ ] **Step 2: 跑测试确认失败**

Run: `pnpm --filter @mini-schedule/core test`
Expected: FAIL（找不到 ./auth 或 ./learner）。

- [ ] **Step 3: 写 `packages/core/src/app/auth.ts`**（契约对齐后端 `WechatLoginRequest/Response`）

```ts
// 学员端登录（当前后端为 mock：openid = "dev_"+code；换真 code2session 时本函数零改动）。

import type { ApiClient } from '../client'

export interface WechatLoginInput {
  brandId: number
  code: string
  nickname?: string
}

export interface AppUserInfo {
  id: number
  brand_id: number
  nickname: string
  avatar_url: string
  vip_level: string
}

export interface WechatLoginResult {
  access_token: string
  refresh_token: string
  user: AppUserInfo | null
  is_new_user: boolean
}

export function wechatLogin(client: ApiClient, input: WechatLoginInput): Promise<WechatLoginResult> {
  return client.post('/api/v1/app/auth/wechat-login', {
    brand_id: input.brandId,
    code: input.code,
    ...(input.nickname ? { nickname: input.nickname } : {}),
  })
}
```

- [ ] **Step 4: 写 `packages/core/src/app/learner.ts`**

```ts
import type { ApiClient } from '../client'

export interface AppProfile {
  id: number
  nickname: string
  avatar_url: string | null
  vip_level: string
  phone: string | null
}

export function getMyProfile(client: ApiClient): Promise<AppProfile> {
  return client.get('/api/v1/app/profile')
}
```

- [ ] **Step 5: 写 barrel `packages/core/src/app/index.ts`**

```ts
export * from './auth'
export * from './learner'
```

- [ ] **Step 6: exports 增加 `"./app": "./src/app/index.ts",`；跑测试 + tsc**

Run: `pnpm --filter @mini-schedule/core test && npx tsc --noEmit -p packages/core/tsconfig.json`
Expected: 全 PASS + tsc 干净。

- [ ] **Step 7: 提交**

```bash
git add packages/core/src/app packages/core/package.json
git commit -m "feat(core): learner-domain auth + profile functions (P1a task 3)

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 4: 脚手架 `apps/learner-taro`（weapp + h5 双目标构建通过）

> 这是 monorepo 集成风险最高的 task：先让**空工程**双目标构建通过，再写业务（spec 风险表）。

**Files:**
- Create: `apps/learner-taro/package.json`、`tsconfig.json`、`babel.config.js`、`project.config.json`、`.env.development`、`.env.production`
- Create: `apps/learner-taro/config/index.ts`
- Create: `apps/learner-taro/src/index.html`、`src/app.tsx`、`src/app.css`、`src/app.config.ts`
- Create: `apps/learner-taro/src/pages/home/index.tsx`、`index.config.ts`、`index.css`

- [ ] **Step 1: 确认 Taro 4 最新版本号**

Run: `npm view @tarojs/cli version`
Expected: 输出一个 `4.x.y` 版本号。**下述所有 `@tarojs/*` 与 `babel-preset-taro` 的 `"4.1.4"` 均替换为该实际版本号**（全部锁死同一版本，不带 `^`）。

- [ ] **Step 2: 写 `apps/learner-taro/package.json`**

```json
{
  "name": "@mini-schedule/learner-taro",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "dev:weapp": "taro build --type weapp --watch",
    "build:weapp": "taro build --type weapp",
    "dev:h5": "taro build --type h5 --watch",
    "build:h5": "taro build --type h5"
  },
  "dependencies": {
    "@mini-schedule/core": "workspace:*",
    "@tanstack/react-query": "^5.62.0",
    "@tarojs/components": "4.1.4",
    "@tarojs/helper": "4.1.4",
    "@tarojs/plugin-framework-react": "4.1.4",
    "@tarojs/plugin-platform-h5": "4.1.4",
    "@tarojs/plugin-platform-weapp": "4.1.4",
    "@tarojs/react": "4.1.4",
    "@tarojs/runtime": "4.1.4",
    "@tarojs/shared": "4.1.4",
    "@tarojs/taro": "4.1.4",
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "@tarojs/cli": "4.1.4",
    "@tarojs/webpack5-runner": "4.1.4",
    "@types/react": "^18.3.0",
    "babel-preset-taro": "4.1.4",
    "typescript": "^5.7.0"
  }
}
```

- [ ] **Step 3: 写 `apps/learner-taro/babel.config.js`**

```js
module.exports = {
  presets: [['taro', { framework: 'react', ts: true }]],
}
```

- [ ] **Step 4: 写 `apps/learner-taro/tsconfig.json`**（独立配置：Taro 用 react-jsx，与 Next 的 preserve 不同，不 extends base）

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true
  },
  "include": ["src", "config"]
}
```

- [ ] **Step 5: 写 `apps/learner-taro/config/index.ts`**（关键：`compile.include` 把 workspace 的 core 纳入编译——monorepo 集成点）

```ts
import path from 'node:path'
import { defineConfig } from '@tarojs/cli'

export default defineConfig({
  projectName: 'learner-taro',
  date: '2026-7-13',
  designWidth: 750,
  deviceRatio: { 640: 2.34 / 2, 750: 1, 828: 1.81 / 2, 375: 2 },
  sourceRoot: 'src',
  outputRoot: process.env.TARO_ENV === 'h5' ? 'dist/h5' : 'dist/weapp',
  framework: 'react',
  compiler: 'webpack5',
  plugins: [],
  mini: {
    compile: {
      include: [path.resolve(__dirname, '../../../packages/core')],
    },
    postcss: {
      autoprefixer: { enable: true },
    },
  },
  h5: {
    publicPath: '/',
    staticDirectory: 'static',
    router: { mode: 'hash' },
    compile: {
      include: [path.resolve(__dirname, '../../../packages/core')],
    },
    devServer: {
      proxy: {
        '/api': { target: 'http://localhost:8082', changeOrigin: true },
      },
    },
  },
})
```

（若该 Taro 版本的 `@tarojs/cli` 不导出 `defineConfig`，去掉包装直接 `export default { ... }`，其余不变，并在报告中注明。）

- [ ] **Step 6: 写 `apps/learner-taro/project.config.json`**（devtools 导入用；touristappid + 关 urlCheck 使 localhost 请求可用）

```json
{
  "miniprogramRoot": "dist/weapp/",
  "projectname": "mini-schedule-learner",
  "appid": "touristappid",
  "compileType": "miniprogram",
  "setting": {
    "urlCheck": false,
    "es6": false,
    "postcss": false,
    "minified": false
  }
}
```

- [ ] **Step 7: 写环境变量文件（两份内容相同；生产值由后续部署批次覆盖）**

`apps/learner-taro/.env.development` 与 `apps/learner-taro/.env.production`：

```
TARO_APP_API_BASE=http://localhost:8082
TARO_APP_DEFAULT_BRAND_ID=21
```

- [ ] **Step 8: 写 h5 模板 `apps/learner-taro/src/index.html`**

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <meta content="width=device-width,initial-scale=1,user-scalable=no" name="viewport" />
  <title>课程预约</title>
</head>
<body>
  <div id="app"></div>
</body>
</html>
```

- [ ] **Step 9: 写入口 `src/app.tsx`、`src/app.css`（空文件）、`src/app.config.ts`**

`src/app.tsx`：
```tsx
import type { PropsWithChildren } from 'react'
import './app.css'

function App({ children }: PropsWithChildren) {
  return children
}

export default App
```

`src/app.config.ts`：
```ts
export default {
  pages: ['pages/home/index'],
  window: {
    navigationBarTitleText: '课程预约',
    navigationBarBackgroundColor: '#ffffff',
    navigationBarTextStyle: 'black',
  },
}
```

- [ ] **Step 10: 写占位首页**

`src/pages/home/index.tsx`：
```tsx
import { View, Text } from '@tarojs/components'
import './index.css'

export default function HomePage() {
  return (
    <View className="page">
      <Text>learner-taro 双目标构建冒烟</Text>
    </View>
  )
}
```

`src/pages/home/index.config.ts`：
```ts
export default { navigationBarTitleText: '首页' }
```

`src/pages/home/index.css`：
```css
.page { padding: 32px; }
```

- [ ] **Step 11: 安装 + 双目标构建**

Run: `pnpm install`
Run: `pnpm --filter @mini-schedule/learner-taro build:weapp`
Expected: 构建成功，产物在 `apps/learner-taro/dist/weapp/`（含 app.js/app.json）。
Run: `pnpm --filter @mini-schedule/learner-taro build:h5`
Expected: 构建成功，产物在 `apps/learner-taro/dist/h5/`（含 index.html）。
若失败：优先核对 `@tarojs/*` 版本是否一致、`compile.include` 路径是否解析到 `web/packages/core`；报告确切报错，不擅自引入额外插件。

- [ ] **Step 12: 加 `.gitignore` 后提交**

`apps/learner-taro/.gitignore`：
```
dist/
```

```bash
git add apps/learner-taro pnpm-lock.yaml
git commit -m "feat(learner-taro): scaffold Taro 4 dual-target app in monorepo (P1a task 4)

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 5: Taro 三端口适配器 + api 单例

**Files:**
- Create: `apps/learner-taro/src/adapters/http.ts`、`storage.ts`、`notifier.ts`
- Create: `apps/learner-taro/src/lib/api.ts`

- [ ] **Step 1: 写 `src/adapters/http.ts`**

```ts
import Taro from '@tarojs/taro'
import { CoreApiError } from '@mini-schedule/core/errors'
import type { ApiEnvelope, AuthTokenStore, HttpClient, HttpRequest } from '@mini-schedule/core/ports'

// HttpClient 的 Taro 实现：weapp 走原生请求（devtools 关 urlCheck），h5 走 XHR（dev 同源 proxy）。
export class TaroHttpClient implements HttpClient {
  constructor(
    private readonly baseUrl: string,
    private readonly tokenStore: AuthTokenStore,
  ) {}

  async request<T>(req: HttpRequest): Promise<ApiEnvelope<T>> {
    const token = await this.tokenStore.getAccessToken()
    try {
      const res = await Taro.request<ApiEnvelope<T>>({
        url: this.baseUrl + req.path,
        method: req.method,
        data: req.body as string | Record<string, unknown> | undefined,
        header: {
          'Content-Type': 'application/json',
          ...(token ? { Authorization: `Bearer ${token}` } : {}),
          ...req.headers,
        },
      })
      return res.data
    } catch (e) {
      throw new CoreApiError('NETWORK_ERROR', '网络请求失败，请稍后重试', e)
    }
  }
}
```

- [ ] **Step 2: 写 `src/adapters/storage.ts`**

```ts
import Taro from '@tarojs/taro'
import type { KeyValueStorage } from '@mini-schedule/core/ports'

export class TaroStorage implements KeyValueStorage {
  async get(key: string): Promise<string | null> {
    try {
      const res = await Taro.getStorage<string>({ key })
      return typeof res.data === 'string' ? res.data : null
    } catch {
      return null // key 不存在时 Taro reject
    }
  }
  async set(key: string, value: string): Promise<void> {
    await Taro.setStorage({ key, data: value })
  }
  async remove(key: string): Promise<void> {
    try {
      await Taro.removeStorage({ key })
    } catch {
      /* key 不存在视为已删除 */
    }
  }
}
```

- [ ] **Step 3: 写 `src/adapters/notifier.ts`**

```ts
import Taro from '@tarojs/taro'
import type { Notifier } from '@mini-schedule/core/ports'

export const taroNotifier: Notifier = {
  showToast(message, kind = 'info') {
    Taro.showToast({ title: message, icon: kind === 'success' ? 'success' : 'none', duration: 2000 })
  },
}
```

- [ ] **Step 4: 写 `src/lib/api.ts`**（组合端口成单例；h5 dev 走同源 proxy 故 base 为空）

```ts
import { createApiClient } from '@mini-schedule/core/client'
import { createAuthTokenStore } from '@mini-schedule/core/ports'
import { TaroHttpClient } from '../adapters/http'
import { TaroStorage } from '../adapters/storage'

const API_BASE =
  process.env.TARO_ENV === 'h5' ? '' : process.env.TARO_APP_API_BASE || 'http://localhost:8082'

export const DEFAULT_BRAND_ID = Number(process.env.TARO_APP_DEFAULT_BRAND_ID) || 1
export const REFRESH_TOKEN_KEY = 'ms-refresh-token'

export const storage = new TaroStorage()
export const authStore = createAuthTokenStore(storage)
export const api = createApiClient(new TaroHttpClient(API_BASE, authStore))
```

- [ ] **Step 5: 双目标构建验证适配器编译**

Run: `pnpm --filter @mini-schedule/learner-taro build:weapp && pnpm --filter @mini-schedule/learner-taro build:h5`
Expected: 两个目标均构建成功（适配器与 core 消费链路编译通过）。

- [ ] **Step 6: 提交**

```bash
git add apps/learner-taro/src/adapters apps/learner-taro/src/lib
git commit -m "feat(learner-taro): Taro adapters for Http/Storage/Notifier ports + api singleton (P1a task 5)

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 6: 登录页 + 首页（react-query 冒烟）

**Files:**
- Create: `src/pages/login/index.tsx`、`index.config.ts`、`index.css`
- Modify: `src/pages/home/index.tsx`（真实首页：profile 昵称）
- Modify: `src/app.tsx`（挂 QueryClientProvider）、`src/app.config.ts`（登录页为入口）

- [ ] **Step 1: 改 `src/app.tsx`**

```tsx
import type { PropsWithChildren } from 'react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import './app.css'

const queryClient = new QueryClient({
  defaultOptions: { queries: { retry: 1, refetchOnWindowFocus: false } },
})

function App({ children }: PropsWithChildren) {
  return <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
}

export default App
```

- [ ] **Step 2: 改 `src/app.config.ts`（登录页在首位 = 默认入口）**

```ts
export default {
  pages: ['pages/login/index', 'pages/home/index'],
  window: {
    navigationBarTitleText: '课程预约',
    navigationBarBackgroundColor: '#ffffff',
    navigationBarTextStyle: 'black',
  },
}
```

- [ ] **Step 3: 写 `src/pages/login/index.tsx`**

```tsx
import { useState } from 'react'
import Taro from '@tarojs/taro'
import { View, Input, Button, Text } from '@tarojs/components'
import { wechatLogin } from '@mini-schedule/core/app'
import { isCoreApiError } from '@mini-schedule/core/errors'
import { api, authStore, storage, REFRESH_TOKEN_KEY, DEFAULT_BRAND_ID } from '../../lib/api'
import { taroNotifier } from '../../adapters/notifier'
import './index.css'

export default function LoginPage() {
  const [brandId, setBrandId] = useState(String(DEFAULT_BRAND_ID))
  const [code, setCode] = useState('')
  const [submitting, setSubmitting] = useState(false)

  async function fetchWeappCode() {
    try {
      const res = await Taro.login()
      if (res.code) setCode(res.code)
      else taroNotifier.showToast('未取到 code，可手动输入', 'error')
    } catch {
      taroNotifier.showToast('获取微信 code 失败，可手动输入', 'error')
    }
  }

  async function submit() {
    const bid = Number(brandId)
    if (!bid || !code.trim()) {
      taroNotifier.showToast('请填写品牌 ID 和登录码', 'error')
      return
    }
    setSubmitting(true)
    try {
      const result = await wechatLogin(api, { brandId: bid, code: code.trim() })
      await authStore.setAccessToken(result.access_token)
      await storage.set(REFRESH_TOKEN_KEY, result.refresh_token)
      Taro.redirectTo({ url: '/pages/home/index' })
    } catch (e) {
      taroNotifier.showToast(isCoreApiError(e) ? e.message : '登录失败', 'error')
    } finally {
      setSubmitting(false)
    }
  }

  return (
    <View className="page">
      <Text className="title">学员登录</Text>
      <Input
        className="input"
        type="number"
        value={brandId}
        placeholder="品牌 ID"
        onInput={(e) => setBrandId(e.detail.value)}
      />
      <Input
        className="input"
        value={code}
        placeholder="登录码（开发期任意字符串）"
        onInput={(e) => setCode(e.detail.value)}
      />
      {process.env.TARO_ENV === 'weapp' && (
        <Button className="btn" onClick={fetchWeappCode}>
          微信获取 code
        </Button>
      )}
      <Button className="btn btn-primary" loading={submitting} onClick={submit}>
        登录
      </Button>
    </View>
  )
}
```

`src/pages/login/index.config.ts`：
```ts
export default { navigationBarTitleText: '登录' }
```

`src/pages/login/index.css`：
```css
.page { display: flex; flex-direction: column; gap: 24px; padding: 48px 32px; }
.title { font-size: 40px; font-weight: 600; }
.input { border: 1px solid #ddd; border-radius: 8px; padding: 16px; }
.btn { margin-top: 8px; }
.btn-primary { background: #10b981; color: #fff; }
```

- [ ] **Step 4: 重写 `src/pages/home/index.tsx`（react-query 冒烟 + 昵称展示）**

```tsx
import Taro from '@tarojs/taro'
import { View, Text, Button } from '@tarojs/components'
import { useQuery } from '@tanstack/react-query'
import { getMyProfile } from '@mini-schedule/core/app'
import { api, authStore } from '../../lib/api'
import './index.css'

export default function HomePage() {
  const profileQuery = useQuery({
    queryKey: ['app-profile'],
    queryFn: () => getMyProfile(api),
    retry: false,
  })

  async function logout() {
    await authStore.setAccessToken(null)
    Taro.redirectTo({ url: '/pages/login/index' })
  }

  if (profileQuery.isLoading) {
    return (
      <View className="page">
        <Text>加载中…</Text>
      </View>
    )
  }
  if (profileQuery.isError) {
    return (
      <View className="page">
        <Text>未登录或会话已过期</Text>
        <Button className="btn" onClick={() => Taro.redirectTo({ url: '/pages/login/index' })}>
          去登录
        </Button>
      </View>
    )
  }
  const p = profileQuery.data
  return (
    <View className="page">
      <Text className="title">你好，{p?.nickname || '学员'}</Text>
      <Text>VIP 等级：{p?.vip_level ?? '-'}</Text>
      <Button className="btn" onClick={logout}>
        退出登录
      </Button>
    </View>
  )
}
```

- [ ] **Step 5: 双目标构建**

Run: `pnpm --filter @mini-schedule/learner-taro build:weapp && pnpm --filter @mini-schedule/learner-taro build:h5`
Expected: 两目标构建成功（react-query 在两目标均可编译；若 weapp 目标报 window/document 引用错误，报告确切报错——这是 spec 风险表的 react-query 兜底触发条件，不擅自换库）。

- [ ] **Step 6: 提交**

```bash
git add apps/learner-taro/src
git commit -m "feat(learner-taro): login + home pages with react-query smoke (P1a task 6)

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 7: 零回归验收 + 手动验收指引

- [ ] **Step 1: core 全部单测 + tsc**

Run: `pnpm --filter @mini-schedule/core test && npx tsc --noEmit -p packages/core/tsconfig.json`
Expected: 全 PASS（5 测试文件）+ tsc 干净。

- [ ] **Step 2: web 零回归**

Run: `pnpm --filter @mini-schedule/brand build`
Expected: 构建成功。
Run: `pnpm --filter @mini-schedule/admin build`
Expected: 构建成功。

- [ ] **Step 3: 确认工作树干净（各 task 已随手提交）**

Run: `git status --short`
Expected: 无未提交改动（`dist/` 已被 learner-taro/.gitignore 忽略）。

- [ ] **Step 4: 输出手动验收指引（人工执行，非本计划自动步骤）**

```
① 起后端：cd backend && CONFIG_PATH=configs/config-app.yaml go run ./cmd/api-app/   (:8082)
② weapp：cd web && pnpm --filter @mini-schedule/learner-taro dev:weapp
   微信开发者工具 → 导入项目 → 目录选 web/apps/learner-taro（appid=touristappid）
   → 登录页输入 品牌ID=21、登录码任意字符串 → 登录成功跳首页显示昵称
③ h5：pnpm --filter @mini-schedule/learner-taro dev:h5 → 浏览器打开提示的本地地址
   → 同流程走通（请求经 devServer proxy 同源转发到 :8082，无 CORS）
④ 首页「退出登录」→ 回登录页；再直接访问首页 → 显示「未登录」兜底
```

---

## Self-Review（写完计划后的自查结果）

**Spec 覆盖**（对照 P1 spec §3 P1a 行）：monorepo 接入双目标构建（Task 4，且按风险表"空工程先行"）；core errors/client/Notifier/app-auth + 单测（Task 1-3）；三端口 Taro 实现（Task 5）；登录页+首页壳含昵称（Task 6）；验收四件套（Task 7）。react-query 冒烟（spec 风险表）在 Task 6 落实。**零后端改动**由 h5 devServer proxy 达成（spec §2.3 的 CORS 项在 dev 阶段规避，生产 H5 部署时再走 CORS/反代，属 P1c/部署批范围）。

**Placeholder 扫描**：无 TBD/TODO；唯一的"替换"是 Taro 版本号（Step 1 有确切获取命令与替换规则，非占位符）。`.env.production` 写入与 dev 相同的具体值并注明由部署批覆盖。

**类型一致性**：`CoreApiError(code, message, details)` 三处使用一致；`createAuthTokenStore` 在 Task 1 定义、Task 5 lib/api.ts 使用；`ApiClient` 动词五个（含 P0 审查补的 PATCH）与 client.test.ts 断言一致；`wechatLogin/getMyProfile` 签名在 Task 3 定义与 Task 6 页面使用一致；`storage/authStore/api/DEFAULT_BRAND_ID/REFRESH_TOKEN_KEY` 导出名在 Task 5 定义与 Task 6 import 一致。

**已知不确定点（执行时按报告规则处置，不擅自改设计）**：`@tarojs/cli` 的 `defineConfig` 导出与 `compile.include` 的确切 schema 随 4.x 小版本可能有差异——Task 4 已给降级写法与"报告确切报错"的停止条件；react-query 在 weapp 的兼容性由 Task 6 Step 5 冒烟裁决（兜底在 spec 风险表）。
