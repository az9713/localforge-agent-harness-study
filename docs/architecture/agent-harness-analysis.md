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

### Harrison Chase's "deep agent" — the contemporary canonical pattern

The most precise contemporary articulation comes from Harrison Chase
(LangChain CEO, [NVIDIA AI Podcast, 2026](https://www.youtube.com/watch?v=c-fsL0gsmo0)).
After watching Claude Code, Manus, Deep Research, and OpenClaw all converge
on the same shape, Chase formalised it as the **deep agent**: a
general-purpose harness that gives an LLM more autonomy inside an
environment. The architectural fingerprint is concrete and short:

> "An LLM in a tool-calling loop, connected to a file system, using
> planning and sub-agents."
>
> — Harrison Chase

Five primitives. Loop, tools, file system, planning, sub-agents. Chase's
thesis is that this is now the *standard* shape — not a research
direction, not one of many options, but the architectural pattern that
production deep agents have all converged on. He frames the whole system
as:

```
Model + Harness + Environment = Deep Agent

Model       = frontier / open / fine-tuned LLM
Harness     = loop, planner, tool use, sub-agents, memory, eval integration
Environment = filesystem, shell, browser, APIs, secure runtime, GPU/cloud/local
```

Two of Chase's claims sharpen the rest of this analysis:

1. **Coding-agent patterns *are* the deep-agent harness.** Chase notes
   that coding-specialised models (e.g. Qwen Coder) make better
   general-purpose agent drivers than their broader counterparts, because
   "the harness has a file system; it has a bash tool" — i.e. the
   harness already looks like a coding environment. Coding competence
   is a proxy for tool discipline, file-system navigation, and
   structured action. This validates LocalForge — a coding-specialised
   harness — as a useful substrate for *general* agentic work, not just
   "build me a Next.js app".
2. **Re-architect every nine months.** Chase's sharper enterprise claim
   is that any agent harness more than ~18 months old is probably
   structurally obsolete, and teams should expect to rewrite roughly
   every nine months. The pace at which the canonical components
   (planning, sub-agents, memory, identity) are being defined means
   that "we shipped it last year" is not a defence.

Throughout the rest of this document, when we score LocalForge against
"the canonical pattern", the canon being referenced is Chase's five
primitives plus the enterprise-readiness layer he calls out (trust =
observability + evals + identity + permissions). The twelve-component
table above is the expanded form; Chase's five-element fingerprint is
the irreducible core.

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
| — | **Async sub-agents** *(Chase)* | 🔴 | All Pi sessions are synchronous. The orchestrator spawns a child process and waits; the child runs Pi to completion. There is no manager-checks-on-background-worker pattern, which Chase identifies as the imminent next step for coding harnesses. |
| — | **Always-on / event-driven** *(Chase)* | 🔴 | The harness is purely click-driven via the kanban Start button (with auto-continue between features within one project run). There is no event loop watching a queue, mailbox, GitHub issue tracker, or filesystem; LocalForge does not wake itself up. Chase calls always-on event-driven agents the next major productivity unlock; LocalForge has the substrate for it but not the trigger surface. |
| — | **Agent identity** *(Chase)* | 🔴 | The agent runs as a child process under the user's local OS account, with no separate identity, no credentials of its own, no audit trail keyed to "the agent" rather than "the user". This is fine for a single-user local tool, but the moment two people use the same harness or it touches anything authenticated, the absence becomes structural. |
| — | **Verification depth (evals)** *(Chase)* | 🔴 | Chase's "5–10 eval scenarios is enough to start" framing makes the gap concrete: LocalForge has zero. The optional Playwright check is a runtime smoke test, not an eval suite. There is no `tests/evals/` with input → expected-behaviour pairs, no LangSmith-style scenario library, no comparison run against prior versions of the prompt or harness. |

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

## 8. Planning — does it really work?

**Short answer: it plans once, up front, and never replans.**

LocalForge does have a real planning step, and it is one of the more
interesting harness design choices in the codebase: planning is just
*another Pi session*, but with a tool surface restricted to feature-CRUD
operations. The bootstrapper agent reads the chat transcript and emits one
`create_feature` call per feature, with `depends_on` edges encoding the
dependency graph. That is genuine plan synthesis, not template
expansion — and the dependency edges are honoured by the scheduler at
runtime, which is the half of "planning" most harnesses get wrong.

Where the harness stops short:

- **Plan is final.** Once the backlog is generated, nothing in the runtime
  ever revisits it. There is no monitor that says "three features in a
  row failed, the original decomposition is wrong, replan". The harness
  treats the bootstrapper's output as ground truth.
- **No goal hierarchy.** Features are flat. There are no epics, no
  milestones, no parent/child relationships. A real product is a tree of
  intent; LocalForge stores it as a list with edges.
- **No artefact.** There is no `PLAN.md` written to disk that humans can
  inspect, edit, or that future agents can read. The plan exists only as
  rows in the `features` table. Compare with the
  [awesome-harness](https://github.com/ai-boost/awesome-harness-engineering)
  pattern of milestone-based artefacts (`Plan.md` ↔ `Implement.md`) and
  the now-common Claude-Code convention of an editable plan in the repo.
- **No re-decomposition during execution.** The coding agent's system
  prompt explicitly tells it *not* to ask the user questions and to make
  reasonable assumptions instead. There is no path for it to say
  "this feature is actually three sub-features" and have the harness
  break it apart.
- **No pre-flight check.** The harness does not call back to a planner
  before each feature to ask "given what we've now built, is this
  feature still well-defined?". The plan written at minute zero is the
  plan executed at minute three hundred.

In the canonical taxonomy this is "plan-then-execute", which is the
weakest of the three planning patterns (plan-then-execute → ReAct →
hierarchical-planner-with-replanning). It works for small, self-similar
projects where the bootstrapper got it right the first time. It does not
adapt.

---

## 9. Sub-agents — does it really work?

**Short answer: no. There are no sub-agents.**

This is worth being precise about, because LocalForge *does* run multiple
Pi sessions, and a casual reader could mistake that for a multi-agent
system. It isn't.

The literature distinguishes two patterns, and LocalForge supports
neither:

- **Sub-agents** (hierarchical delegation). A parent agent in the middle
  of its own loop spawns a subordinate agent with its own context window,
  its own system prompt, and a focused brief — "review this PR", "find
  callers of this function", "research API X". The subordinate runs to
  completion and returns a summary that the parent integrates back into
  its own working context. This is the
  [Claude Code Task tool](https://code.claude.com/docs/en/sub-agents)
  pattern (up to 7 parallel sub-agents per main agent), the
  [LangGraph supervisor](https://github.com/langchain-ai/langgraph-supervisor-py)
  pattern, and the [Google ADK](https://developers.googleblog.com/developers-guide-to-multi-agent-patterns-in-adk/)
  hierarchical agent tree.
- **Agents-as-tools** (function-call style). A specialised agent is
  packaged behind a tool schema — `code_review(diff)` returns a review
  object, `web_research(question)` returns a summary. The caller never
  sees the inner agent's loop; from its perspective it called a tool that
  happened to be expensive.

LocalForge has neither, because:

- The Pi session inside `agent-runner.mjs` is given the built-in toolset
  (`read`, `bash`, `edit`, `write`, `grep`, `find`, `ls`) and **nothing
  else**. There is no `delegate(brief)` tool, no `code_review(diff)` tool,
  no `research(question)` tool. Pi cannot invoke another Pi.
- The orchestrator's "concurrent agent slots" are *peer* sessions: each
  one runs a different feature. They share no parent context, do not
  return summaries to one another, and do not participate in any joint
  task. They are siblings, not a supervisor + subordinates.
- Pi itself does have a primitive that could expose sub-agents
  (`customTools` + `defineTool`), and LocalForge already uses it for the
  feature-CRUD tools. The technical lift to wrap a Pi session as a tool
  exists today and is not used.

The consequences are concrete:

- **Context bloat.** Long-running coding sessions accumulate read-file
  outputs, grep results, and bash logs in the same context window the
  model is using to reason. A sub-agent could absorb a 300-line file
  into its own context, return a 5-line summary, and keep the parent's
  context clean. LocalForge has no such mechanism.
- **No specialisation.** Every Pi session is the same generalist agent
  with the same system prompt. There is no "test-writer" Pi, no
  "code-reviewer" Pi, no "researcher" Pi. The prompt has to cover every
  job, which weakens it for each one.
- **No parallelism within a feature.** The harness can run three
  features in parallel via slots, but it cannot run "implement + test +
  document" in parallel within one feature, because there is no parent
  to coordinate the three sub-tasks.

This is the single most impactful gap relative to current state-of-the-
art coding harnesses. Anthropic's Claude Code, OpenAI's Codex/Symphony,
and most production LangGraph deployments rely heavily on sub-agent
delegation; LocalForge has not yet adopted the pattern at all.

**The async-sub-agent direction.** Harrison Chase argues that the next
visible step is *asynchronous* sub-agents: an orchestrator agent spins
up long-running background sub-agents and periodically checks in on them
rather than blocking until they return. LocalForge is structurally close
to enabling this — the orchestrator already manages parallel Pi sessions
in spawned child processes — but the relationship today is *peer
parallelism*, not *manager + workers*. To get to Chase's pattern,
LocalForge would need (a) the synchronous sub-agent primitive (Tier 8)
first, and then (b) a way for the parent session to fire-and-monitor
rather than fire-and-block. The session-state durability work in Tier 1
is the prerequisite for both.

---

## 10. File system access — does it really work?

**Short answer: as a tool surface yes, as a memory substrate no.**

The file system has *two* roles in modern coding harnesses, and they
should be evaluated separately.

### Role 1 — file system as the tool surface (🟢 covered)

This is "the agent reads and writes source code". LocalForge handles this
well:

- The full Pi built-in toolset is enabled for coding sessions: `read`,
  `write`, `edit`, `bash`, `grep`, `find`, `ls`. These are
  schema-validated tools with structured outputs that Pi feeds back as
  observations.
- The runner sets `cwd` to the project folder
  (`projects/<slug>/`) and the system prompt asserts that this is the
  workspace.
- Every `write`/`edit`/`bash` is checked by the workspace-guard
  extension (`makeWorkspaceGuardExtension`) before execution. Paths are
  resolved (with MSYS-style normalisation on Windows), and anything
  resolving outside the project root is rejected. Bash commands are
  scanned for absolute paths that would escape the workspace, with
  exemptions for `/dev/null`, `/tmp/`, `/usr/`, etc.
- Broad `node`-killers (`taskkill /F /IM node.exe`, `killall node`,
  certain `pkill -f node` shapes) are blocked because they would crash
  the harness server itself.

This is genuine harness work — not just "we let the model write files",
but "we let the model write files and we check every path against a
containment policy first". Many harnesses don't do the second half.

### Role 2 — file system as the memory substrate (🔴 not covered)

This is the role the harness literature has converged on hardest in
2025-2026. Anthropic shipped a [memory tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
that gives the model a directory it can `create`, `read`, `update`,
`delete` in across sessions. Claude Code maintains `CLAUDE.md`,
`AGENTS.md`, and a per-project notes convention. Anthropic's
"[Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)"
post and the "Auto Dream" memory consolidation feature both treat the
file system as the agent's writable, persistent state — not as the code
under construction, but as the agent's *own* knowledge base. The pattern
has a name in the awesome-harness taxonomy ("filesystem as foundational
memory") and is at this point table-stakes.

LocalForge does not implement any of this:

- There is **no agent-writable memory file**. No `AGENT_NOTES.md`, no
  `CLAUDE.md`, no `.localforge/notes/`. The agent can `write` such a
  file (the tool layer doesn't forbid it), but no future Pi session is
  configured to read it. The system prompt is built from static
  templates and the project's `coder_prompt` setting; project-folder
  files are not auto-injected.
- There is **no skills directory wired in**. The repo ships a
  `.agents/skills/` folder (frontend-design, localforge, playwright-cli,
  skill-creator) but the runner explicitly tells Pi `noSkills: true`.
  These files are inert from the agent's perspective.
- There is **no consolidation** ("Auto Dream"). Even if the agent did
  write notes, there is no scheduled compaction step that would prune
  stale entries, merge duplicates, or summarise lessons.
- The **project's own files are not used as agent memory either**.
  Pi sees the source code as the *target*, not as a memory of decisions.
  There is no `DECISIONS.md`, no `RUNBOOK.md`, no `.agent-state.json`
  that the harness would surface to subsequent runs.
- The **chat_messages table** persists bootstrapper conversation but is
  not surfaced to the coding agent. Once the backlog is generated, the
  coding agent never sees the original user intent.

**Why this matters more than the other gaps.** Memory and the file
system are the two things that turn a stateless tool-loop into something
that learns. Anthropic's framing is that "files are a new kind of
memory" — not metaphorically, literally — and modern long-running
harnesses treat the workspace directory as a hybrid of "code under
construction" and "agent's notebook". LocalForge has the former and not
the latter.

Chase's deep-research anecdote is the cleanest illustration: when
LangChain first prototyped a deep agent, they "gave it access to a
bunch of files, and we just put it in this virtual file system and had
it do some research. And it wasn't even really doing RAG. It was just
grepping and globbing like a coding agent would over these files. And
it worked fantastically well." The file system *is* the memory.
LocalForge has all the grep/glob/read tooling already wired up; what's
missing is the convention that some files in the project folder are the
agent's notes, and the harness is responsible for maintaining and
re-presenting them on each run.

The fix is small — Tier 2 in the recommendations below covers it — but
the absence is structural.

---

## 11. Honest critique

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
7. **It is, by Chase's calendar, due for re-architecture.** Harrison
   Chase's working assumption is that any agent harness more than
   ~18 months old is structurally obsolete and should be expected to be
   rewritten roughly every nine months. LocalForge's design predates the
   "deep agent" canonical pattern crystallising; some of what's missing
   here (sub-agents, agent memory, async, identity) is missing
   specifically because those primitives weren't yet considered
   table-stakes when LocalForge was designed. That isn't a flaw in
   LocalForge — it's a property of the field. But it is a reason to
   treat the recommendations below as urgent rather than aspirational.

The *integrity* of the harness is good. The *ambition* is small.

---

## 12. Recommendations — making LocalForge a real long-running, self-evolving harness

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
6. **Add a global skill library — the "Markdown + scripts" form.**
   Re-enable Pi's `noSkills: false` and adopt the existing
   `.agents/skills/` folder structure that already ships in the repo.
   Per Chase's framing, a skill is not a prompt — it's a Markdown file
   with instructions plus executable scripts (Python that hits a URL,
   bash that runs a build, optionally GPU-accelerated jobs). The agent
   chooses which skills to invoke; the scripts are the deterministic
   part. Curate a small starter set (workspace-guard-friendly bash
   patterns, common Next.js fixes, Drizzle idioms, dev-server reset)
   and let users add more.
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

### Tier 8 — sub-agents (close the biggest single gap)

23. **Wrap a Pi session as a tool.** Define a `delegate(brief, tools[])`
    custom tool that, when called by the parent Pi session, spawns a
    fresh Pi session with its own context window, system prompt, and
    restricted toolset, runs it to completion, and returns the final
    assistant text as the tool result. This is the
    [Claude Code Task tool](https://code.claude.com/docs/en/sub-agents)
    pattern reproduced inside LocalForge.
24. **Specialised sub-agent profiles.** Ship a small set of opinionated
    sub-agent configs the parent can target by name: `code-reviewer`
    (read-only, no bash, gets a diff), `test-writer` (read + write to
    `tests/` only), `researcher` (read + grep + bash, no write). Each
    has a focused system prompt that the generalist coding agent can
    delegate to.
25. **Parallel sub-agents within one feature.** Allow the parent to
    issue multiple `delegate` calls concurrently. Today's "agent slots"
    are inter-feature parallelism; this is intra-feature parallelism, and
    it's where the wall-clock wins compound. Cap at 3 concurrent
    sub-agents per parent to match the existing concurrency cap.
26. **Sub-agent context isolation.** Each delegated session gets its own
    in-memory `SessionManager`. The parent never sees the sub-agent's
    raw tool outputs — only the final summary. This is the same
    "absorb-then-summarise" trick that keeps Claude Code's main
    conversation clean during deep file searches.

### Tier 9 — file system as memory substrate

27. **Auto-include `AGENT_NOTES.md` in the system prompt.** Tier 2
    item #5 introduces the file; this item makes the harness *read* it
    on every coding-session start and prepend it to the system prompt
    behind a clearly-labelled section. Anthropic's
    [memory tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
    is the same idea formalised.
28. **Auto-Dream consolidation pass.** A scheduled (or
    on-project-completion) Pi session that reads the project's notes,
    failure summaries, and per-feature lessons and rewrites them into
    a deduplicated, contradiction-free knowledge file. Anthropic's
    "Auto Dream" is the reference. The cost is one extra Pi run per
    project; the benefit is that the knowledge file stays useful as it
    grows.
29. **Surface bootstrapper chat to coding sessions.** Today the
    `chat_messages` rows are dropped after the backlog is generated.
    Append a short summary of the original user intent to every coding
    session's prompt — the agent should know *why* a feature exists, not
    just *what* it is.

### Tier 10 — async sub-agents (Chase's near-term direction)

Builds on Tier 8 (synchronous sub-agents). Implement these only after
Tier 8 lands.

30. **Fire-and-monitor delegation.** Add a `delegate_async(brief, …)`
    tool that returns a handle (`subagent_id`) immediately rather than
    blocking. The parent continues its loop and can call
    `check_subagent(subagent_id)` to read partial progress or
    `await_subagent(subagent_id)` when it's ready to consume the
    result. This is the literal pattern Chase predicts will dominate
    coding harnesses next.
31. **Background-task table.** Add `background_tasks(id, parent_session_id,
    subagent_session_id, brief, status, progress_summary, started_at,
    finished_at)` so the orchestrator can render a "what's running in
    the background" panel and so the parent's `check_subagent` call
    returns durable state, not just in-memory state.
32. **Manager UI mode.** Optional second mode for the kanban: instead
    of "you click Start, agents work the backlog", the user chats with
    a single orchestrator agent ("how's feature 12 going? please add
    tests for the auth flow") that owns the backlog and dispatches
    sub-agents. Closer to Chase's "talk to the orchestrator, not the
    coder" pattern.

### Tier 11 — always-on / event-driven (Chase's biggest predicted unlock)

33. **Trigger surface.** Add a small dispatcher process that watches
    one or more event sources — local file (drop a brief into
    `inbox/`), GitHub webhook, a polled HTTP endpoint, even
    `chokidar` on a folder — and starts an orchestrator session per
    event. The body of the trigger becomes the bootstrapper input.
    The harness wakes itself up.
34. **Approval-gated reply pattern.** For event sources that produce
    user-facing replies (email, Slack, GitHub PR comments), the agent
    drafts the response, the harness shows it for one-click approve or
    edit before it goes out. Mirrors Chase's email-agent example.
    Reuses the HITL gate from Tier 7.
35. **Cost-class routing.** The orchestrator picks a smaller / cheaper
    model for the high-frequency event paths and the configured
    "premium" model only when the event escalates. This is also where
    the Nemotron-coalition pattern (frontier orchestrator + open
    sub-agents) lives — see Tier 12.

### Tier 12 — agent identity & credentials

This is the enterprise blocker Chase calls out specifically: *whose
credentials does the agent use?* For a single-user local tool today,
the answer is "the user's" by default, which is fine. The moment the
harness is shared, it stops being fine.

36. **Per-agent identity record.** Add `agent_identities(id, name,
    description, owner_user, created_at)` and require every spawned
    Pi session to be associated with an identity (defaulting to a
    "system" identity for backward compatibility). Surface the
    identity in every log line and SSE event.
37. **Scoped credentials per identity.** Move provider auth (LM Studio
    API key, future MCP-server auth, future GitHub tokens for the
    event trigger) from "ambient process env" to "looked up by agent
    identity at session start". Identities have credentials; processes
    don't.
38. **Audit trail keyed by identity.** Every `agent_logs` row gets an
    `identity_id` foreign key. The activity panel can filter by
    identity. Compliance / "what did Tom the marketing agent do
    yesterday?" becomes a query, not a forensic trawl.
39. **Revocation primitive.** A single setting that disables an
    identity, blocks any in-flight session running under it, and
    refuses new sessions until re-enabled. Cheap to implement, very
    expensive *not* to have when something goes wrong.

### Tier 13 — evaluation-driven development (start small)

Per Chase: 5–10 scenarios is enough to start. Don't wait for 1,000.

40. **`tests/evals/` directory.** Each eval is a JSON file:
    `{ project_brief, expected_features_min, expected_files_present,
    expected_build_passes, max_turns_total }`. The harness has an
    `npm run eval` script that spins up a clean project per scenario,
    runs the bootstrapper + orchestrator, asserts the expected
    outcomes, and writes a one-line pass/fail to stdout.
41. **Eval as CI gate.** Block merges to main when fewer than N of the
    eval scenarios pass. Cheap protection against prompt regressions —
    which are otherwise invisible until users complain.
42. **Eval expansion from production.** When a real LocalForge run does
    something surprising — good or bad — the activity panel gets a
    "save as eval scenario" button that captures the chat, prompt
    settings, and observed outcome. The eval suite grows from real
    use, exactly as Chase describes.

---

## 13. Bottom line

LocalForge is a **real harness** — narrow, correct, and observable, with a
genuine permission layer and a clean separation between the model SDK (Pi)
and the harness (everything else). Its scope today is "build a small app
end-to-end on a local model with auto-continue", and within that scope it
works.

By the canonical pattern Harrison Chase identifies — *loop + tools +
file system + planning + sub-agents* — LocalForge has the loop and the
tools, has only the *target-code* half of the file system, has planning
that finalises at minute zero, and has no sub-agents at all. Three out
of five primitives are partially or fully missing.

To honestly call itself a "long-running, self-evolving deep agent
harness" it needs the following, in this order:

1. **Durable Pi sessions** (Tier 1) — agents survive restarts.
2. **A writable agent memory + failure feedback loop** (Tier 2) — agents
   leave behind something useful for the next agent.
3. **Real verification gates + a small eval suite** (Tier 3 + Tier 13)
   — the harness can tell the difference between a good run and a bad
   one. Per Chase: start with 5–10 scenarios, not 1,000.
4. **Synchronous sub-agent delegation** (Tier 8) — closes the biggest
   single gap relative to current SOTA coding harnesses.
5. **File system as a memory substrate, not just a tool surface**
   (Tier 9) — the canonical pattern.
6. **MCP and per-tool permissions** (Tier 5) — third-party tool surface.

Tiers 4, 6, 7, 10, 11, 12 are the next horizon — better planning,
distributed control plane, HITL polish, async sub-agents, always-on /
event-driven triggers, and agent identity. These are what take LocalForge
from "complete narrow harness" to "Chase-shaped deep-agent platform".
Each is worth doing, but only after the six load-bearing items above
land — each of those is a prerequisite for the next.

The blunt summary: LocalForge today is a competent *narrow* harness in a
field that is rapidly standardising on a *deep* one. Closing that gap is
not a research project — every primitive that's missing has a working
reference implementation in the public domain (Claude Code, LangGraph,
Anthropic's memory tool, OpenClaw, Symphony). The work is integration,
not invention.

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
- [Where to use sub-agents versus agents as tools — Google Cloud](https://cloud.google.com/blog/topics/developers-practitioners/where-to-use-sub-agents-versus-agents-as-tools/)
- [Create custom subagents — Claude Code Docs](https://code.claude.com/docs/en/sub-agents)
- [Subagents in the SDK — Claude API Docs](https://platform.claude.com/docs/en/agent-sdk/subagents)
- [LangGraph Supervisor — GitHub](https://github.com/langchain-ai/langgraph-supervisor-py)
- [Developer's guide to multi-agent patterns in ADK — Google](https://developers.googleblog.com/developers-guide-to-multi-agent-patterns-in-adk/)
- [Memory tool — Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
- [Effective context engineering for AI agents — Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Using agent memory — Claude API Docs](https://platform.claude.com/docs/en/managed-agents/memory)
- [Harrison Chase on deep agents, async sub-agents, and agent identity — NVIDIA AI Podcast (2026)](https://www.youtube.com/watch?v=c-fsL0gsmo0) — primary source for the deep-agent fingerprint, the 9-month re-architecture cadence, the skills-as-Markdown-plus-scripts framing, and the always-on / agent-identity directions cited throughout this document
- [Deep Agents — LangChain blog](https://blog.langchain.com/) (search "deep agents")
