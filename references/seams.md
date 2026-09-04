# 核心 seam 与架构映射

DSH 的每一部分都是插件：模型适配器、工具注册表、会话日志、agent loop 本身，都可以从配置替换。一个 **seam** 是一项可替换能力，包含三种角色：声明接口的 Service Definition、实现它的 Service Provider、使用它的 Consumer（通常是面向模型的工具）。seam 正是「替换一个提供方就能改变整个产品」的原因。

## 主干核心包

| 包 | 职责 | ctx 键 |
| --- | --- | --- |
| core/session | 仅追加的 SessionEvent 日志与内存存储（唯一真源） | ctx.sessions |
| core/system-prompt | 提示词片段与工具 schema 的组装 | ctx.systemPrompt |
| core/tools | 作用域化的工具注册表与带把关的执行流水线 | ctx.tools |
| core/agent | Agent 接口、活跃注册表、agent/* 事件词汇 | ctx.agents |
| core/agent-loop | 实现 Agent 接口的默认驱动器（唯一具体循环插件） | ctx.agentLoop |
| core/scope | 按 agent 划分作用域的注册原语 | 库，无 ctx 键 |
| llm/llm | 消息与流式词汇表 + 适配器 seam | ctx.llm |

扩展插件依赖 agent（包括需要发起 Agent 时），而绝不直接依赖 agent-loop——循环保持可替换。scope/ 是唯一的非服务包（零依赖库 createScope/scopeOf/scopeTarget）。

## 能力 seam 与核心服务全表

本表完整转录自「能力 Seams 与核心服务」页面（生成自 scripts/gen-doc-graphs.ts）；列含义：所属包＝拥有服务声明的包，实现＝已知实现包。修改或使用这些服务前仍以生成页面为准。

| ctx 键 | 角色 | 所属包 | 实现 | 说明（浓缩） |
| --- | --- | --- | --- | --- |
| ctx.attachments | seam | attachment | attachment-local | 宿主在会话事件前提交已接受的图片；适配器把持久引用解析为原生内容 |
| ctx.llm | seam | llm | llm-deepseek, llm-pi-ai, llm-replay | 适配器注册提供方实现；agent-loop 与压缩消费无提供方流服务 |
| ctx.tokenMeter | core | token-meter | - | 按会话隔离的回放折叠区；不可变、带修订的测量结果 |
| ctx.toolResultPruner | core | compaction-tool-result-pruner | - | 压缩前用可回放单节点表层替换改写过大的工具结果 |
| ctx.sessions | core | session | - | 仅追加 Session 实例 + 持久事件流（持久化/查询/反馈等消费） |
| ctx.invariants | core | invariants | - | 配套子路径注册包本地检查；负责选择、唯一性、子 fiber、归属失败 |
| ctx.typert | core | typert-registry | - | 注册实时 zod 贡献；网关消费调用描述符与提供方 |
| ctx.typertGateway | core | api-gateway | - | Remote 描述符 ↔ 实时服务，经共享 Connection RPC 载体一元调用 |
| ctx.sessionPersistence | seam | session-persistence | session-persistence-jsonl, session-persistence-sqlite | 各后端持久化同一套 SessionEvent 词汇；组合时选择后端 |
| ctx.settings | seam | settings | settings-file | 插件注册命名空间 schema 并解析分层值；提供方存原始文档 |
| ctx.credentials | seam | credentials | credentials-local | 配置携带机密引用；提供方拥有值；按操作解析（轮换即刻生效） |
| ctx.sessionTelemetry | seam | session-telemetry | session-telemetry-otel | 捕获会话记录、脱敏、交给后端；输出离开当前进程 |
| ctx.storage | seam | storage | storage-json, storage-sqlite | 后端按名并列注册；领域优先形态挂到枢纽，转为不透明 KV 原语（配套 storage-domain） |
| ctx.storageDomain | core | storage-domain | - | 等待所有后端就绪，发布受生命周期约束的类型化持久状态服务 |
| ctx.messageFeedback | core | message-feedback | - | 本地逐 assistant 消息反馈、compare-and-set、Host Remote 契约；不进会话历史/遥测 |
| ctx.workspaceRegistry | core | workspace | - | 带 WorkspaceId 品牌类型的记录；Host RPC 与 GUI 投影 |
| ctx.sessionQuery | seam | session-query | session-query-sqlite | 精确读取/过滤/追踪；SQLite 后端加全文、排序、摘要、游标世代 |
| ctx.sessionReferenceResolver | core | session-reference | - | 当前表层中有界的对话快照投影为持久但不可信的消息上下文 |
| ctx.sessionTitle | seam | session-title | session-title-first-prompt-llm, session-title-all-prompts-llm | 确定性回退、最新标题折叠区、唯一可选异步提供方注册 |
| ctx.systemPrompt | core | system-prompt | - | 每步骤收集提示词段落与面向模型的工具 schema |
| ctx.tools | core | tools | - | 注册能力、Code Mode 传输、策略前/单调守卫/环绕分派/策略后/最终观测流水线 |
| ctx.userQuestions | seam | user-questions | - | UI 前端提供当前生效的人工回答提供方；tool-ask-user 在 ask() promise 上暂停调用 |
| ctx.planMode | core | plan-mode | - | 折叠计划/模式状态、轮次边界刷新用户选择、注册 /plan |
| ctx.agentPresets | core | agent-presets | - | 发现 preset 目录，创建期把 preset cordis.yml 挂载到 agent 作用域之下 |
| ctx.commands | core | commands | - | 插件注册直接面向人的命令，不经模型 |
| ctx.sessionProjections | core | session-projection | - | 各领域注册状态驱动的折叠单元；维护每个会话的水位状态 |
| ctx.sessionProjectionCache | core | session-projection-cache | - | 按会话持久检查点 + 冷读取阶梯（缓存行 + 持久化尾部回放） |
| ctx.skills | seam | skill | skill-badge, skill-filesystem | 合并提供方 skill 目录；tool-skill 渲染会话前缀目录并加载完整正文 |
| ctx.agents | core | agent | - | 实时 Agent 句柄、创建/恢复工厂 seam、进程本地发起方传播 |
| ctx.agentDefaultModel | core | agent-default-model | - | 经 settings 分层默认 ModelSelection，直接入口与 Host 支撑入口共享 |
| ctx.agentLoop | bundle | agent-loop | - | 唯一具体循环插件；扩展包依赖 dsh-agent 的事件和服务，不依赖此包 |
| ctx.goals | core | goal | - | 从会话日志折叠带修订的目标状态；实时延续激活在进程本地 |
| ctx.e2b | core | e2b | - | 共享 E2B SDK 句柄、远程工作目录与最终沙箱处置 |
| ctx.subprocess | seam | subprocess | subprocess-local, subprocess-e2b | Bash/PTY/LSP/进程外 subagent 都经它 spawn；进程树/会话生命周期、stdio 处置、kill 升级 |
| ctx.shell | seam | shell | bash-local, bash-sandbox, pwsh-local | 面向模型的 shell 工具与钩子桥接消费；沙箱/远程/PowerShell 执行器可替换 |
| ctx.shellEnv | core | shell-env | - | effect 作用域的 DSH_* 事实；shell 工具每次执行收集可信快照重建命名空间 |
| ctx.terminals | seam | terminal | terminal-bash | 注册表负责精确到 Agent 的会话身份与清理；tool-terminal 提供 owner 作用域模型接口 |
| ctx.sandbox | seam | sandbox | sandbox-local | 消费方交出即将 spawn 的确切 argv；后端按每次调用策略包装并报告强制执行 |
| ctx.sandboxPolicy | core | sandbox-policy | - | 统一保存部署默认模式与工作区根目录；执行器与提供方读取 |
| ctx.approval | seam | approval | acp | 一次性权限决策经 approval/request waterfall 分派；无回答方 fail-closed |
| ctx.permissionPresets | core | permission-presets | - | 面向用户的预设表；一次切换写 permission/preset 事件并贯通两个选项事件 |
| ctx.codeRuntime | seam | code-runtime | code-runtime-worker | 运行模型编写的程序；各后端采用不同基础环境/语言 |
| ctx.fs | seam | fs | fs-local, fs-sandbox, fs-e2b | tool-fs 读写；fs-sandbox 按共享沙箱模式限制变更；fs-observation-policy 经 fs/* 事件门禁 |
| ctx.compaction | seam | compaction | compaction-basic | 消费步骤后压力事件与请求错误恢复事件；不存在面向模型的压缩工具 |
| ctx.subagents | seam | subagent | subagent-spawn-in-process, subagent-fork-in-process, subagent-acp, subagent-codex, subagent-claude-code, subagent-dsh-sdk | 提供方实现传输；可选基于 Activation 的延续编排 |
| ctx.jobs | seam | jobs | jobs-local | 生产方（后台 bash、PTY 发送、subagent 委派）登记工作；tool-jobs 读取/列出/终止 |
| ctx.web | seam | web | web-search-exa, web-search-perplexity, web-search-deepseek, web-fetch-http | 搜索与抓取提供方注册到同一 seam；tool-web 负责稳定模型名 |
| ctx.spillStore | seam | spill | spill-local | 后端保存过大工具文本并返回定位/取回提示；spill-policy 决定何时 spill |
| ctx.directoryPicker | seam | directory-picker | directory-picker-native, directory-picker-browse | 原生后端打开 OS 选择器；浏览后端为应用内浏览器提供列表与创建原语 |
| ctx.webServer | core | webserver | - | node:http 载体：具名路由注册表、索引转换 tap、静态 dist 回退 |
| ctx.clientModules | core | modules | - | 增量 dsh.client 扫描组合 DSH_BOOT 入口图，通知重建/图变更订阅方 |
| ctx.workflowEngine | seam | workflow | workflow-worker-thread | 每上下文一个引擎、无具名提供方注册表；agent() 调用经 ctx.subagents 扇出 |
| ctx.lsp | seam | lsp | lsp-local | 提供方注册与选择 + 恰好四种操作的标准化查询执行；无协议逃生口 |
| ctx.apiProxy | core | apiproxy | - | 与传输无关的 Host 网关接口；每条打开的 Host 流自行订阅转发事件 |
| ctx.dynamicCordisRunner | core | cordis-host-runner | - | 内存定义注册表、Host 半 vm 沙箱、request-run 往返流程 |
| ctx.cordisInspect | core | cordis-host-runner | - | 注册 Host inspect 提供方、镜像 Client 提供方 manifest、路由 Client 查询 |

抽象 seam（abstract seam）的含义：接口在 harness 定义，实现由组合时选择的提供方插件提供。完整消费方与配套插件列见「能力 Seams 与核心服务」页面的生成表格，全部可用插件见「插件配置目录」。

## 新行为的归属位置

| 目标 | 机制 |
| --- | --- |
| 添加模型提供方 | 在 ctx.llm 注册其适配器 |
| 添加面向模型的能力 | 在 ctx.tools 注册；其 schema 加入提示词组装 |
| 让某个会话拥有不同的能力集合 | 组装 agent preset；其中的服务行需要 isolate realm |
| 添加 shell 执行 | 注册 ctx.shell 后端；本地后端经 ctx.subprocess spawn 进程 |
| 添加持久化终端执行 | 注册 ctx.terminals 后端和 dsh-tool-terminal |
| 添加用户命令 | 在 ctx.commands 注册；无需模型轮次即可分派 |
| 添加后台工作 | 在 ctx.jobs 注册；job_* 工具负责收集或停止 |
| 添加文件系统访问或策略 | 注册 ctx.fs 提供方，或监听 fs/* 事件 |
| 限制所启动的进程 | 使用 ctx.sandbox 后端；消费方在启动进程前包装 argv |
| 拦截请求、工具或轮次 | 使用相应的 agent/* 或 tools/* 事件；agent/turn-stopping 会停止轮次 |
| 添加模型可见上下文 | 调用 agent.inject()；它落到下一次获准的请求中 |
| 添加 UI 或编辑器集成 | 驱动 ctx.agents 并从 session/event 渲染 |
| 添加浏览器半设置面板 | 浏览器半注册 React 组件到 settings.section 等 slot；调主进程方法走 `ctx.connection.rpc.handle/call`（见 references/connection-rpc.md），**不要**用 `@Remote` |
| 添加 Web Client Chat 节点 | 注册 ConversationNodeDefinition + keyed renderer |
| 添加持久会话状态 | 扩展 SessionEventMap；从日志渲染和回放 |
| 生成会话标题 | 注册唯一的 ctx.sessionTitle 提供方 |
| 管理同会话目标 | 使用 ctx.goals；通过 agent/* 续跑 |
| fork 活跃会话 | ctx.sessions.fork(source, boundary?, childSessionId?) |
| 将注册项限定到单个 agent | 使用该 agent 的 agent.ctx |

产品功能到扩展点的更细映射（钩子、/goal、/loop、工作流、压缩、MCP、skill、cron、遥测等）见 references/plugin-forms.md。

## 轮次流程

一个**步骤** = 一次模型请求 + 它调用的工具；一个**轮次** = 零到多个步骤（领取首条输入之前打开，不再欠任何工作时关闭）。

```
turn/start
  → claim next-step input + 一条排队消息
  → 组装 prompt 段与工具 schema → agent/pre-step
  → reject | enter(messages)（首次被拒绝或改写为空 → 关闭无步骤的轮次）
  → step/start → 追加 user/message → 从日志派生模型历史
  → agent/request → llm/stream → assistant/chunk* → assistant/message
  → tool/call* → tools/pre-execute → tools/execute → tools/post-execute → tool/result*
  → step/end
  → 工具欠另一个请求或新输入到达 → 下一步；否则
  → agent/turn-stopping
turn/end
```

- turn/*、step/*、user/message、assistant/*、tool/* 是**持久会话事件**；其余分属三个事件域（会话、agent、能力）的实时扩展点。
- agent/pre-step、agent/request、llm/stream 与三个 tools/* 事件是 waterfall（监听器必须调用 next()）；agent/turn-stopping 是 serial（无 next()）。以上具体分发模式以生成目录为准。
- **模型可见即已记录**：抵达模型请求的一切必须能从日志重建，由运行时不变式断言。新增模型可见输入 = 新增会话事件（扩展 SessionEventMap 并从日志渲染）。

## 会话事件（12 种变体）

turn/start、turn/end、step/start、step/end、user/message、assistant/chunk、assistant/message、tool/call、tool/result、steering/message、todo/write、request/header。每个条目携带单调 seq、time 与按 type 判别的 data payload；surface 变体还可引用较早事件（sourceEventSeqs）并携带 surfaceOp。LLM 消息历史由日志派生（deriveMessages()），不单独存储。

## 全仓通用类型模式（…Map → derived-union）

几乎所有可扩展和类型都用同一模式：以判别标签为键的接口（…Map），联合类型由 keyof 派生；插件通过声明合并添加变体，无需修改拥有该类型的包。

```ts
interface ThingMap { 'a': { kind: 'a' }; 'b': { kind: 'b' } }
type Thing = ThingMap[keyof ThingMap]   // 判别联合

declare module '@deepseek-ai/dsh-llm' {
  interface ThingMap { 'c': { kind: 'c' } }   // 插件扩展
}
```

规范 map 包括 ContentBlockMap、MessageSourceMap、FinishReasonMap（dsh-llm）、TurnTriggerMap（dsh-session）等；工具 schema 与事件签名以生成的目录为准。
