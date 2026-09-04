# 插件形态扩展与功能映射

harness 扩展的参考模式。代码片段省略了 import 和辅助实现，无法直接复制运行；具体编写路径见本技能的 plugin-anatomy / tools / llm-adapter / connection-rpc 与 SKILL.md 各场景。

## 五种扩展形态

1. **工具插件** —— 在 ctx.tools 上注册。第一方工具用类型化的 defineTool（execute 参数类型化、结果构造、run_in_background 模式），也可直接注册原始 JSON Schema ToolDefinition（MCP 来源的工具就是这样到达的）。
2. **钩子插件** —— 在拦截点上运行的普通 Cordis 插件，不需要外部协议（「原生钩子」）。钩子插件本身并不等同于权限门禁；沙箱、权限和 plan-mode 插件也使用这些扩展点。
3. **UI 插件（聊天侧）** —— 从 session/event 事件流渲染（助手 token 流以 assistant/chunk 到达，加上轮次/步骤边界与工具活动），并通过 agent.followup() / agent.steer() 把输入驱动回去。浏览器插件向内建 Web Client 贡献业务行时，注册 ConversationNodeDefinition 与 keyed Chat renderer。
4. **浏览器半面板插件（设置/管理侧）** —— 插件同时拥有主进程半与浏览器半（dsh.client.inject 入口）。浏览器半注册 React 面板（settings.section / 其他 slot），主进程半把方法经 `ctx.connection.rpc.handle('/<channel>', dispatch, {authority:'loopback'})` 暴露给浏览器调用。`dispatch` 是单 switch；端点失败用 `RpcResult` 信封返回而不是 throw。完整样板见 references/connection-rpc.md。
5. **外部协议驱动** —— 将协议对端接入 ctx.agents，可服务 UI 或自动化客户端。packages/acp/acp 是仅面向自动化的完整示例（ACP JSON-RPC stdio）。

> 第 3 类 UI 插件是「聊天侧」——给 Chat 业务节点贡献分片。第 4 类是「设置侧」——给设置/管理面板暴露主进程方法。两类都涉及浏览器半，但通信机制不同：聊天侧走 `session/event` 流，浏览器半面板走 `ctx.connection.rpc`。**不要把第 4 类混进第 3 类**——不要用 `@Remote` 或 `TypertRemoteService` 跨两边暴露方法（详见 references/connection-rpc.md 第一节）。

## 钩子插件（以权限门禁为例）

```ts
import type { Context } from '@deepseek-ai/cordis'
import type { PreToolDecision, ToolExecution } from '@deepseek-ai/dsh-tools'

declare function isAllowed(exec: ToolExecution): Promise<boolean>

export const name = 'permission-gate'
export function apply(ctx: Context) {
  ctx.on('tools/pre-execute', async (exec, next): Promise<PreToolDecision> => {
    if (!(await isAllowed(exec))) {
      return { kind: 'deny', reason: 'Denied by policy.' }
    }
    return next()
  })
}
```

选择规则：

| 需要 | 扩展点 |
| --- | --- |
| 可重排的策略层（allow/deny/ask） | tools/pre-execute waterfall |
| 不变式需要单调的最终拒绝 | ctx.tools.guard() |
| 包裹实际分发生命周期（超时/重试/指标；仅 exec.signal 可替换） | tools/execute |
| 显式结果变换 | tools/post-execute |
| 对不可变最终结果的受限观察 | tools/result |

## UI 插件

```ts
import type { Context } from '@deepseek-ai/cordis'
import { createUserMessage } from '@deepseek-ai/dsh-llm'
import { SessionId } from '@deepseek-ai/dsh-session'

declare function render(text: string): void
declare function onUserInput(handler: (text: string) => void): void

export const name = 'my-ui'
export const inject = ['agents']
export function apply(ctx: Context) {
  ctx.on('session/event', (_session, event) => {
    if (event.type === 'assistant/chunk' && event.data.chunk.type === 'text-delta') {
      render(event.data.chunk.text)
    }
  })
  onUserInput(text => ctx.agents.get(SessionId('client-session'))?.followup(createUserMessage({
    content: [{ type: 'text', text }],
    source: { kind: 'user' },
  })))
}
```

## 浏览器半面板插件（settings.section 等 slot）

主进程半在 `apply()` 里挂 `ctx.connection.rpc.handle`：

```ts
import type { Context } from '@deepseek-ai/cordis'
import { dispatch, RPC_CHANNEL } from './host/rpc.ts'
import { YourService } from './host/service.ts'

export const name = 'your-plugin'
export const inject = ['connection']
export async function apply(ctx: Context) {
  const service = new YourService(ctx)
  ctx.inject(['connection'], () => {
    ctx.effect(() => {
      const handler = async (endpoint: string, payload: unknown) =>
        dispatch(service, endpoint, payload)
      const disposer = ctx.connection.rpc.handle(RPC_CHANNEL, handler, { authority: 'loopback' })
      // disposer 形状跨实现不一致，统一兼容
      return () => {
        Promise.resolve(disposer).then((d) => {
          if (typeof d === 'function') d()
          else if (d !== null && typeof d === 'object' && 'dispose' in d) {
            (d as { dispose: () => void }).dispose()
          }
        }).catch(() => {})
      }
    }, 'your-plugin: rpc channel')
  })
}
```

浏览器半注册 React 组件并直接调 host RPC：

```ts
// src/client/index.ts —— dsh.client.inject 的另一个 apply
export async function apply(ctx: { /* slots/connection/locale */ }) {
  ctx.slots.inject('settings.section', () =>
    ctx.slots.register(
      { name: 'settings.section', id: 'your-plugin', order: 40, inject: () => ({ ctx }) },
      YourSettingsSection,
    ),
  )
}
```

```tsx
// src/client/YourSettingsSection.tsx
import { callRpc } from './rpc.ts'
export function YourSettingsSection({ ctx }: { ctx: ClientContext }) {
  const [view, setView] = useState<YourView | null>(null)
  useEffect(() => { callRpc<YourView>(ctx, 'list').then(setView).catch(console.error) }, [])
  // ...
}
```

完整样板、endpoint 联合、`RpcResult<T>` 信封、错误类、`ctx.connection` 结构化强转、disposer 三种返回形状兼容、迁移检查清单见 references/connection-rpc.md。

## 外部协议驱动

- stdio 驱动拥有 stdout；通过工厂创建或恢复 agent；把协议请求映射为 followup() 或 cancel()。
- 底层提示词请求返回其**持久入队回执**——不会通过关联 MessageId 与 turn/end 获得结果。整个 agent 的状态应单独发布。
- 自动化方法可以从回执等待到下一次 idle，并概括这一显式拥有的区间；UI 通常持续观察开放式事件流。
- 通过 AgentHandle.dispose() 拆除 agent，使 dispose 达到完全停稳。

## 可运行的组装示例

- 可运行叶子从 examples/*/cordis.yml 加载各自的插件树；根目录 demo:* 脚本和这些叶子目录是权威清单。
- 产品 dsh 启动器负责 Web 与一次性 headless 执行；ACP 叶子使用 @deepseek-ai/dsh-acp-demo；JSON-RPC 叶子使用 @deepseek-ai/dsh-sdk-jsonrpc-demo；headless 快照叶节点显式挂载 @deepseek-ai/dsh-agent-spine-demo 与 JSONL 持久化。

## 功能→机制映射

每个产品功能都映射到一个文档化扩展点上的监听器——没有任何一行修改循环本身。system-prompt/assemble 是专家协作式整体装配变换：其返回的装配结果具有权威性，监听器作者有责任保留活跃的 Code Mode 与结构化输出协议的贡献。

| 产品功能 | 插件机制 |
| --- | --- |
| 钩子系统（用户级 + 项目级） | agent/session-start、agent/pre-step、agent/request、tools/pre-execute、tools/post-execute、agent/turn-stopping 上的监听器；dsh-hooks-claude-code / dsh-hooks-codex 桥接器把钩子配置文件映射到这些扩展点 |
| /goal | ctx.goals 管理持久状态；Round driver 通过公共 Agent 调度同会话 Round |
| /loop | 在 turn/end 会话事件上 followup() 下一次迭代，或强制继续 |
| 动态工作流 | ctx.workflowEngine + worker-thread 引擎 + workflow 工具；结构化子任务经作用域化提示词/工具注册、单调工具守卫、最终 tools/result 提交与 concludeTurn() 标记强制输出 |
| 排队消息 + steering | 核心 Agent.followup() / Agent.steer() |
| 上下文压缩（自动 + 手动） | ctx.compaction seam + dsh-compaction-basic；自动压力检查运行在串行 agent/pre-step，标准溢出恢复在 agent/request-error |
| 系统提示词可配置性 | ctx.systemPrompt.section()，支持排序与作用域局部覆盖 |
| AGENTS.md（根目录） | 一个读取该文件的 section 提供方 |
| AGENTS.md（子目录，按需触发）+ 文件变更通知 | 从 watcher / 工具结果监听器调用 agent.inject() |
| 内置工具 | ctx.tools.register()；schema 自动流入装配——dsh-tool-* 系列（bash、fs、web、subagent、todo）是已交付示例 |
| ToolSearch / 渐进式披露 | 当可见集变化时替换作用域化的 ctx.tools.restrict() 注册；注册表保持展示、查找和执行三者对齐 |
| 工具截止时间 / 重试 / 指标 | 用 tools/execute 包裹核心分发；包装层可替换 exec.signal、委托执行，并在同一词法生命周期内检视规范化结果 |
| 最终工具结果指标 / 审计 / 捕获 | 用 tools/result 观察不可变权威结果；仅当需要变换结果或附加上下文时才用 tools/post-execute |
| 单调终端轮次策略 | 从成功的终端工具调用 ToolExecution.concludeTurn()；该步骤后循环停止 |
| 子进程沙箱（landlock / sandbox-exec） | 通过 dsh-bash-sandbox 使用 ctx.sandbox 后端；能力级拒绝用 tools/pre-execute |
| 权限系统 / AskUserQuestion | 从 tools/pre-execute 返回 ask 并经 ctx.approval 应答；为普通用户提问注册独立的面向模型 ask 工具 |
| Plan mode | @deepseek-ai/dsh-plan-mode：落日志的 plan/mode 状态、plan:policy 引导段、/plan [message] 入口、/plan off 直接退出、经用户评审的 exit_plan_mode 出口 |
| subagent 委派 | ctx.subagents 提供方注册表（spawn/fork in-process、acp、codex、claude-code、dsh-sdk）+ dsh-tool-subagent 向模型暴露已配置提供方 |
| MCP | 每个服务器一个插件：发现工具 → ctx.tools.register() |
| skill（技能） | section + 工具注册；调用时通过 inject() 注入 skill 内容 |
| 记忆 | section 提供方 + 工具 |
| 定时任务（cron） | 插件注册面向模型的调度工具；定时器触发 → 空闲时 followup(…, {source: {kind: 'cron', …}})／忙碌时 inject() 通知 |
| UI（GUI；CLI 输出 JSONL） | 监听 session/event（助手分片、边界、工具活动）；输入 → followup() |
| Web Client Chat 业务节点 | 注册 ConversationNodeDefinition 与 conversation.chat.node keyed renderer |
| 遥测 / 可回放 trace | session/event → JSONL；回放 = sessions.create(id, { seed }) |
| 模型适配器 | 通过 registerAdapter 注册 LlmAdapter 子类（dsh-llm-deepseek、dsh-llm-pi-ai） |
| 插件热重载 | 每个注册都是一个 ctx.effect → 随仓库提供的 HMR 直接生效 |
