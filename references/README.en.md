# References

<p align="center">
  <samp>
    <strong>English</strong> ·
    <a href="./README.md">中文</a>
  </samp>
</p>

This directory is the skill's progressive-disclosure detail library: SKILL.md keeps only the hard rules, scenario workflows and checklists, while the concrete standards load on demand from these 13 files. Read just the file for the topic you're working on.

| File | Covers | Read it when |
| --- | --- | --- |
| [plugin-anatomy.md](./plugin-anatomy.md) | Plugin shapes, lifecycle, Fiber state machine, auto-cleanup, HMR | writing/modifying plugins or handling lifecycle/HMR |
| [services.md](./services.md) | Defining/consuming services, inject, isolation | defining or consuming services, injecting deps |
| [events.md](./events.md) | Five dispatch modes, naming, persistent-session-event distinction | using events for communication/extension points |
| [config.md](./config.md) | Plugin config and cordis.yml rows | making a plugin configurable |
| [context-api.md](./context-api.md) | Context API, Fiber class, registry, inherited framework API | using scopes, the service store, or Fiber API |
| [three-roles.md](./three-roles.md) | Three-role capability design | splitting a swappable capability |
| [tools.md](./tools.md) | Complete tool-development contract | developing a model-facing tool |
| [llm-adapter.md](./llm-adapter.md) | LLM adapter protocol | adding a model provider |
| [plugin-forms.md](./plugin-forms.md) | Five extension forms + feature→mechanism map | writing hook/UI/settings-panel/protocol-bridge plugins |
| [connection-rpc.md](./connection-rpc.md) | Browser ↔ host Connection RPC boilerplate, endpoint union, envelope; when Typert `./remote` stays legal | npm settings panels calling host methods; bare `@Remote` / handwritten manifests / `createRequire` source bridges |
| [packaging.md](./packaging.md) | Packaging, install, layer order | packaging/installing a plugin |
| [workspace-package.md](./workspace-package.md) | In-monorepo package checklist & naming | adding a package in the monorepo |
| [seams.md](./seams.md) | Capability-seam catalog, architecture map | looking up built-in services/placement |

Authoritative source is the DeepSeek Harness documentation site (2026-08 snapshot); where it disagrees with generated references, the generated references win.
