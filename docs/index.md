# LocalForge

LocalForge is a local-first autonomous coding harness: you describe an app, it turns the idea into ordered features, then local model-driven coding agents build those features in project folders on your machine. The docs below explain how the system works from the UI click down to the spawned agent process.

---

## Documentation

| Section | What's inside |
|---------|---------------|
| [What is LocalForge?](overview/what-is-this.md) | Mental model, end-to-end flow, and scope boundaries |
| [Key concepts](overview/key-concepts.md) | Glossary for projects, features, sessions, providers, logs, and settings |
| [Quickstart](getting-started/quickstart.md) | Install, configure a local model, migrate SQLite, and run the app |
| [Project lifecycle](concepts/project-lifecycle.md) | How a project moves from folder creation to completion |
| [Bootstrapper](concepts/bootstrapper.md) | How chat becomes a validated backlog |
| [Orchestrator](concepts/orchestrator.md) | How LocalForge chooses and schedules agent work |
| [Agent runner](concepts/agent-runner.md) | How the child process talks to Pi, writes code, verifies, and reports back |
| [Pi integration](concepts/pi-integration.md) | Where Pi sits, when it's invoked, how it's configured, and what it does (and doesn't) own |
| [System design](architecture/system-design.md) | Runtime components, data flows, and design decisions |
| [**Agent harness blueprint**](architecture/agent-harness-blueprint.md) | Vendor-neutral reference for the agent-harness pattern — what *should* exist, regardless of implementation. The prescriptive sibling of the analysis below. |
| [Agent harness analysis](architecture/agent-harness-analysis.md) | LocalForge scored against the blueprint, with a prioritised path to long-running and self-evolving |
| [High-impact projects](projects.md) | Six solo-feasible, publishable projects this codebase + docs make uniquely well-positioned to undertake |
| [Runtime settings](reference/runtime-settings.md) | Global settings, project overrides, environment variables, and package scripts |
| [Common issues](troubleshooting/common-issues.md) | Model, database, server, agent, and verification failures |

New to the codebase? Read [what LocalForge is](overview/what-is-this.md), then [project lifecycle](concepts/project-lifecycle.md), then [agent runner](concepts/agent-runner.md).
