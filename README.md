# LocalForge

> **This is a clone of the original [leonvanzyl/localforge](https://github.com/leonvanzyl/localforge), enhanced with comprehensive onboarding documents.**
>
> Demo video: [No Claude Code. No Codex. Just Local AI Agents](https://www.youtube.com/watch?v=L_AMm7fD7tQ)

> Build apps on autopilot with local AI models. No cloud, no API keys, no per-token billing.

## Quick Start

### 1. Prerequisites

- **Node.js 20+** — [download here](https://nodejs.org/)
- **LM Studio** — [download here](https://lmstudio.ai/)

### 2. Set up LM Studio

1. Open LM Studio and download a model (e.g. `google/gemma-4-31b`)
2. Load the model and start the local API server (default: `http://127.0.0.1:1234`)

### 3. Install and run LocalForge

```bash
git clone https://github.com/leonvanzyl/localforge.git
cd localforge
npm install
npm run db:migrate
npm run dev
```

Open **http://localhost:7777** in your browser.

### 4. Build something

1. Create a new project from the sidebar
2. Describe your app — the AI bootstrapper generates features automatically
3. Click **Start** and watch agents build it feature by feature

---

## Documentation

The full docs are in [`docs/index.md`](docs/index.md). Start there if you want
to understand how LocalForge works internally: project creation, bootstrapper
feature generation, orchestrator scheduling, Pi agent execution, SQLite state,
SSE logs, settings, and troubleshooting.

---

## What is LocalForge?

LocalForge lets you describe an app in plain language and watch AI coding agents
build it on your own hardware. Point it at a model running in LM Studio, click
Start, and the orchestrator breaks your idea into features, tracks them on a
kanban board, and deploys agents to implement and test them one at a time.

## Pi is the brain. LocalForge is the harness.

LocalForge does **not** ship its own LLM agent loop. The actual coding is done
by **[Pi](https://www.npmjs.com/package/@mariozechner/pi-coding-agent)** —
the open-source `@mariozechner/pi-coding-agent` SDK — which LocalForge embeds
as a library and points at your local model.

```
LocalForge                          │  Pi
────────────────────────────────────┼──────────────────────────────────
Kanban, project folders,            │  AgentSession (turn loop),
SQLite state, orchestrator,         │  ModelRegistry, AuthStorage,
SSE log feed, settings UI,          │  built-in tools (read, write, edit,
workspace-guard extension           │  bash, grep, find, ls),
                                    │  resource loader, session manager
```

LocalForge invokes Pi in exactly **two** places:

1. **Bootstrapper** (`app/api/agent-sessions/[id]/generate-features/route.ts`) —
   chat → backlog. Pi runs with **no built-in tools**; it can only call
   custom feature-CRUD tools that write to SQLite. Pi acts as a structured-
   output engine here.
2. **Coding agent** (`scripts/agent-runner.mjs`, spawned by the orchestrator) —
   feature → code. Pi gets the **full built-in toolset** plus a workspace-
   guard extension that blocks any write/edit/bash outside the project
   folder. This is where Pi reads files, edits source, runs `npm install`,
   fixes type errors, etc.

Each Pi session is single-purpose, single-feature, and disposable. LocalForge
never forks Pi — it uses it as a library, configures it per session, subscribes
to its event stream (`tool_execution_start`, `message_update`, `turn_end`),
and turns those events into the live activity panel you see in the UI.

For the full deep-dive — model config, system prompts, hook points, lifecycle
of a single session, what does and does not call Pi — see
[`docs/concepts/pi-integration.md`](docs/concepts/pi-integration.md).

## How the orchestrator works

1. You create a project, either manually or by chatting with the AI bootstrapper.
2. Features land in the **Backlog** column of the kanban, ordered by priority
   and respecting dependencies.
3. Click **Start** — LocalForge spawns an agent session pointed at the configured
   local model, passes the highest-priority ready feature, and moves the card to
   **In Progress**.
4. The agent writes code, runs Playwright tests, and captures screenshots. Live
   output streams into the activity panel via SSE.
5. On success the card moves to **Completed**. On failure the feature returns to
   the backlog with demoted priority so other features can go first.
6. When all features pass, confetti.

## Scripts

| Command | What it does |
| --- | --- |
| `npm run dev` | Start the dev server on port 7777 |
| `npm run build` | Production build |
| `npm start` | Start production server |
| `npm run db:generate` | Generate a Drizzle migration from schema changes |
| `npm run db:migrate` | Apply pending migrations |
| `npm test` | Run Playwright tests |

## Tech stack

- **Frontend:** Next.js 16 (App Router) + React 19, Tailwind CSS + shadcn/ui, dnd-kit, Sonner
- **Backend:** Next.js API routes (Node.js), SQLite + Drizzle ORM, Server-Sent Events
- **Agents:** Pi coding-agent SDK, configured for LM Studio/Ollama via OpenAI-compatible endpoints
- **Testing:** Playwright

## Project layout

```
app/                 Next.js App Router routes + API handlers
  api/               REST API endpoints
components/          React components
  ui/                shadcn/ui primitives
lib/
  db/                Drizzle schema + SQLite connection
  agent/             Pi agent integration + orchestrator
data/                SQLite database file (git-ignored)
projects/            User-created project folders (git-ignored)
drizzle/             Generated migrations
tests/               Playwright specs
```

## Configuration

Per-project model config is stored in `.pi/models.json` inside each project folder.
Override per project via project settings, or globally via **Settings** in the sidebar.

**Playwright verification** (off by default) runs after each feature; **Playwright headed browser**
shows a real Chromium window during that check and passes `playwright-cli open --headed` to the
coding agent so you can watch browser automation locally. When the `CI` environment variable is set,
verification stays headless regardless of the headed toggle. Headed mode uses a short Playwright
`slowMo` so actions are easier to follow.

## License

Apache License 2.0 — see [LICENSE](LICENSE) for details.
