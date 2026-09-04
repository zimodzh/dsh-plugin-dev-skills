# dsh-plugin-dev

<p align="center">
  <samp>
    <strong>中文</strong> ·
    <a href="./README.en.md">English</a>
  </samp>
</p>

<p align="center">
  <img src="https://img.shields.io/github/repo-size/zimodzh/dsh-plugin-dev-skills?style=flat-square" alt="repo size" />
  <img src="https://img.shields.io/github/last-commit/zimodzh/dsh-plugin-dev-skills?style=flat-square" alt="last commit" />
  <img src="https://img.shields.io/github/license/zimodzh/dsh-plugin-dev-skills?style=flat-square" alt="MIT license" />
  <img src="https://img.shields.io/badge/Agent_Skills-compliant-4D6BFE?style=flat-square" alt="Agent Skills compliant" />
</p>

一套遵循 [Agent Skills 规范](https://agentskills.io) 的技能，用于开发 [**DeepSeek Harness（DSH）**](https://github.com/deepseek-ai/deepseek-harness) 插件。

DSH 是一个插件化的 Agent Harness SDK：模型适配器、工具注册表、会话日志、甚至 agent loop 本身，全都是可以从配置里替换的 Cordis 插件。本技能把[官方文档](https://deepseek-harness.github.io/deepseek-harness/guide/quickstart)里散落在教程、参考手册与生成目录中的约定，收敛成一套可执行的标准——任何加载了它的 agent，都能用同一种方式开发 DSH 插件。

> **注意：** 本技能为社区维护项目，与 DeepSeek 官方无隶属关系，亦未获官方背书。

## 技能里有什么

```
dsh-plugin-dev/
├── SKILL.md      # 入口：frontmatter、9 条硬规则、6 个场景工作流、决策速查表、完成前检查清单
├── references/   # 13 份详细标准，按需加载；索引见 references/README.md
├── examples/     # 两个可复制、可运行的最小示例
│   ├── hello-plugin/
│   └── greet-tool/
└── evals/        # description 的触发评测集与评测方法
```

references 覆盖：插件形态与生命周期 · 服务与依赖注入 · 五种事件分发模式 · 插件配置 · 上下文/Fiber/注册表 API · 三种角色能力设计（Definition/Provider/Consumer）· 工具开发 · LLM 适配器协议 · 五种插件形态（工具/钩子/UI/设置面板/协议桥）· 浏览器半 Connection RPC · 打包与安装 · 仓库内 workspace 包 · 完整能力 seam 目录。

## 目录结构

```
dsh-plugin-dev/
├── SKILL.md                        # 技能入口：frontmatter、9 条硬规则、6 个场景工作流、检查清单
├── LICENSE                         # MIT 许可证
├── README.md / README.en.md        # 本说明（中文主 / 英文附）
├── references/                     # 13 份详细标准（渐进式披露，按需加载）
│   ├── README.md / README.en.md    #   目录索引：文件｜内容｜何时读
│   ├── plugin-anatomy.md           #   插件形态、生命周期、Fiber、自动清理、HMR
│   ├── services.md                 #   服务定义/提供/消费、inject、隔离
│   ├── events.md                   #   五种事件分发模式、命名
│   ├── config.md                   #   插件配置与 cordis.yml 行
│   ├── context-api.md              #   上下文 API、Fiber 类、注册表、继承的框架 API
│   ├── three-roles.md              #   能力三种角色（seam）设计
│   ├── tools.md                    #   工具开发完整约定
│   ├── llm-adapter.md              #   LLM 适配器协议
│   ├── plugin-forms.md             #   五种扩展形态 + 功能→机制映射
│   ├── connection-rpc.md           #   浏览器半 Connection RPC；裸 @Remote 禁止，完整 Typert ./remote 仍合法
│   ├── packaging.md                #   打包、安装与层序
│   ├── workspace-package.md        #   monorepo 内新建包的清单与命名
│   └── seams.md                    #   核心 seam 与能力服务全表、架构映射
├── examples/                       # 可复制、可运行的最小示例
│   ├── README.md / README.en.md    #   示例索引
│   ├── hello-plugin/               #   最小插件（生命周期 / 自动清理）
│   │   ├── README.md / README.en.md
│   │   ├── index.js
│   │   ├── package.json
│   │   └── cordis.patch.yml
│   └── greet-tool/                 #   最小模型工具（defineTool）
│       ├── README.md / README.en.md
│       ├── index.js
│       ├── package.json
│       └── cordis.patch.yml
└── evals/                          # description 触发评测集
    ├── README.md / README.en.md    #   评测方法（训练/验证集划分）
    └── trigger-queries.json        #   12 正例 + 9 负例
```

## 安装

技能名为 `dsh-plugin-dev`，Agent Skills 规范要求所在文件夹同名；本仓库名为 `dsh-plugin-dev-skills`。克隆时直接指定目标文件夹名即可一步到位：

```bash
git clone https://github.com/zimodzh/dsh-plugin-dev-skills.git ~/.claude/skills/dsh-plugin-dev
```

把目标目录换成你所用 agent 的对应路径（见下表）；也可以下载 release 压缩包，解压后把文件夹改名为 `dsh-plugin-dev`。

无需构建、无需脚本依赖、无需任何配置——以上说的是技能本身。实际开发 DSH 插件则需要一个可用的 DSH 环境：Node.js、pnpm，以及示例中用到的 `dsh`。

| Agent | 项目级 | 用户级 |
| --- | --- | --- |
| DeepSeek Harness | `<project>/.dsh/skills/`（rank 100）或 `<project>/.agents/skills/`（rank 200） | `~/.dsh/skills/`（rank 400） |
| Claude Code | `<project>/.claude/skills/` | `~/.claude/skills/` |
| Codex | `<project>/.codex/skills/` | `~/.codex/skills/` |
| VS Code Copilot | `<project>/.agents/skills/` | `~/.agents/skills/` |
| 其它兼容 agent | 按该 agent 的技能目录约定 | 同上 |

验证：向 agent 提问"开发一个 DSH 插件 / 写一个 DSH 工具"，技能应被触发；在 DSH 里也可以直接用 `skill(dsh-plugin-dev)` 工具加载确认。

## 版本对应

内容蒸馏自 [DeepSeek Harness 官方文档站](https://deepseek-harness.github.io/deepseek-harness/guide/quickstart)（2026-08 快照），并遵循官方「接口以生成参考为准」的原则：技能内容与仓库生成参考不一致时，**以生成参考为准**。发现偏差欢迎[提 issue 或 PR](https://github.com/zimodzh/dsh-plugin-dev-skills/issues)。

## 触发评测

[`evals/trigger-queries.json`](evals/trigger-queries.json) 是 description 的回归评测集（12 条正例 + 9 条负例）。修改 description 前请先跑评测并记录通过率；方法论（含训练/验证集划分、防过拟合）见 [`evals/README.md`](evals/README.md)。

## 示例

- `examples/hello-plugin` —— 最小插件（bundle 格式）：`dsh plugin --profile demo add ./examples/hello-plugin` 后 `dsh --profile demo` 启动，应看到加载日志和每 5 秒一次的心跳，卸载时自动清理。
- `examples/greet-tool` —— 最小模型工具：安装后对 agent 说 "Use the greet tool to greet Ada."，应收到 "Hello, Ada!"。

完整步骤见 [examples/README.md](examples/README.md)。

## 范围边界

覆盖**仓库内、文件式**的 DSH 插件开发：插件包、cordis.yml 行、patch overlay、工具、适配器、组合包、profile、仓库内 workspace 包。不覆盖会话内动态插件（`cordis_define`/`cordis_run` 流）与 agent preset 组合编辑——这两类由各部署的专项技能或官方工具负责。

## 维护与贡献

- 更新任何 references 前，先核对[官方文档](https://deepseek-harness.github.io/deepseek-harness/guide/quickstart)对应页面（文档站或源码生成区块），并在 PR 中注明来源。
- 遵守 Agent Skills 约束：name 为 kebab-case 且与目录一致；description ≤ 1024 字符（DSH 目录注入提醒默认 500）；正文渐进式披露。
- 欢迎 PR：修正、更多示例、扩充评测集、其它语言版本。

## License

MIT——见 [LICENSE](LICENSE)。
