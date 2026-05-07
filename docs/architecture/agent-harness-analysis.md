# LocalForge as an agent harness — an objective analysis

> **Scope.** This document evaluates LocalForge against the canonical
> requirements of an "agent harness" as the term has crystallised in 2025-2026
> writing from OpenAI, LangChain, Martin Fowler, MongoDB, and the
> [awesome-harness-engineering](https://github.com/ai-boost/awesome-harness-engineering)
> taxonomy. It is not a sales document. Where LocalForge falls short of the
> reference, this document says so.
>
> **Audience.** Contributors deciding what to build next, and readers who want
> to know whether LocalForge is the right substrate for their use case.

---

## 1. What is an agent harness?

An **agent harness** is the engineered scaffolding around an LLM that turns
a stateless next-token predictor into something that can do work in the world.
The LLM by itself produces tokens; the harness is everything else: the prompt
assembly, the tool layer, the planning loop, the memory store, the
permissions checks, the verification gates, the observability plumbing, and
the orchestration that schedules and survives failures.

The shorthand from MongoDB's harness essay captures the inversion well: the
LLM is the *smallest* part of an agent system. OpenAI's "harness engineering"
post and their internal Symphony project have made the same point — they
shipped roughly a million lines of Codex-generated code in five months not
because the model got dramatically better, but because they invested heavily
in the harness around it.

A "coding agent harness" is the task-specific case: the harness is shaped to
drive a coding agent through software-engineering work end-to-end.

### The canonical components (twelve)

Synthesising the [awesome-harness-engineering](https://github.com/ai-boost/awesome-harness-engineering)
taxonomy with Martin Fowler's harness-engineering article and LangChain's
"anatomy of an agent harness", a production agent harness has these
distinguishable concerns:

| # | Component | What it does |
|---|---|---|
| 1 | **Agent loop** | The Thought → Action → Observation cycle: prompt → model call → tool call → tool result → next prompt → … until done. |
| 2 | **Planning & task decomposition** | Turn a goal into ordered, dependency-aware tasks, with the option to replan locally when a task fails. |
| 3 | **Context delivery & compaction** | Decide what the model sees on each call: retrieval, summarisation, compaction near the context window, prompt caching. |
| 4 | **Tool design** | Schema-validated tools with stable names, structured outputs, and risk annotations. |
| 5 | **Skills / MCP / extensions** | Standardised interface (MCP, A2A) for plugging in third-party capabilities and reusable user-authored skills. |
| 6 | **Permissions & authorization** | Pre-action policy checks (allow/deny rules, hooks, sandboxing) so the agent cannot do dangerous things by accident. |
| 7 | **Memory & state** | Working memory + archival memory + recall memory, often three-tier, with cross-session persistence and semantic search. |
| 8 | **Task runners & orchestration** | Multi-agent topologies, supervisor/sub-agent delegation, durable queues, isolated workspaces per task. |
| 9 | **Verification & CI integration** | Eval gates, type-checks, tests, browser checks, and "proof-of-work" artefacts (CI status, screenshots, walkthroughs). |
| 10 | **Observability & tracing** | Per-decision event logs, OTEL traces, replayable history. |
| 11 | **Debugging & developer experience** | Mid-run inspection, tool-call replay, tool-definition introspection. |
| 12 | **Human-in-the-loop** | Structured approve/deny/continue gates with low-friction pauses for sensitive actions. |

Two cross-cutting properties matter just as much:

- **Durability.** Long-running agents need checkpoint/resume so a server
  restart, network blip, or model-server hiccup doesn't lose hours of work.
- **Self-evolution.** Mature harnesses let the agent grow new skills, learn
  from past sessions, and refine its memory without retraining the model.

The rest of this document checks LocalForge against each of these.

---

## 2. The roles LocalForge actually plays

LocalForge is a real agent harness — narrow, opinionated, and local-first —
not a wrapper around a model API. The clearest way to see this is to list
the harness roles it owns, separately from the LLM-side responsibilities it
delegates to Pi (`@mariozechner/pi-coding-agent`). The Pi/LocalForge split
is documented in detail in [`pi-integration.md`](../concepts/pi-integration.md).

| Harness role | LocalForge mechanism | Where in the code |
|---|---|---|
| **Goal capture** | Bootstrapper chat panel | `app/api/agent-sessions/[id]/messages/route.ts`, `chat_messages` table |
| **Planning** | Pi session with feature-CRUD-only tools turns chat into a backlog | `app/api/agent-sessions/[id]/generate-features/route.ts`, `lib/agent/feature-crud-tools.ts` |
| **Task store** | `features` + `feature_dependencies` SQLite tables, with priority + status | `lib/db/schema.ts`, `lib/features.ts` |
| **Scheduler** | Picks highest-priority ready feature; respects dependencies | `findNextReadyFeatureForProject` in `lib/features.ts` |
| **Concurrency control** | Per-project cap (1–3) on parallel coding sessions | `getMaxConcurrentAgentsForProject` in `lib/agent/orchestrator.ts` |
| **Workspace isolation** | Each project gets its own `projects/<slug>/` folder; Pi runs with `cwd` set there | `lib/projects.ts`, `spawnAgentRunner` in orchestrator |
| **Process isolation** | Coding sessions run in spawned Node child processes | `spawn(process.execPath, [agent-runner.mjs, …])` |
| **Permissions** | Workspace-guard Pi extension blocks any write/edit/bash outside the project folder, and broad `node` killers that would crash the harness itself | `makeWorkspaceGuardExtension` in `scripts/agent-runner.mjs` |
| **Tool layer** | Two profiles: bootstrapper gets only feature-CRUD tools (no fs/shell); coding agent gets `read/write/edit/bash/grep/find/ls` | `customTools` + `noTools: "builtin"` vs full toolset |
| **Watchdog** | 30-min per-session timeout (`LOCALFORGE_SESSION_TIMEOUT_MS`) kills hung runners | `SESSION_TIMEOUT_MS` in orchestrator |
| **Verification** | Optional post-run Playwright check (Chromium navigates dev server, asserts non-empty title, screenshots) | `runPlaywrightTests` in `agent-runner.mjs` |
| **Persistence** | SQLite control plane: projects, features, sessions, logs, chat, settings | `lib/db/schema.ts` |
| **Observability** | Pi event stream → JSON-lines on stdout → `agent_logs` rows + SSE broadcast | `attachChildHandlers` + `appendAgentLog` |
| **Live UI** | Server-Sent Events for live tool-call/log streaming, with on-reconnect replay from SQLite | `app/api/agent/stream/[sessionId]/route.ts` |
| **Failure handling** | On failure, demote feature priority and return to backlog; on success, mark complete and auto-continue | `finalizeSession` + `maybeContinueWithNextFeature` |
| **Recovery on restart** | Reconciles orphaned `in_progress` sessions on next start (closes session row, returns feature to backlog) | top of `startOrchestrator` |
| **Project completion** | Once every feature is `completed`, project flips to `completed` and broadcasts a one-shot event for the celebration screen | `markProjectCompletedIfAllDone` |

That's the harness. Pi owns the Thought-Action-Observation loop *inside* a
single feature; LocalForge owns everything that surrounds, schedules,
isolates, gates, persists, and reports on those loops.

---

## 3. When each role activates — a session walkthrough

Here is exactly when each harness role engages, ordered by the sequence the
user experiences:

1. **User creates a project** (UI → `createProject()`).
   *Roles active:* workspace isolation (folder created), persistence
   (project row inserted), settings (per-project `.pi/models.json` written).
2. **User opens the bootstrapper and chats about the app idea.**
   *Roles active:* goal capture (chat messages persisted to
   `chat_messages`).
3. **User clicks "Generate features".**
   *Roles active:* planning. A Pi session spins up *inside* the API route
   process with `noTools: "builtin"` and only the feature-CRUD tools. Pi
   reads the chat transcript and emits one tool call per feature, plus
   dependency edges. Hard cap: 40 turns. The result is a populated backlog.
4. **User clicks "Start" on the kanban.**
   *Roles active:* scheduler (picks highest-priority ready feature),
   concurrency control (refuses if at cap), persistence (creates
   `agent_sessions` row, flips feature to `in_progress`), recovery (any
   orphaned in-progress features are reset to backlog first).
5. **Orchestrator spawns the runner child process.**
   *Roles active:* process isolation (`spawn(node, [agent-runner.mjs, ...])`),
   workspace isolation (cwd = project folder), watchdog (30-min timer
   armed), observability (startup log line + SSE broadcast).
6. **Runner instantiates Pi.**
   *Roles active:* tool layer (full built-in toolset registered),
   permissions (workspace-guard extension hooks every `tool_call`), context
   delivery (system prompt built from `buildCodingSystemPrompt`, including
   project-specific `coder_prompt` override).
7. **Pi runs its loop.** Each Pi event becomes:
   - `tool_execution_start` → "Reading X" / "Editing Y" log line via
     `summariseToolUse`. *Observability.*
   - `message_update` (text_delta) → buffered for final summary.
     *Observability.*
   - `turn_end` → turn counter; abort if `LOCALFORGE_MAX_TURNS` exceeded.
     *Watchdog.*
   - Workspace-guard fires on any write/edit/bash that escapes the project
     folder. *Permissions.*
8. **Pi finishes (or errors).** The runner classifies the error:
   - Transient (`ECONNREFUSED`, "rate_limit", "Failed to generate a valid
     tool call", 5xx) → up to 3 retries with exponential backoff, fresh Pi
     session each time. *Failure handling.*
   - Non-transient → straight to `done` with `outcome: "failure"`.
9. **Optional verification.** If `playwright_enabled=true`, the runner
   imports `chromium` directly, navigates the project's dev server, asserts
   non-empty title, takes a screenshot. *Verification.*
10. **Runner emits `done` and exits.** Orchestrator's `child.on("close")`
    flips the feature to `completed` (success) or demotes its priority and
    returns it to backlog (failure). *Persistence + failure handling.*
11. **Auto-continue.** `maybeContinueWithNextFeature` fills any open agent
    slots with the next ready features. *Scheduler + concurrency.*
12. **Project completion.** When the last feature flips to `completed`,
    `markProjectCompletedIfAllDone` flips the project status and broadcasts
    a one-shot event. The UI fires confetti. *Project lifecycle.*

The full event trail per session is captured in `agent-runner-debug.log` at
the repo root for post-mortem analysis.

---

## 4. Requirement-by-requirement scorecard

Against the twelve canonical components plus the two cross-cutting
properties, here is LocalForge's status. Scores: **🟢 Solid**,
**🟡 Partial / minimal**, **🔴 Missing**.

| # | Requirement | Score | Evidence |
|---|---|:-:|---|
| 1 | Agent loop (TAO) | 🟢 | Delegated to Pi. Real tool loop, real tool execution, real observations fed back. |
| 2 | Planning & task decomposition | 🟡 | Done **once** at project setup via the bootstrapper. No mid-project replanning loop, no goal hierarchy, no Plan.md → Implement.md split. Failed features are not analysed; they are simply demoted and re-queued. |
| 3 | Context delivery & compaction | 🔴 | None at the harness layer. The system prompt is a fixed string + project `coder_prompt`. No retrieval, no summarisation, no compaction near context window, no prompt caching beyond what the local provider does. The 128k context window is asserted as a flat number. |
| 4 | Tool design | 🟢 | Pi's built-in tools are schema-validated. The custom feature-CRUD tools are defined with TypeBox and explicit error returns. |
| 5 | Skills / MCP / extensions | 🔴 | `DefaultResourceLoader` is constructed with `noSkills: true, noPromptTemplates: true, noThemes: true, noExtensions: true`. Pi's own extension/skill mechanism is **explicitly disabled**. There is no MCP integration, no A2A. |
| 6 | Permissions & authorization | 🟡 | Workspace guard is a real, working pre-action check (path containment, broad-killer regex). But it is the *only* layer — no allow/deny lists for tools, no risk annotations, no per-project policy file, no escalation prompt. |
| 7 | Memory & state | 🔴 | `SessionManager.inMemory()`. Each Pi session starts cold. No cross-session memory, no archival memory, no recall, no fact graph, no learnings carried between features. The only persisted "memory" is the agent's own log of what it did, which it cannot read on the next run. |
| 8 | Task runners & orchestration | 🟡 | Single-project, single-tier orchestrator with parallel agent slots (≤ 3). No supervisor/sub-agent delegation, no multi-project queue, no per-task isolated *git worktree* (each feature shares the same project folder). |
| 9 | Verification & CI integration | 🟡 | Optional Playwright check is real but minimal: it asserts a non-empty page title and takes one screenshot. There is no test suite invocation, no type-check gate, no lint gate, no semantic review pass. The agent's own success signal is treated as authoritative when verification is off (the default). |
| 10 | Observability & tracing | 🟢 | Two layers: durable `agent_logs` table with replay on reconnect, and live SSE EventEmitter. Plus `agent-runner-debug.log` for full Pi-event traces. No OTEL, but for a local-first tool the durable+live split is solid. |
| 11 | Debugging & DX | 🟡 | Logs are good and per-session debug files exist. But there is no mid-run inspection UI, no tool-call replay tool, no eval harness for the harness itself. |
| 12 | Human-in-the-loop | 🟡 | Stop button (SIGTERM → SIGKILL fallback) and manual feature edit. No mid-run approve/deny gates on tool calls, no "agent is about to do X — approve?" prompts. |
| — | **Durability** | 🟡 | SQLite control plane is durable: feature status, session rows, logs all survive restart, and orphaned sessions are reconciled on next start. **But:** Pi's own conversation state is in-memory; if a session is killed mid-feature, the agent restarts the feature from scratch, not from the last tool call. There is no checkpoint/resume of an in-flight Pi session. |
| — | **Self-evolution** | 🔴 | Nothing. There is no skill library written by past runs, no failure post-mortem fed back into the next run's prompt, no editable agent memory file, no "lessons learned" mechanism. The harness is the same on run #1 and run #1000. |

Aggregate read: LocalForge is a **competent narrow-scope harness** — the
"glue" parts (orchestration, persistence, observability, isolation,
permissions) are present and working. It is **weak on the parts that matter
most for autonomy at scale** — context engineering, memory, durability of
in-flight runs, verification depth, planning beyond the initial backlog,
and any form of self-improvement.

---

## 5. Long-running task support — does it really work?

**Short answer: it works for a *project*, but not for a *single agentic
task*.**

The distinction matters. LocalForge can keep running for hours,
auto-continuing through 30 features in a backlog. That's "long-running" in
the wall-clock sense. But each individual Pi session is bounded:

- **Per-session watchdog**: 30 minutes hard limit
  (`LOCALFORGE_SESSION_TIMEOUT_MS`).
- **Per-session turn cap**: 100 default, 1000 ceiling (`LOCALFORGE_MAX_TURNS`).
- **Per-session retry count**: 3 attempts, only on transient errors.

If the harness or the host machine restarts mid-feature:

- The control plane (feature status, log history) is preserved in SQLite.
- The Pi session itself is **lost completely**. There is no checkpoint of
  the model conversation, no replay log of tool calls and their outputs.
- The orphaned in-progress feature is silently demoted back to "backlog" on
  next start (see `startOrchestrator` reconciliation block).
- The next agent that picks it up will start from scratch with the same
  feature description — including any work the previous agent already did
  to disk, which it will see fresh and may overwrite or duplicate.

**Why this isn't true durable execution.** The harness-engineering
literature (LangGraph, DBOS, Temporal) treats the agent as a long-running
*workflow* with checkpoint-at-each-step semantics. Every tool call is
recorded with its inputs and outputs, every node transition is durable, and
on restart the workflow resumes at the last successful step with a
deterministic replay. Tool calls are made idempotent by design. LocalForge
has none of this. It treats the Pi session as a black box that either
finishes or doesn't.

**Why it works in practice anyway.** Local features are small (~minutes of
LLM time), the watchdog catches genuine hangs, the auto-retry handles
transient model-server errors, and the file system itself is the durable
substrate for code. So if the agent crashes after editing five files, those
five files are still there — the next attempt just re-implements the
remaining work. For a local-first hobby/dev tool this is acceptable. For a
production "leave it running overnight" harness it is not.

---

## 6. Self-improvement support — does it really work?

**Short answer: no. Not in any meaningful sense.**

LocalForge has no mechanism for an agent to leave behind anything useful
for the next agent. Specifically:

- No persistent **agent memory file** (e.g. `AGENT_NOTES.md`) that
  successive runs read and append to.
- No **skill library**: an agent cannot define a reusable "skill" or "macro"
  that future agents can call. Pi's skill mechanism is explicitly disabled
  via `noSkills: true`.
- No **failure post-mortem**: when a feature fails, the only signal
  preserved is "priority demoted". The error message, the failing tool
  call, the partial work — none of it is summarised into the next prompt.
- No **prompt evolution**: the system prompt is a function of static config
  + project settings. There is no place where the harness mutates the
  prompt based on what worked or didn't work in the past.
- No **eval feedback loop**: the post-run Playwright check is pass/fail,
  not "here's what failed, try again with this hint".
- No **cross-project transfer**: lessons from project A do not surface in
  project B even on the same machine.

The harness is, in self-evolution terms, **completely static**. The frame
of reference matters: this is the level of the *Hermes Agent / SAGE / MemSkill*
literature, where the harness deliberately gives the agent a writable
long-term store and re-reads it on each run. LocalForge does not do this.

---

## 7. Tool calling — how does it work?

Tool calling is one of LocalForge's stronger areas, because it inherits
Pi's machinery and adds two thoughtful customisations on top.

**Layer 1: the underlying mechanism is Pi's.** Pi speaks OpenAI-compatible
tool-calling against LM Studio / Ollama. The tool schemas (name,
description, parameter shape) are built into Pi's resource loader and
injected into each model call. The model emits structured tool_use blocks;
Pi parses, validates, executes, and feeds the result back as tool_result
blocks for the next turn. LocalForge does **not** reimplement any of this.

**Layer 2: LocalForge picks the toolset per session type.** This is the
clever part. The harness has two distinct tool profiles:

- **Bootstrapper** — `noTools: "builtin"` + `customTools:
  buildFeatureCrudTools(projectId)`. The bootstrapper agent has *only*
  `list_features`, `create_feature`, `update_feature`, `delete_feature`,
  and dependency tools. It cannot read files, run shell commands, or grep.
  Its only path to changing the world is through validated SQLite writes.
- **Coding agent** — `tools: ["read", "bash", "edit", "write", "grep",
  "find", "ls"]` (Pi's built-ins) + workspace-guard extension. Full file
  and shell access, scoped to the project folder.

This per-session restriction is the harness doing real work: Pi *could*
expose all those tools to both agents; LocalForge deliberately doesn't.

**Layer 3: pre-execution hook (the workspace guard).** The harness
registers a Pi extension (`makeWorkspaceGuardExtension`) that fires on
every `tool_call` event and can return `{ block: true, reason }`. It
checks:

- `write`/`edit` paths must resolve inside the project root (Windows MSYS
  paths like `/c/Users/...` are normalised first).
- `bash` commands are scanned for absolute paths that escape the workspace,
  with sensible exemptions for `/dev/null`, `/tmp/`, `/usr/`, etc.
- `bash` commands are scanned for broad node-killers (`taskkill /F /IM
  node.exe`, `killall node`, certain `pkill -f node` shapes) that would
  crash the harness server.

When the guard blocks a call, Pi gets the rejection back as a tool_result
and the model can adapt — exactly the right behaviour.

**Layer 4: tool-call observability.** Every `tool_execution_start` event
is summarised (`summariseToolUse`) into a human readable line ("Reading
package.json", "Running: npm install", "Editing src/App.tsx") and emitted
to stdout, persisted in `agent_logs`, and broadcast via SSE.

What's missing at the tool layer:

- No **tool annotations** for risk level (mutating vs read-only,
  destructive vs reversible).
- No **MCP** integration — third-party MCP servers cannot be plugged in.
- No **per-project allow/deny lists** beyond the workspace guard's
  hard-coded rules.
- No **approval gates** — every allowed tool call runs immediately.

---

## 8. Honest critique

LocalForge is best understood as a **demo-quality coding harness** that
takes the unusual stance of being entirely local-model driven. Within that
framing, it is well-built: the orchestrator is robust, the SQLite control
plane is correct, the SSE log feed is genuinely live, and the
two-different-toolsets approach to bootstrapping vs coding is a real piece
of harness design that many cloud harnesses skip.

The problems are systemic, not cosmetic, and they all point in the same
direction:

1. **The harness assumes the agent is reliable.** No checkpointing, no
   memory, no post-mortems on failure. This is fine when the agent works
   the first time. It does not degrade gracefully when it doesn't.
2. **Verification is the weakest leg.** A single "page has a title"
   Playwright check is not a meaningful eval. The agent's own claim of
   success is treated as authoritative by default.
3. **There is no learning surface.** Run the same project ten times and
   the harness behaves identically every time. No skills accrue, no notes
   persist, no preferences are remembered.
4. **Pi's own extension points are deliberately turned off.** Skills,
   prompt templates, themes, extensions — Pi has all of them and
   LocalForge says `no` to each. This was probably the right call for v0
   (deterministic baseline) but it leaves the most powerful parts of the
   embedded SDK unused.
5. **The bootstrapper plans once, with no replanning.** Real projects
   discover requirements as they're built. The current model is "decide
   the backlog up front, then chip away". This works for small apps and
   breaks for anything where reality contradicts the original plan.
6. **Single-machine, single-process control plane.** SQLite + in-memory
   EventEmitter is fine for one developer; not fine for "team uses the
   same harness". The architecture is consciously local-first, which is a
   choice not a defect, but worth naming.

The *integrity* of the harness is good. The *ambition* is small.

---

## 9. Recommendations — making LocalForge a real long-running, self-evolving harness

Below is a concrete, prioritised path. Each item is sized so that a
LocalForge-built feature could implement it.

### Tier 1 — make individual sessions durable (foundation)

These are cheap to implement and unlock everything else.

1. **Persist Pi's conversation state per session.** Replace
   `SessionManager.inMemory()` with an on-disk store keyed by `session_id`,
   under `data/sessions/<id>/`. Write after every `message_end`. On restart,
   reload and resume. Use Pi's own `SessionManager` interface — the disk
   variant is straightforward.
2. **Persist tool-call outputs.** Add an `agent_tool_calls` table:
   `(id, session_id, turn, tool_name, args_json, result_json, started_at,
   ended_at, ok)`. Write on `tool_execution_start` / `tool_execution_end`.
   This is the durable replay log a real workflow engine needs.
3. **On session restart, replay the tool-call log into the new Pi session
   as cached results** so the agent doesn't redo work that already
   happened. Idempotency keys on file writes are a follow-up.
4. **Surface per-session failure context.** When a feature fails,
   summarise the last assistant message, the last 3 tool calls, and any
   errors into a `failures` table. The next agent that picks the feature
   reads this and is told "previous attempt failed because X — avoid
   repeating".

### Tier 2 — give the agent memory it can write

5. **Add a per-project `AGENT_NOTES.md`** that lives in
   `projects/<slug>/.localforge/AGENT_NOTES.md` and is automatically
   prepended to every coding agent's system prompt. Add a `note` tool that
   appends a line to it. The agent can record assumptions, gotchas,
   decisions; future runs see them.
6. **Add a global skill library.** Re-enable Pi's `noSkills: false` and
   adopt the existing `.agents/skills/` folder structure that already
   ships in the repo. Curate a small starter set (workspace-guard-friendly
   bash patterns, common Next.js fixes, Drizzle idioms) and let users add
   more.
7. **Failure → memory feedback loop.** When a feature succeeds, ask the
   agent (one extra Pi call) for a one-line lesson and append it to the
   project notes. When a feature fails, do the same for the failure mode.
   This is the cheapest possible self-evolution mechanism and it pays off
   immediately.

### Tier 3 — strengthen verification

8. **Run the project's own test suite** if `package.json` has a `test`
   script and emit pass/fail counts as `test_result` log lines. Today the
   runner runs Playwright headlessly but does not run the project's own
   tests. This is a five-line change in `agent-runner.mjs`.
9. **Type-check gate.** If `tsconfig.json` exists, run `tsc --noEmit`
   after the agent finishes and emit the output as tool feedback for one
   final retry turn (not a fresh session — same context).
10. **Build gate.** Same idea for `npm run build`. The agent's own loop
    already does this but the harness should make it a hard post-condition.
11. **Eval the harness itself.** A `tests/harness-eval/` directory with
    fixed projects + expected feature counts + golden screenshots, run on
    CI. This is the meta-eval that catches regressions in the harness
    rather than in any one agent.

### Tier 4 — better planning

12. **Mid-project replanning.** When 3 consecutive features fail, pause
    auto-continue and run a planner-Pi session that reads the current
    backlog + failure notes and proposes either revised acceptance
    criteria or a new feature breakdown. Show the diff to the user for
    approval.
13. **Goal hierarchy.** Add an `epics` table (or just a parent_id on
    features) so the planner can decompose top-down without flattening the
    backlog into one tier.
14. **Explicit Plan.md / Implement.md artefacts.** Adopt the
    awesome-harness convention: every project has a `docs/PLAN.md` the
    bootstrapper writes and the orchestrator reads on each schedule
    decision. The plan can be edited by hand mid-project.

### Tier 5 — broaden the tool surface

15. **MCP integration.** Add a `mcpServers` setting and pass through to
    Pi's tool registry. Users can plug in `mcp-server-filesystem`,
    `mcp-server-postgres`, etc. without harness code changes. This is the
    biggest force-multiplier in the list.
16. **Per-tool permission policy.** A `permissions.json` per project with
    `allow`/`deny`/`ask` rules per tool name + argument shape.
    Default-allow today; default-ask for `bash` would be a big leap in
    safety with little friction.
17. **Tool risk annotations.** Mark which tools mutate filesystem vs read,
    surface that in the activity panel, and let the user toggle "always
    confirm destructive bash" without confirming every read.

### Tier 6 — turn it into a real long-running harness

18. **Workflow-style durable execution.** Adopt or write a tiny
    state-machine engine: each Pi session is a workflow run; each turn is
    a step; checkpoints are written after every step; on restart the
    workflow resumes. The work in Tier 1 is the prerequisite.
19. **Git worktree per feature.** Each in-progress feature gets its own
    worktree under `projects/<slug>/.worktrees/feature-<id>/`. Two
    parallel agents on the same project no longer fight over the same
    files. Merge to the main worktree on success.
20. **Distributed control plane (opt-in).** Replace SQLite with a
    Postgres mode behind a single connection-string setting so a team
    can share one harness. This is the biggest architectural change in
    the list and should be Postgres-or-stay-local, not both.

### Tier 7 — human-in-the-loop polish

21. **Mid-run approve/deny gate.** A Pi extension that pauses on tool
    calls flagged as risky, surfaces them in the UI (existing SSE
    channel), and resumes on approve/deny. Builds on the workspace guard
    pattern already proven to work.
22. **Diff-preview before file writes.** Show the proposed diff in the
    activity panel and let the user one-click reject. Small UX project,
    big trust win.

---

## 10. Bottom line

LocalForge is a **real harness** — narrow, correct, and observable, with a
genuine permission layer and a clean separation between the model SDK (Pi)
and the harness (everything else). Its scope today is "build a small app
end-to-end on a local model with auto-continue", and within that scope it
works.

To honestly call itself a "long-running, self-evolving agentic harness" it
needs four things, in this order:

1. Durable Pi sessions (Tier 1).
2. A writable agent memory + failure feedback loop (Tier 2).
3. Real verification gates (Tier 3).
4. MCP and per-tool permissions (Tier 5).

Tiers 4, 6, and 7 are the polish that takes it from "ambitious local-first
harness" to "production-grade autonomous engineering platform". They are
worth doing in that order, but only after the first four tiers land — each
of those is a load-bearing prerequisite for the next.

---

## Sources

- [What Is an Agent Harness? — Firecrawl](https://www.firecrawl.dev/blog/what-is-an-agent-harness)
- [Harness engineering for coding agent users — Martin Fowler](https://martinfowler.com/articles/harness-engineering.html)
- [The Anatomy of an Agent Harness — LangChain](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness)
- [The Agent Harness: Why the LLM Is the Smallest Part of Your Agent System — MongoDB](https://www.mongodb.com/company/blog/technical/agent-harness-why-llm-is-smallest-part-of-your-agent-system)
- [Harness engineering: leveraging Codex in an agent-first world — OpenAI](https://openai.com/index/harness-engineering/)
- [An open-source spec for Codex orchestration: Symphony — OpenAI](https://openai.com/index/open-source-codex-orchestration-symphony/)
- [awesome-harness-engineering — GitHub](https://github.com/ai-boost/awesome-harness-engineering)
- [Long-running Agents — Addy Osmani](https://addyo.substack.com/p/long-running-agents)
- [AI Agent Workflow Checkpointing and Resumability — Zylos Research](https://zylos.ai/research/2026-03-04-ai-agent-workflow-checkpointing-resumability)
- [Durable Execution — LangChain docs](https://docs.langchain.com/oss/python/langgraph/durable-execution)
- [A Survey of Self-Evolving Agents — arXiv 2507.21046](https://arxiv.org/html/2507.21046v3)
- [SAGE: Self-evolving Agents with Reflective and Memory-augmented Abilities — arXiv 2409.00872](https://arxiv.org/html/2409.00872v2)
- [Hermes Agent: Self-Improving AI with Persistent Memory — YUV.AI](https://yuv.ai/blog/hermes-agent)
