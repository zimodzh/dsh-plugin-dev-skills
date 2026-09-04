# dsh-plugin-dev

<p align="center">
  <samp>
    <strong>English</strong> ·
    <a href="./README.md">中文</a>
  </samp>
</p>

<p align="center">
  <img src="https://img.shields.io/github/repo-size/zimodzh/dsh-plugin-dev-skills?style=flat-square" alt="repo size" />
  <img src="https://img.shields.io/github/last-commit/zimodzh/dsh-plugin-dev-skills?style=flat-square" alt="last commit" />
  <img src="https://img.shields.io/github/license/zimodzh/dsh-plugin-dev-skills?style=flat-square" alt="MIT license" />
  <img src="https://img.shields.io/badge/Agent_Skills-compliant-4D6BFE?style=flat-square" alt="Agent Skills compliant" />
</p>

An [Agent Skills](https://agentskills.io)–compliant skill that teaches agents how to develop plugins for [**DeepSeek Harness (DSH)**](https://github.com/deepseek-ai/deepseek-harness).

DeepSeek Harness is a plugin-based SDK for building agent harnesses: model adapters, the tool registry, the session log, even the agent loop itself — everything is a Cordis plugin that can be swapped from configuration. This skill distills the [official DSH documentation](https://deepseek-harness.github.io/deepseek-harness/guide/quickstart) into one executable standard for writing those plugins, so any agent that loads it develops DSH plugins the same way every time.

> **Note:** This is a community-maintained skill. It is not affiliated with, sponsored by, or officially endorsed by DeepSeek.

## What's inside

```
dsh-plugin-dev/
├── SKILL.md      # entry point: frontmatter, 9 hard rules, 6 scenario workflows,
│                 # decision tables, and a pre-completion checklist
├── references/   # 13 detailed standards, loaded on demand (index: references/README.md)
├── examples/     # two minimal, copy-and-run example plugins
│   ├── hello-plugin/
│   └── greet-tool/
└── evals/        # trigger-evaluation set and methodology for the description
```

The reference library covers: plugin anatomy & lifecycle · services & dependency injection · all five event dispatch modes · plugin configuration · Context/Fiber/registry APIs · three-role capability design (Definition/Provider/Consumer) · tool development · the LLM adapter protocol · five plugin forms (tool/hook/UI/settings panel/protocol bridge) · browser-half Connection RPC · packaging & installation · in-repo workspace packages · the complete capability-seam catalog.

## Directory structure

```
dsh-plugin-dev/
├── SKILL.md                        # skill entry: frontmatter, 9 hard rules, 6 scenario workflows, checklists
├── LICENSE                         # MIT license
├── README.md / README.en.md        # this doc (Chinese primary / English secondary)
├── references/                     # 13 detailed standards (progressive disclosure, on-demand)
│   ├── README.md / README.en.md    #   index: file | covers | when to read
│   ├── plugin-anatomy.md           #   plugin shapes, lifecycle, Fiber, auto-cleanup, HMR
│   ├── services.md                 #   defining/consuming services, inject, isolation
│   ├── events.md                   #   five dispatch modes, naming
│   ├── config.md                   #   plugin config and cordis.yml rows
│   ├── context-api.md              #   Context API, Fiber class, registry, inherited framework API
│   ├── three-roles.md              #   three-role capability design
│   ├── tools.md                    #   complete tool-development contract
│   ├── llm-adapter.md              #   LLM adapter protocol
│   ├── plugin-forms.md             #   five extension forms + feature→mechanism map
│   ├── connection-rpc.md           #   browser-half Connection RPC; bare @Remote forbidden, full Typert ./typert + ./remote still legal
│   ├── packaging.md                #   packaging, install, layer order
│   ├── workspace-package.md        #   in-monorepo package checklist & naming
│   └── seams.md                    #   capability-seam catalog, architecture map
├── examples/                       # minimal, copy-and-run examples
│   ├── README.md / README.en.md    #   example index
│   ├── hello-plugin/               #   minimal plugin (lifecycle / auto-cleanup)
│   │   ├── README.md / README.en.md
│   │   ├── index.js
│   │   ├── package.json
│   │   └── cordis.patch.yml
│   └── greet-tool/                 #   minimal model-facing tool (defineTool)
│       ├── README.md / README.en.md
│       ├── index.js
│       ├── package.json
│       └── cordis.patch.yml
└── evals/                          # trigger-evaluation set for the description
    ├── README.md / README.en.md    #   methodology (train/validation split)
    └── trigger-queries.json        #   12 should-trigger + 9 should-not-trigger
```

## Installation

The skill name is `dsh-plugin-dev`, and Agent Skills requires the containing folder to be named the same. This repository is called `dsh-plugin-dev-skills` — clone it directly into a folder named `dsh-plugin-dev`:

```bash
git clone https://github.com/zimodzh/dsh-plugin-dev-skills.git ~/.claude/skills/dsh-plugin-dev
```

Adjust the target directory as needed for your agent (see the table below). Prefer downloading a release tarball instead? Extract it and rename the folder to `dsh-plugin-dev`.

No build step, no script dependencies, no configuration — for the skill itself. Developing DSH plugins, however, assumes a working DSH environment: Node.js, pnpm, and `dsh` (used by the examples).

| Agent | Project-level | User-level |
| --- | --- | --- |
| DeepSeek Harness | `<project>/.dsh/skills/` (rank 100) or `<project>/.agents/skills/` (rank 200) | `~/.dsh/skills/` (rank 400) |
| Claude Code | `<project>/.claude/skills/` | `~/.claude/skills/` |
| Codex | `<project>/.codex/skills/` | `~/.codex/skills/` |
| VS Code Copilot | `<project>/.agents/skills/` | `~/.agents/skills/` |
| Other compatible agents | follow that agent's skill-directory convention | same |

To verify: ask the agent "开发一个 DSH 插件" or "write a DSH tool" — the skill should trigger. In DSH you can also load it directly with the `skill(dsh-plugin-dev)` tool.

## Version tracking

The content was distilled from the [official DeepSeek Harness documentation site](https://deepseek-harness.github.io/deepseek-harness/guide/quickstart) (snapshot dated 2026-08) and follows the project's own principle: when the skill disagrees with the repository's generated references, **the generated references win**. If you hit a discrepancy, please [open an issue or PR](https://github.com/zimodzh/dsh-plugin-dev-skills/issues).

## Trigger evals

[`evals/trigger-queries.json`](evals/trigger-queries.json) is the regression set for the skill's description (12 should-trigger + 9 should-not-trigger queries). Before changing the description, run the evals and record pass rates — [`evals/README.en.md`](evals/README.en.md) explains the methodology, including train/validation splits to avoid overfitting.

## Examples

- `examples/hello-plugin` — a minimal plugin in bundle format. Run `dsh plugin --profile demo add ./examples/hello-plugin`, then `dsh --profile demo`: you should see `[hello-plugin] plugin loaded!` and a heartbeat every 5 seconds, cleaned up automatically on unload.
- `examples/greet-tool` — a minimal model-facing tool. After installing it, ask the agent: "Use the greet tool to greet Ada." It should reply "Hello, Ada!".

See [examples/README.en.md](examples/README.en.md) for the full walkthroughs.

## Scope

This skill covers **file-based** DSH plugin development: plugin packages, cordis.yml rows, patch overlays, tools, adapters, bundles, profiles, and in-repo workspace packages. Out of scope: in-session dynamic plugins (the `cordis_define`/`cordis_run` flow) and agent-preset composition editing — those are handled by each deployment's own dedicated skills and tools.

## Contributing

- Before updating any reference, check the corresponding [official documentation](https://deepseek-harness.github.io/deepseek-harness/guide/quickstart) page (site or generated source blocks) and cite it in your PR.
- Respect the Agent Skills constraints: kebab-case `name` matching the folder; `description` ≤ 1024 chars (DSH's catalog reminder caps at 500); progressive disclosure in the body.
- PRs are welcome: fixes, more examples, a larger eval set, and translations.

## License

MIT — see [LICENSE](LICENSE).
