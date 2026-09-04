# 浏览器半 ↔ 主进程 Connection RPC

DSH 插件可以同时拥有两份代码：主进程半（host，挂在 `apply(ctx)` 里）和浏览器半（client，挂在 `dsh.client.inject` 的另一个 apply 里）。当浏览器半需要读写主进程状态——典型场景是设置面板增删 DSN、测连接、保存配置——主进程半必须把方法**经 Connection RPC 通道**暴露出来，而不是经 `@Remote` 装饰器。

本文件是这一通道的权威样板；以 `dsh-mcp-manager` 和 `dsh-plugin-dbhub` 的实现为基线（两者的 host 半与服务定义几乎一一对应）。

## 为什么不用 `@Remote` / `@deepseek-ai/dsh-typert-protocol`

`@Remote` 装饰器把方法标记写入 `@deepseek-ai/dsh-typert-protocol` 内部的模块级 `WeakMap`。DSH 网关读同一份 WeakMap 来发现可调方法。这要求主进程半和浏览器半加载**同一个 module 实例**——但插件 npm 分发后双方各自 `node_modules` 里各有一份副本，标记 WeakMap 不同，浏览器半调用时网关会报 "Service has no visible typertRemote binding"。

绕开的唯一办法是把协议包 `createRequire` 到 deepseek-harness monorepo 源码树的 `lib/index.js`，强行使双方加载同一文件。这条路径：

- 需要操作员克隆 deepseek-harness 源码（`DSH_HARNESS_ROOT` 或硬编码路径）
- 需要在客户端镜像同一份 manifest（手工维护两份 `invocations[]`）
- npm 用户拿不到源码就直接挂掉

**Connection RPC 没有这个问题**：channel 字符串 + endpoint 名字 + JSON payload + 信封，是 wire-level 约定，不依赖 module identity。npm 分发、双副本、pnpm workspace 隔离都不影响。

判定表：

| 想做的事 | 用的机制 |
| --- | --- |
| 同进程内被其他**主进程**插件消费 | 公开 Cordis `Service`，加 `inject` 消费方；见 references/services.md |
| 同进程内被其他**主进程**插件以事件方式响应 | `ctx.on(...)`，见 references/events.md |
| **浏览器半**调主进程半方法（设置面板/可视化编辑） | **`ctx.connection.rpc.handle` + `ctx.connection.rpc.call`**，本文件 |
| 跨进程/跨语言协议桥（ACP、stdio JSON-RPC） | ACP / JSON-RPC stdio，见 references/plugin-forms.md 末节 |

## 协议形状

```ts
// 一根命名通道下挂多个 endpoint；每个 endpoint 一份强类型载荷 + 信封返回值。
const RPC_CHANNEL = '/<your-plugin>'         // 例：'/dbhub'、'/mcp-manager'
type Endpoint = 'list' | 'save' | '...'      // endpoint 联合

// 信封：成功直接带值，失败带结构化错误；客户端按 code 渲染而不是抛字符串。
type RpcResult<T> =
  | { ok: true; value: T }
  | { ok: false; error: { code: string; message: string; details?: Record<string, unknown> } }
```

服务端**不要**把异常 throw 进 RPC 层——网关会把它们包成基础设施级错误，丢失 `code`。失败一律返回 `fail(code, message)`。

## 主进程半（host）样板

```ts
// src/index.ts
import type { Context } from '@deepseek-ai/cordis'
import { dispatch, RPC_CHANNEL } from './host/rpc.ts'
import { YourService } from './host/service.ts'

// `ctx.connection.rpc` 由 host 组合（dsh-base + dsh-web-app）挂载。
declare module '@deepseek-ai/cordis' {
  interface Context {
    connection: {
      rpc: {
        handle(
          channel: string,
          handler: (endpoint: string, payload: unknown) => Promise<unknown>,
          opts?: { authority?: string },
        ): unknown                  // 返回的 disposer 形状见下文
      }
    }
  }
}

export const name = 'your-plugin'
export const inject = ['connection']                  // 等到 connection 挂上

export async function apply(ctx: Context): Promise<void> {
  const service = new YourService(ctx)

  ctx.inject(['connection'], () => {
    ctx.effect(() => {
      const handler = async (endpoint: string, payload: unknown) =>
        dispatch(service, endpoint, payload)
      const disposer = ctx.connection.rpc.handle(
        RPC_CHANNEL,
        handler,
        { authority: 'loopback' },                   // 浏览器调用只走 loopback
      )
      // 返回的 disposer 形状跨实现不一致：可能同步函数、Promise<() => void>、
      // 或 { dispose: () => void }。统一兼容，避免卸载时报错。
      return () => {
        Promise.resolve(disposer).then((d) => {
          if (typeof d === 'function') d()
          else if (d !== null && typeof d === 'object' && 'dispose' in d) {
            (d as { dispose: () => void }).dispose()
          }
        }).catch(() => { /* best-effort cleanup */ })
      }
    }, 'your-plugin: rpc channel')
  })
}
```

```ts
// src/host/rpc.ts —— 单 switch 路由 endpoint 到 Service 方法
import type { YourService } from './service.ts'
import { RPC_CHANNEL, type Endpoint, type RpcResult } from '../shared/types.ts'

function ok<T>(value: T): RpcResult<T> { return { ok: true, value } }
function fail<T = never>(code: string, message: string): RpcResult<T> {
  return { ok: false, error: { code, message } }
}

export async function dispatch(
  service: YourService,
  endpoint: string,
  payload: unknown,
): Promise<RpcResult<unknown>> {
  switch (endpoint as Endpoint) {
    case 'list':
      return ok(await service.list())
    case 'save':
      try {
        return ok(await service.save(payload as Parameters<YourService['save']>[0]))
      } catch (err) {
        return fail('save-failed', err instanceof Error ? err.message : String(err))
      }
    default:
      return fail('unknown-endpoint', `unknown endpoint "${endpoint}"`)
  }
}

export { RPC_CHANNEL }
```

`YourService` 是普通 Cordis `Service` 子类（`extends Service`），方法上**不**加任何装饰器——这与 `@Remote` 路径的关键差别。

## 浏览器半（client）样板

```ts
// src/client/rpc.ts
import type { ClientContext } from '@deepseek-ai/dsh-client-runtime/client'
import type { ClientConnectionRpc } from '@deepseek-ai/dsh-client-connection/client'
import { RPC_CHANNEL, type Endpoint, type RpcResult } from '../shared/types.ts'

// host 端类型在 ctx.connection 上声明；浏览器端 runtime 没声明，要结构化强转。
function connectionRpcOf(ctx: ClientContext): ClientConnectionRpc {
  const connection = (ctx as unknown as { connection?: { rpc: ClientConnectionRpc } }).connection
  if (connection === undefined) {
    throw new Error('connection service is unavailable (is @deepseek-ai/dsh-client-connection loaded?)')
  }
  return connection.rpc
}

export class YourRpcError extends Error {
  readonly code: string
  constructor(error: { code: string; message: string }) {
    super(`${error.code}: ${error.message}`)
    this.name = 'YourRpcError'
    this.code = error.code
  }
}

export async function callRpc<T>(
  ctx: ClientContext,
  endpoint: Endpoint,
  payload?: unknown,
): Promise<T> {
  const raw = await connectionRpcOf(ctx).call(RPC_CHANNEL, endpoint, payload ?? null)
  const result = raw as unknown as RpcResult<T>
  if (result.ok) return result.value
  throw new YourRpcError(result.error)
}
```

面板里调用：

```tsx
const value = await callRpc<DbhubView>(ctx, 'list')
// 失败抛 YourRpcError；UI 用 code + message 渲染，不要把 message 当 raw exception 显示。
```

## 类型与依赖

| 项 | 取处 |
| --- | --- |
| `ClientContext` | `@deepseek-ai/dsh-client-runtime/client`（type-only 也行；为通过 typecheck 建议升为正式 dep） |
| `ClientConnectionRpc` | `@deepseek-ai/dsh-client-connection/client` |
| `connectionRpcOf` 内的 ctx.connection 强转 | 必有——host 端类型没在 browser runtime 暴露，是 DSH 类型层面的已知缺口 |

## 反模式（看完就要避开）

- **用 `@Remote` 装饰 Service 方法**：`dsh-typert-protocol` 是 DSH monorepo 内部包；npm 分发场景必坏。
- **手写并维护 `invocations[]` manifest**：等同于重复维护一遍类型定义；用 TypeScript 联合 + 信封就够。
- **`createRequire` 桥接到 deepseek-harness 源码树**：仅在你正开发 deepseek-harness 自身时合法，对外发布插件不要用。
- **throw 异常出 `dispatch`**：丢失 `code`，面板只能显示 "Service error: Internal error" 一类无信息消息。
- **dispatch 返回非 `RpcResult` 的对象**：客户端按 `ok`/`error` 解包，错型会让 `if (result.ok)` 抛 `TypeError`。
- **channel 名不带前导 `/`**：与 DSH 内部 `/internal/*` 等约定不一致；建议固定 `'/your-plugin'`。

## 迁移检查清单（从 `@Remote` 改造时）

- [ ] 删除 `src/typert*.ts` 与 `client/typert-remote.ts`（以及一切手写 manifest）
- [ ] `Service` 改继承 `Service`（不再继承 `TypertRemoteService`），去掉方法上的 `@Remote()`
- [ ] 把方法集合收敛成 endpoint 联合（`type Endpoint = ...`），与 `dispatch` 的 switch 对齐
- [ ] `shared/types.ts` 加 `RPC_CHANNEL` 常量、`RpcResult<T>` 信封、`Endpoint` 联合
- [ ] `host/rpc.ts` 写 `dispatch(service, endpoint, payload)` 单 switch
- [ ] `index.ts` 加 `inject: ['connection']`，`ctx.effect` 里 `handle(channel, handler, {authority:'loopback'})`
- [ ] `client/rpc.ts` 写 `callRpc<T>` + 自定义错误类
- [ ] `client/index.ts` 删掉 TYPERT_REMOTE manifest 的 mount，改成把 `ctx` 直接 inject 给面板组件
- [ ] `package.json` 去掉 `@deepseek-ai/dsh-typert-protocol`；`@deepseek-ai/dsh-client-connection` 列为 peer/devDep
- [ ] README / 包说明里把"需要 deepseek-harness 源码 checkout"的字样删掉

## 参考实现

- `@js2hou/dsh-mcp-manager` 的 `src/index.ts` + `src/client/rpc.ts`
- `@xcr1234/dsh-plugin-dbhub` 的 `src/index.ts` + `src/host/rpc.ts` + `src/client/rpc.ts`

两者的 channel 字符串、信封、错误类、disposer 兼容方式都可以直接当模板抄。