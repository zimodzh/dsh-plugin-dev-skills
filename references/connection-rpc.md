# 浏览器半 ↔ 主进程 Connection RPC

DSH 插件可以同时拥有两份代码：主进程半（host，挂在 `apply(ctx)` 里）和浏览器半（client，挂在 `dsh.client.inject` 的另一个 apply 里）。npm 分发的设置面板要读写主进程状态时，**优先**把方法经 Connection RPC 通道暴露出来。

本文件是这条通道的权威样板，以 [`Js2Hou/dsh-mcp-manager`](https://github.com/Js2Hou/dsh-mcp-manager) 的 host / client 实现为基线。

完整 Typert generator 产出、并经 `package.json` 导出 `./typert` + `./remote`、由网关 `ctx.remote.$mount()` 消费的路径**仍然合法**。禁止的是半截 Typert：裸 `@Remote`、手写 `invocations[]` manifest、`createRequire` 挂 harness 源码。

## 何时用 RPC，何时用 Typert

| 想做的事 | 用的机制 |
| --- | --- |
| 同进程内被其他**主进程**插件消费 | 公开 Cordis `Service`，加 `inject` 消费方；见 references/services.md |
| 同进程内被其他**主进程**插件以事件方式响应 | `ctx.on(...)`，见 references/events.md |
| **npm 设置面板**调主进程半方法 | **优先** `ctx.connection.rpc.handle` + `ctx.connection.rpc.call`，本文件 |
| 完整 Typert 流水线（标记 + generator + `./typert`/`./remote` + `$mount`） | 合法；见下文「合法的 Typert 路径」 |
| 跨进程/跨语言协议桥（ACP、stdio JSON-RPC） | ACP / JSON-RPC stdio，见 references/plugin-forms.md 末节 |

## 为什么裸 `@Remote` 不够

`@Remote` 装饰器把方法标记写入 `@deepseek-ai/dsh-typert-protocol` 内部的模块级 `WeakMap`。没有 generator 产物时，网关只能读这份 WeakMap——这要求主进程半和浏览器半加载**同一个 module 实例**。插件 npm 分发后双方各自 `node_modules` 各有一份副本，标记不可见，调用报 `Service has no visible typertRemote binding`。

常见的半截绕法都禁止：

- 手写并维护一份 `invocations[]` manifest（等于再维护一遍类型）
- 把协议包 `createRequire` 到 deepseek-harness 源码树的 `lib/index.js`，强行使双方加载同一文件（要 `DSH_HARNESS_ROOT`，npm 用户没有源码就挂）

**Connection RPC 没有这个问题**：channel 字符串 + endpoint 名字 + JSON payload + 信封，是 wire-level 约定，不依赖 module identity。

完整 Typert 路径不靠这份 WeakMap 当唯一真相：generator 把描述符写进 `./remote`，客户端 mount 的是产物，不是装饰器副作用。

## 协议形状

```ts
const RPC_CHANNEL = '/<your-plugin>'         // 例：'/mcp-manager'
type Endpoint = 'list' | 'save' | '...'

type RpcResult<T> =
  | { ok: true; value: T }
  | { ok: false; error: { code: string; message: string; details?: Record<string, unknown> } }
```

服务端不要把异常 throw 进 RPC 层——网关会把它们包成基础设施级错误，丢失 `code`。失败一律返回 `fail(code, message)`。

## 主进程半（host）

按 `dsh-mcp-manager`：`inject: ['connection']`，用 `import type {}` 拉进 `Context.connection` 声明合并，`ctx.effect` 里 `handle`，卸载时 `void dispose()`。

```ts
import type { Context } from '@deepseek-ai/cordis'
import type {} from '@deepseek-ai/dsh-client-connection'
import { RPC_CHANNEL, type Endpoint, type RpcResult } from './shared.ts'

export const name = 'your-plugin'
export const inject = ['connection']

function ok<T>(value: T): RpcResult<T> { return { ok: true, value } }
function fail<T = never>(code: string, message: string): RpcResult<T> {
  return { ok: false, error: { code, message } }
}

async function dispatch(endpoint: string, payload: unknown): Promise<RpcResult<unknown>> {
  switch (endpoint as Endpoint) {
    case 'list':
      return ok(await list())
    case 'save':
      try {
        return ok(await save(payload))
      } catch (err) {
        return fail('save-failed', err instanceof Error ? err.message : String(err))
      }
    default:
      return fail('unknown-endpoint', `unknown endpoint "${endpoint}"`)
  }
}

export function apply(ctx: Context): void {
  ctx.effect(() => {
    const dispose = ctx.connection.rpc.handle(
      RPC_CHANNEL,
      (endpoint, payload) => dispatch(endpoint, payload),
      { authority: 'loopback' },
    )
    return () => { void dispose() }
  }, 'your-plugin: rpc channel')
}
```

业务方法放在普通函数或 Cordis `Service` 上即可，**不要**为了 RPC 去加 `@Remote`。

## 浏览器半（client）

```ts
import type { ClientContext } from '@deepseek-ai/dsh-client-runtime/client'
import type { ClientConnectionRpc } from '@deepseek-ai/dsh-client-connection/client'
import { RPC_CHANNEL, type Endpoint, type RpcResult } from '../shared.ts'

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

`connectionRpcOf` 的结构化强转是必有的：host 端类型没在 browser runtime 暴露。面板里 `await callRpc<View>(ctx, 'list')`，失败用 `code + message` 渲染。

## 类型与依赖

| 项 | 取处 |
| --- | --- |
| `Context.connection` 声明合并 | `import type {} from '@deepseek-ai/dsh-client-connection'`（不要手写 `declare module`） |
| `ClientContext` | `@deepseek-ai/dsh-client-runtime/client` |
| `ClientConnectionRpc` | `@deepseek-ai/dsh-client-connection/client` |

## 反模式（看完就要避开）

- **裸 `@Remote` / `TypertRemoteService`，不跑 generator、不导出 `./remote`**：npm 分发场景必坏。
- **手写并维护 `invocations[]` manifest**：等同于重复维护一遍类型；RPC 用 TypeScript 联合 + 信封，Typert 用 generator 产物。
- **`createRequire` 桥接到 deepseek-harness 源码树**：对外发布插件不要用。
- **throw 异常出 `dispatch`**：丢失 `code`，面板只能显示 "Service error: Internal error"。
- **dispatch 返回非 `RpcResult` 的对象**：客户端按 `ok`/`error` 解包，错型会让 `if (result.ok)` 抛 `TypeError`。
- **channel 名不带前导 `/`**：与 DSH 内部 `/internal/*` 等约定不一致；固定 `'/your-plugin'`。

## 合法的 Typert 路径

官方五步仍然合法，不要因为设置面板优先 RPC 就整段删掉：

1. 用 `@typert object` / `@typert service <key>` / `@Remote` 标记公开面
2. 运行时 `ctx.typert.lookups.register()` / `ctx.typert.contexts.*` 登记身份
3. `WorkspaceTypertGenerator` 写出 `lib/typert.host.*` 与 `lib/typert.remote-client.*`
4. `package.json` 的 `exports` / `files` 暴露 `./typert` 与 `./remote`
5. 消费方 `ctx.remote.$mount()` 挂上产物

缺任何一步（尤其是 3–5）就退回上面的半截路径，改走 Connection RPC。

## 从半截 Typert 迁到 RPC

- [ ] 删除手写 `src/typert*.ts`、`client/typert-remote.ts`、`invocations[]`
- [ ] 去掉 `createRequire` / `DSH_HARNESS_ROOT` 源码桥
- [ ] 若没有完整 generator + `./remote`：去掉方法上的裸 `@Remote()`，改普通 `Service`
- [ ] `shared.ts` 加 `RPC_CHANNEL`、`RpcResult<T>`、`Endpoint` 联合
- [ ] host：`inject: ['connection']`，`ctx.effect` 里 `handle(channel, dispatch, {authority:'loopback'})`，`return () => { void dispose() }`
- [ ] client：`callRpc<T>` + 自定义错误类；面板直接拿 `ctx`，不要 mount 手写 manifest
- [ ] README 里删掉「需要 deepseek-harness 源码 checkout」

## 参考实现

- [`Js2Hou/dsh-mcp-manager`](https://github.com/Js2Hou/dsh-mcp-manager) 的 `src/index.ts` + `src/client/rpc.ts` + `src/shared.ts`

channel 字符串、信封、错误类、`void dispose()` 都可以直接当模板抄。
