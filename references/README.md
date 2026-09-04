# 参考资料

<p align="center">
  <samp>
    <strong>中文</strong> ·
    <a href="./README.en.md">English</a>
  </samp>
</p>

本目录是 dsh-plugin-dev 技能「渐进式披露」的详细标准库：SKILL.md 只保留硬规则、场景工作流与检查清单，具体约定按需加载到这里的 13 份文件里。开发时命中哪个主题就读哪一份，不必通读。

| 文件 | 内容 | 何时读 |
| --- | --- | --- |
| [plugin-anatomy.md](./plugin-anatomy.md) | 插件形态、生命周期、Fiber 状态机、自动清理、HMR | 写/改插件、处理生命周期或 HMR |
| [services.md](./services.md) | 服务定义/提供/消费、inject、隔离 | 定义或消费服务、注入依赖 |
| [events.md](./events.md) | 五种事件分发模式、命名、持久会话事件区分 | 用事件通信、监听扩展点 |
| [config.md](./config.md) | 插件配置与 cordis.yml 行 | 让插件可配置 |
| [context-api.md](./context-api.md) | 上下文 API、Fiber 类、注册表、继承的框架 API | 用作用域、服务存储或 Fiber API |
| [three-roles.md](./three-roles.md) | 能力三种角色（seam）设计 | 拆分可替换能力 |
| [tools.md](./tools.md) | 工具开发完整约定 | 开发模型工具 |
| [llm-adapter.md](./llm-adapter.md) | LLM 适配器协议 | 接入新模型提供方 |
| [plugin-forms.md](./plugin-forms.md) | 五种扩展形态与功能→机制映射 | 写钩子/UI/面板/协议桥插件 |
| [connection-rpc.md](./connection-rpc.md) | 浏览器半 ↔ 主进程 Connection RPC 样板、endpoint 联合、信封、迁移清单 | 浏览器半面板要调主进程服务时；遇到 `@Remote`/`TypertRemoteService` 时 |
| [packaging.md](./packaging.md) | 打包、安装与层序 | 打包安装、交付插件 |
| [workspace-package.md](./workspace-package.md) | monorepo 内新建包清单与命名 | 在 monorepo 内新建包 |
| [seams.md](./seams.md) | 核心 seam 与能力服务全表、架构映射 | 查内置服务、归属位置 |

权威来源为 DeepSeek Harness 官方文档站（2026-08 快照）；与仓库生成参考不一致处，以生成参考为准。
