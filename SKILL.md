---
name: dsh-plugin-dev
description: 开发 DeepSeek Harness (DSH) 插件的标准与权威参考：编写/修改/审查/调试 DSH/Cordis 插件、服务、事件、插件配置、模型工具、LLM 适配器、三种角色拆分、打包安装、workspace 包、cordis.yml 组合时使用；提到 DSH 插件、Cordis、plugin、服务、事件、工具、适配器即触发。 The authoritative standard for developing DeepSeek Harness (DSH) plugins — create, modify, review or debug DSH/Cordis plugins, services, events, config, model tools, LLM adapters, three-role capabilities, packaging, and cordis.yml composition.
license: MIT
compatibility: 适用于任何支持 Agent Skills 的 agent。内容面向 DeepSeek Harness 的 Cordis 插件生态（@deepseek-ai/* 包）。
metadata:
  author: Stardust
  version: "1.2.0"
---

# 开发 DSH 插件标准

本技能是开发 DeepSeek Harness（DSH）插件的唯一标准：它把 DSH 官方文档中分散在教程、参考手册与生成目录里的约定，收敛为可执行的工作流、硬规则与检查清单。DSH 的一切都是插件——模型适配器、工具注册表、会话日志乃至 agent loop 本身——因此按本技能行事，就是在按 DSH 自己的架构方式扩展它。

## 适用范围

- 仓库内、文件式的 DSH 插件开发：编写插件包、注册 cordis.yml 行、patch overlay、开发工具、接模型、打包安装。
- DSH monorepo 内的 workspace 包开发（packages/<group>/<pkg>）也属本技能（references/workspace-package.md）。
- 本技能**不**覆盖：会话内动态插件（cordis_define/cordis_run 流）与 agent preset 组合编辑——这两类由各部署的专项技能或官方工具覆盖。

## 硬规则（任何场景都必须遵守）

1. **接口以生成参考为准。** 服务名、公开方法、事件签名、ctx 键均以仓库自动生成的子系统页面与 TypeScript 接口为准；不要凭服务名、示例或旧代码推断完整 API，也不要维护另一份静态清单。
2. **所有贡献都是副作用。** 通过 ctx 做的一切注册（事件监听、工具、适配器、ctx.effect）在插件卸载时自动撤销；不要在模块作用域创建进程级/页面级副作用；不返回 disposer 的第三方订阅要主动查清清理机制。
3. **waterfall 监听器必须调用 next()。** 不调用 next() 即有意短路下游（用于拦截/网关），不是可选项。
4. **失败要响亮。** apply 抛异常则进程终止；配置校验失败则明确报错；schema 应表达自身完备的约束，不要在运行时悄悄吞掉错误。
5. **必需依赖用 inject 声明，可选依赖用 ctx.get() 判空。** 不要用 inject 规避 undefined 检查；也不要直接访问未声明注入的 ctx.xxx——未声明的服务经服务解析器求值可能得到 undefined。
6. **配置一律 Schemastery。** 导出 interface Config 与同名 Schema，默认值写在 schema 里；不导出普通对象充当 Config；凡不同部署可能改值的参数都必须进配置。
7. **工具 execute 返回规范 JSON 值，不返回内容块。** 面向人类的文本放 output.render；部署策略/钩子不要内建进工具体。
8. **模型可见即已记录。** 新增任何模型可见输入，都要落在会话日志可重建的机制里（新增持久事件或经 agent.inject()），并有运行时不变式断言。
9. **浏览器半调主进程方法走 `ctx.connection.rpc`。** 跨 host/client 半暴露方法必须经 Connection RPC 通道（`ctx.connection.rpc.handle` + `ctx.connection.rpc.call`），样板见 references/connection-rpc.md。**不要**用 `@deepseek-ai/dsh-typert-protocol` 的 `@Remote`/`TypertRemoteService`——`@Remote` 把方法标记写入模块级 `WeakMap`，要求两边加载同一个 module 实例，npm 分发场景无法满足；这是已经踩过的坑。

## 标准工作流

### 场景 A：新建一个插件

1. 确定插件要贡献什么（服务？工具？监听？），用 references/seams.md 的「新行为的归属位置」表选择机制。
2. 创建 src/<name>.ts，导出 name、inject（可选）、apply(ctx, config?)。先写函数形态；要对外提供服务时再换 Service 类形态。
3. 本地开发回路（源码 checkout）：

```bash
mkdir -p scratch-plugin/src
```

```yaml
# scratch-plugin/cordis.yml —— patch overlay；插件路径必须是绝对路径
- insert:
    - id: hello
      name: '/absolute/path/to/deepseek-harness/scratch-plugin/src/my-plugin.ts'
```

```bash
pnpm dsh web --patch ./scratch-plugin/cordis.yml   # 打开 http://127.0.0.1:3080
```

4. 验证加载与卸载：启动日志出现、停用时资源确实清理（references/plugin-anatomy.md）。

### 场景 B：给模型加一个工具

1. inject: ['tools']，用 defineTool 定义（name/description/parameters/output/execute）。
2. 按 references/tools.md 的 execute 约定实现：schema DSL 声明参数与输出、execute 返回规范值、遵守 exec.signal、可选 presentationMeta/UI 卡片。
3. 需要策略时挂 tools/* 事件（pre-execute/guard/execute/post-execute/result），不要把策略写死在工具体里（权限门禁示例见 references/plugin-forms.md）。
4. 重启后让模型实际调用验证。

### 场景 C：可替换能力（多种提供方）

1. 按 references/three-roles.md 拆 Definition / Provider / Consumer 三个包（按需合并角色，不要预防性拆分）。
2. Definition 拥有 Request/Result 类型与抽象 Service；Provider 实现；Consumer 暴露为工具或服务消费方。
3. 在 cordis.yml 只列 Provider 与 Consumer 行；换提供方只换一行。

### 场景 D：接入新的模型提供方

1. 继承 LlmAdapter 实现 stream()，按 StreamChunk 协议输出分片；错误用带稳定 code 的 LlmError（或在带内以 finish { kind: 'error' | 'aborted' } 结束）；每个 HTTP 请求合并 attributionHeaders() 并传递 signal。
2. ctx.llm.registerAdapter(['<provider>'], adapter)；可选覆写 resolveModel()/listModels()。
3. 在组合中配置 agent-loop 的 provider/model 验证。详见 references/llm-adapter.md（含 7 条协议义务）。

### 场景 E：打包与安装交付

1. 建组合包：package.json 声明 dsh.bundle（指向 cordis.patch.yml），patch 行按包名引用插件；仅供 import 的库包不声明 dsh.bundle。
2. 安装：dsh plugin --profile <name> add <pkg>；先 dsh --profile <name> --dump-config 验证层，再启动。
3. 记住层顺序与整行替换语义（references/packaging.md）。

### 场景 F：在 DSH monorepo 内新建包

按 references/workspace-package.md 的逐文件清单执行：创建包（package.json 不变式/tsconfig/src/README）→ 根配置注册 → 包拓扑与命名（角色词表）→ README Model Experience 结构 → 验证命令。

## 决策速查

| 我要…… | 机制 | 详情 |
| --- | --- | --- |
| 向其他插件公开能力 | Service（类形态） | references/services.md |
| 消费已有能力 | inject 或 ctx.get() | references/services.md |
| 插件间通信 / 扩展点 | ctx.on / ctx.emit 等事件（五种分发模式） | references/events.md |
| 让用户可配置 | Config + Schemastery | references/config.md |
| 给模型加能力 | defineTool → ctx.tools.register（或原始 JSON Schema） | references/tools.md |
| 接新模型提供方 | LlmAdapter + ctx.llm.registerAdapter | references/llm-adapter.md |
| 拆可替换能力 | 三种角色（Definition/Provider/Consumer） | references/three-roles.md |
| 分发插件 | 组合包 + profile | references/packaging.md |
| 查内置服务 / 归属位置 | seam 目录、架构映射 | references/seams.md |
| 作用域/服务存储/Fiber API | ctx.extend/isolate/provide/mixin、fiber.* | references/context-api.md |
| 钩子 / UI / 协议桥插件 | 四种扩展形态 + 功能→机制映射 | references/plugin-forms.md |
| 浏览器半面板调主进程服务 | `ctx.connection.rpc.handle/call`，endpoint + 信封 | references/connection-rpc.md |
| 在仓库内新建包 / 命名 | workspace 包清单 + 角色词表 | references/workspace-package.md |

## 完成前检查清单

- [ ] 接口查过生成参考（服务/事件/ctx 键均以生成为准），没有凭名字猜 API。
- [ ] 所有注册走 ctx（事件/工具/适配器/effect），卸载可清理；顺序敏感的清理放在同一个 ctx.effect 里。
- [ ] 必需依赖注入声明正确；可选依赖有 undefined 处理。
- [ ] 有配置的插件：Config 用 Schemastery，默认值进 schema，无效配置在加载时响亮失败。
- [ ] 有工具：execute 返回规范 JSON 值；render 负责人类可读；策略走 tools/* 事件；遵守 exec.signal；并发安全声明了 isConcurrencySafe（未声明 → exclusive）。
- [ ] 有 waterfall 监听：调用并返回 next()（除非有意短路）。
- [ ] 有 LLM 适配器：StreamChunk 协议完整（块配对、index 按首次出现、usage 在 finish 前、arguments 全程原始 JSON 字符串）；错误走两条合法路径之一；不支持的字段抛 UNSUPPORTED；需要原生回放时发 finish.replayState；传了 attributionHeaders 与 signal。
- [ ] 组合行与层序正确；新增行在 --dump-config 中可见；HMR/重启后无残留注册。
- [ ] 若模型可见内容变化：落在日志可重建的机制内。
- [ ] 浏览器半调主进程方法：走 `ctx.connection.rpc.handle/call`，**不**用 `@Remote`/手写 typert manifest/源码树桥接。
- [ ] 新建 workspace 包：package.json 不变式、恰一个 aggregate、命名符合角色词表、README 有 Model Experience 结构，且 constraints/typecheck/lint/build/hygiene 全绿。

## 参考文件（按需加载，不要一次全读）

> 完整索引与「何时读」对照表见 references/README.md。

- references/plugin-anatomy.md —— 写/改插件、处理生命周期或 HMR 时读
- references/services.md —— 定义或消费服务、注入依赖时读
- references/events.md —— 用事件通信、监听扩展点时读
- references/config.md —— 让插件可配置时读
- references/context-api.md —— 用作用域、服务存储或 Fiber API 时读
- references/three-roles.md —— 拆分可替换能力时读
- references/tools.md —— 开发模型工具时读
- references/llm-adapter.md —— 接入新模型提供方时读
- references/plugin-forms.md —— 写钩子/UI/协议桥插件时读
- references/connection-rpc.md —— 浏览器半面板要调主进程服务方法时读；遇到 `@Remote` / `TypertRemoteService` 时也读
- references/packaging.md —— 打包安装、交付 plugin 时读
- references/workspace-package.md —— 在 monorepo 内新建包时读
- references/seams.md —— 查内置服务、归属位置或架构映射时读

## 验证

- 格式自检：name 为 kebab-case 且与目录名一致；description 非空且简洁（本技能控制在 500 字符内）；正文用标准 Markdown；所有引用的 references 文件真实存在。
- 目录命名：本技能名（name）为 dsh-plugin-dev，与托管仓库名 dsh-plugin-dev-skills 不同——克隆/解压后必须把文件夹命名为 dsh-plugin-dev（与 name 一致），否则按目录名寻址的加载器可能发现不了本技能。
- 有 skills-ref CLI 的环境可执行：skills-ref validate ./dsh-plugin-dev
- 触发回归：修改 description 后运行 evals/trigger-queries.json 评测集（方法见仓库 evals/README.md）。
- 行为验证：按场景 A 走通一个最小插件（加载日志 + 卸载清理），再按场景 B 走通一个 greet 工具。
