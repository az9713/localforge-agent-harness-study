# Agent Harness Study — using LocalForge as the case study

> **What this repo is.** A close-reading of a real, working coding-agent
> harness. The headline asset is
> [**`docs/architecture/agent-harness-analysis.md`**](docs/architecture/agent-harness-analysis.md) —
> an objective scorecard of LocalForge against the canonical 12 components of
> an agent harness (synthesised from OpenAI's harness-engineering work,
> LangChain's harness anatomy, Martin Fowler, MongoDB, and the
> `awesome-harness-engineering` taxonomy), plus a 7-tier roadmap for turning
> any narrow harness into a long-running, self-evolving one.
>
> **What this repo isn't.** A new product. The runnable code is a clone of
> [leonvanzyl/localforge](https://github.com/leonvanzyl/localforge) — kept
> intact so the analysis stays grounded in real, executable code. Demo:
> [No Claude Code. No Codex. Just Local AI Agents](https://www.youtube.com/watch?v=L_AMm7fD7tQ).

---

## Start here

| If you want to… | Read |
|---|---|
| Understand what an agent harness *is* and how to evaluate one | [**Agent harness analysis**](docs/architecture/agent-harness-analysis.md) — the headline document |
| Understand how Pi (`@mariozechner/pi-coding-agent`) plugs into a harness | [Pi integration deep-dive](docs/concepts/pi-integration.md) |
| Run the harness yourself | [Quick start](#quick-start) below |
| Read the full onboarding docs (architecture, lifecycle, settings, troubleshooting) | [`docs/index.md`](docs/index.md) |

---

## The headline contribution

LocalForge by itself is a small, opinionated, local-first coding harness. The
contribution of *this* fork is treating it as a **specimen** — a complete,
running, narrow-scope harness small enough to read end-to-end — and then
measuring it against what the broader literature says a harness should do.

The result lives in
[**`docs/architecture/agent-harness-analysis.md`**](docs/architecture/agent-harness-analysis.md):

- **What is an agent harness?** Twelve canonical components: agent loop,
  planning, context delivery, tool design, skills/MCP, permissions, memory,
  orchestration, verification, observability, debugging, human-in-the-loop —
  plus durability and self-evolution as cross-cutting properties.
- **The roles LocalForge plays**, mapped to specific files and code paths.
- **A session walkthrough** showing exactly *when* each role activates.
- **A requirement-by-requirement scorecard** with 🟢 Solid / 🟡 Partial /
  🔴 Missing verdicts and the evidence behind each.
- **Honest answers** to the three questions everyone asks of a harness:
  - *Does it support long-running tasks?* — Project-long yes, per-session
    no. Pi state is in-memory; killed sessions restart from scratch.
  - *Does it support self-improvement?* — No. Pi's skill/extension/
    prompt-template hooks are explicitly disabled, and there is no
    failure-feedback loop.
  - *Does it support tool calling?* — Yes, with two thoughtful
    customisations on top of Pi's mechanism: per-session toolset profiles
    and a workspace-guard `tool_call` extension.
- **A 7-tier prioritised roadmap** to evolve a narrow harness like
  LocalForge into a real long-running, self-evolving one — starting with
  durable Pi sessions, then writable agent memory + failure feedback, then
  real verification gates, then MCP and per-tool permissions.

If you only read one file in this repo, read that one.

## Why LocalForge as the case study

Three reasons it's a useful specimen:

1. **It's small enough to read.** The whole harness — orchestrator, agent
   runner, persistence, observability — is a few hundred lines across half
   a dozen files. You can hold the entire system in your head.
2. **It's a real harness, not a wrapper.** It has genuine harness
   responsibilities: process isolation, workspace guard, durable control
   plane, SSE observability, scheduler with concurrency control, watchdog,
   reconciliation on restart. None of these are about the LLM. All of them
   are the harness doing harness work.
3. **It deliberately leaves the LLM SDK alone.** Pi
   (`@mariozechner/pi-coding-agent`) owns the agent loop; LocalForge owns
   everything else. That clean split is unusually good for studying where
   the harness ends and the model begins — see
   [Pi integration deep-dive](docs/concepts/pi-integration.md).

---

## Quick start

### 1. Prerequisites

- **Node.js 20+** — [download here](https://nodejs.org/)
- **LM Studio** — [download here](https://lmstudio.ai/)

### 2. Set up LM Studio

1. Open LM Studio and download a model (e.g. `google/gemma-4-31b`)
2. Load the model and start the local API server (default: `http://127.0.0.1:1234`)

### 3. Install and run

```bash
git clone https://github.com/az9713/localforge-tutorial.git
cd localforge-tutorial
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

## Pi is the brain. LocalForge is the harness.

This split is the single most important thing to understand about the
specimen. LocalForge does **not** ship its own LLM agent loop — Pi
(`@mariozechner/pi-coding-agent`) does. LocalForge embeds Pi as a library
and provides the harness around it.

```
LocalForge (the harness)            │  Pi (the agent SDK)
────────────────────────────────────┼──────────────────────────────────
Kanban, project folders,            │  AgentSession (turn loop),
SQLite state, orchestrator,         │  ModelRegistry, AuthStorage,
SSE log feed, settings UI,          │  built-in tools (read, write, edit,
workspace-guard extension,          │  bash, grep, find, ls),
verification, scheduler             │  resource loader, session manager
```

LocalForge invokes Pi in exactly **two** places:

1. **Bootstrapper** (`app/api/agent-sessions/[id]/generate-features/route.ts`) —
   chat → backlog. Pi runs with **no built-in tools**; it can only call
   custom feature-CRUD tools that write to SQLite. Pi as a structured-
   output engine.
2. **Coding agent** (`scripts/agent-runner.mjs`, spawned by the
   orchestrator) — feature → code. Pi gets the **full built-in toolset**
   plus a workspace-guard extension that blocks any write/edit/bash
   outside the project folder.

Each Pi session is single-purpose, single-feature, and disposable. The full
deep-dive — model config, system prompts, hook points, lifecycle — is in
[`docs/concepts/pi-integration.md`](docs/concepts/pi-integration.md).

---

## How the harness operates (one session)

1. You create a project, either manually or by chatting with the
   bootstrapper.
2. Features land in the **Backlog** column of the kanban, ordered by
   priority and respecting dependencies.
3. Click **Start** — the orchestrator picks the highest-priority ready
   feature, spawns a child process, instantiates Pi inside it, and moves
   the card to **In Progress**.
4. The agent reads files, writes code, runs `npm install`, fixes type
   errors. Live output streams to the activity panel via SSE; durable
   copies land in the `agent_logs` table.
5. On success the card moves to **Completed** and auto-continue picks the
   next ready feature. On failure the feature returns to the backlog with
   demoted priority so other features can go first.
6. When every feature is `completed`, the project flips to `completed` and
   broadcasts a one-shot event. Confetti.

The harness owns every step except the model call inside step 4. That's
what makes it a harness.

---

## Tech stack (the specimen)

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
docs/                Onboarding + analysis (the contribution)
  architecture/
    agent-harness-analysis.md   ← the headline asset
    system-design.md
  concepts/
    pi-integration.md
    orchestrator.md
    agent-runner.md
    bootstrapper.md
    project-lifecycle.md
```

## Scripts

| Command | What it does |
| --- | --- |
| `npm run dev` | Start the dev server on port 7777 |
| `npm run build` | Production build |
| `npm start` | Start production server |
| `npm run db:generate` | Generate a Drizzle migration from schema changes |
| `npm run db:migrate` | Apply pending migrations |
| `npm test` | Run Playwright tests |

## Configuration

Per-project model config is stored in `.pi/models.json` inside each project
folder. Override per project via project settings, or globally via
**Settings** in the sidebar.

**Playwright verification** (off by default) runs after each feature;
**Playwright headed browser** shows a real Chromium window during that
check and passes `playwright-cli open --headed` to the coding agent so you
can watch browser automation locally. When the `CI` environment variable
is set, verification stays headless regardless of the headed toggle.
Headed mode uses a short Playwright `slowMo` so actions are easier to
follow.

---

## Credits

Runnable code is a clone of
[leonvanzyl/localforge](https://github.com/leonvanzyl/localforge) by Leon
van Zyl, used unmodified as the study specimen. Demo video:
[No Claude Code. No Codex. Just Local AI Agents](https://www.youtube.com/watch?v=L_AMm7fD7tQ).
The agent-harness analysis, the Pi integration deep-dive, and the rest of
the `docs/` tree are the contribution of this fork.

## License

Apache License 2.0 — see [LICENSE](LICENSE) for details.
