# 服务与依赖

服务是一个插件向其他插件公开的能力。inject 声明插件需要哪些服务。

## 什么是服务

在 Harness 中 tools、llm、agents 都是服务。服务是挂载在 ctx 上的命名能力：

```ts
ctx.tools   // ToolRuntime 服务
ctx.llm     // LLM 服务
ctx.agents  // Agent 服务
```

任何插件都可以提供服务，供其他插件使用。

## 使用服务

```ts
export const inject = ['tools']
export function apply(ctx: Context) {
  // apply 执行时，inject 声明的服务已经全部就绪
  ctx.tools.register(/* ... */)
}
```

框架保证：apply 执行时 inject 声明的服务已就绪；未就绪则插件处于 PENDING 等待。cordis.yml 中各项并发启动，列表位置不保证加载先后——顺序由服务依赖（inject）决定，而非文件位置。

## 可选依赖

```ts
export function apply(ctx: Context) {
  const metrics = ctx.get('metrics')   // 不声明 inject，缺省时得到 undefined
  metrics?.record('plugin_loaded', 1)
}
```

不要用 inject 规避 undefined 检查；也不要直接访问未声明注入的 ctx.xxx——ctx 是服务解析器代理，未声明的服务求值可能得到 undefined，且其消失时插件不会被通知重载。

## 提供服务（Service 基类）

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Context { metrics: MetricsService }   // TS 声明合并给 ctx.metrics 补类型
}

export default class MetricsService extends Service {
  static inject = ['llm']                 // 服务也可以依赖其他服务
  constructor(ctx: Context) { super(ctx, 'metrics') }  // 'metrics' 是服务名
  record(event: string, value: number) { /* ... */ }
}
```

加载这个插件后，消费方即可通过 ctx.metrics 访问它：

```ts
export const inject = ['metrics']
export function apply(ctx: Context) {
  ctx.metrics.record('tool_call', 1)
}
```

子类在构造函数中调用 super(ctx, name)：服务会立即注册，并随所属 fiber 自动移除。Service 还有一组静态符号成员（init、check、config、invoke、extend、tracker、resolveConfig），细节以 Service 参考页为准。

底层替代：不经 Service 基类、直接用 ctx.provide() / ctx.accessor() / ctx.mixin() 操作服务存储，见 references/context-api.md。

## 依赖的行为

- **必需依赖消失**（提供方卸载）→ 依赖它的插件自动 dispose；服务重新出现时自动重新加载。这防止插件调用已不存在的服务。
- **可选依赖**：用 ctx.get() 按需读取并处理 undefined。

## 服务隔离（isolate realm）

cordis.yml 支持服务隔离——同一个服务可以有多个实例，不同插件组看到不同实例：

```yaml
- id: group-a
  name: '@deepseek-ai/cordis-plugin-group'
  group: true
  isolate:
    shell: true
  config:
    - name: '@deepseek-ai/dsh-bash-local'
      config:
        timeoutMs: 5000
    - name: './src/plugin-a.ts'
- id: group-b
  name: '@deepseek-ai/cordis-plugin-group'
  group: true
  isolate:
    shell: true
  config:
    - name: '@deepseek-ai/dsh-bash-local'
      config:
        timeoutMs: 60000
    - name: './src/plugin-b.ts'
```

plugin-a 与 plugin-b 各自看到组内 Bash 实例，互不影响。preset 自有服务必须放在 isolate realm 之后，否则会发布到进程全局层（agentPresets 在创建期会拒绝此类向根服务 realm 发布服务的行，见「能力 Seams 与核心服务」页）。

## Harness 内置服务

服务名、公开方法和源码位置由仓库自动生成到各子系统的页面（含 cordis-surface 区块）。开发插件时应以这些生成区块和服务的 TypeScript 接口为准，不要维护另一份静态清单。常用 ctx 键与归属见 references/seams.md。

## 何时不该用普通 Service

普通 Cordis `Service` 服务的是「同进程内、被主进程半的注入消费方调用」。两类**反向**调用要用别的机制：

- **浏览器半（设置面板、可视化编辑）要调主进程方法**——用 `ctx.connection.rpc.handle/call`，把方法挂到一根 `/<channel>` 上、暴露 endpoint + payload + 信封。样板见 references/connection-rpc.md。**不要**用 `@deepseek-ai/dsh-typert-protocol` 的 `@Remote`/`TypertRemoteService`：`@Remote` 把方法标记写入模块级 `WeakMap`，npm 分发场景下主进程与浏览器加载的不是同一个 module 实例，标记不可见。
- **其他主进程插件以事件方式响应**——`ctx.on(...)`（见 references/events.md），不是 Service。

判定顺序：

1. 消费方是同进程主进程半插件 → 普通 Service 或事件
2. 消费方是浏览器半（设置面板）→ `ctx.connection.rpc`
3. 跨进程/跨语言 → ACP / JSON-RPC stdio
