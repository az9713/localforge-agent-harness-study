# Pi inside LocalForge

> **TL;DR** — Pi is the brain. LocalForge is the harness around it.
>
> Pi (`@mariozechner/pi-coding-agent`) is an embeddable coding-agent SDK: it
> owns the LLM tool-loop, the prompt format, and the file/bash/grep tool
> implementations. LocalForge owns the kanban, the SQLite state, the project
> folders, the orchestrator, and the live-log UI. Whenever LocalForge needs
> an LLM to actually *do something* — generate the backlog, or implement a
> feature — it spins up a Pi `AgentSession`, points it at a local model, and
> streams the events back into the UI. There is no LocalForge-specific agent
> loop. **Pi is the agent.**

---

## What is Pi?

Pi is the open-source coding agent published as the npm package
[`@mariozechner/pi-coding-agent`](https://www.npmjs.com/package/@mariozechner/pi-coding-agent).
It is the same core engine that powers the standalone `pi` CLI; the SDK form
exposes the runtime so other applications can embed it.

Pinned in `package.json`:

```json
"@mariozechner/pi-coding-agent": "^0.70.2"
```

Pi gives LocalForge five things it does not have to build itself:

| Pi provides | What that means in practice |
|---|---|
| `AgentSession` | The turn-based LLM tool-loop — prompt → tool calls → tool results → next prompt → … until the model says it is done |
| `ModelRegistry` + `AuthStorage` | Provider/model wiring with a pluggable auth layer |
| `DefaultResourceLoader` | Loads the system prompt, agent dir, skills, themes, etc. |
| `SessionManager` | Per-cwd session state and message history |
| Built-in tools | `read`, `write`, `edit`, `bash`, `grep`, `find`, `ls` — production-quality file and shell tools with sane defaults |

LocalForge does **not** fork Pi, patch it, or wrap a private copy. It uses
Pi as a library, configured per session.

---

## The two places Pi is invoked

LocalForge calls into Pi from exactly two places. Everything else — the
kanban, the project folder, the SQLite schema, the SSE log feed — is pure
LocalForge code that reacts to events Pi emits.

### 1. Bootstrapper — turning chat into a backlog

**Where:** `app/api/agent-sessions/[id]/generate-features/route.ts`
**When:** the user finishes the bootstrapper chat and clicks "Generate
features".
**What Pi sees:** the chat transcript so far, plus the
`FEATURE_GEN_SYSTEM_PROMPT` instructing it to call feature CRUD tools.
**Tools available:** **none of Pi's built-ins.** The route disables them
with `noTools: "builtin"` and substitutes a small set of custom tools
(`buildFeatureCrudTools(projectId)`) that talk straight to the LocalForge
SQLite layer:

```ts
const { session: piSession } = await createAgentSession({
  cwd: project.folderPath,
  authStorage: piRuntime.authStorage,
  modelRegistry: piRuntime.modelRegistry,
  model: piRuntime.model,
  thinkingLevel: "off",
  sessionManager: SessionManager.inMemory(project.folderPath),
  resourceLoader,
  noTools: "builtin",
  customTools: buildFeatureCrudTools(project.id),
});
```

This is Pi as a **structured-output engine**: every "feature" it creates is
a tool call, validated by the LocalForge tool layer, persisted in SQLite.
Pi never touches the filesystem in this mode.

The session runs **inside the Next.js API route process** (no spawn), with
`maxDuration = 600` seconds and a hard cap of 40 turns.

### 2. Coding agent — implementing one feature

**Where:** `scripts/agent-runner.mjs`, spawned by `lib/agent/orchestrator.ts`
**When:** a user clicks **Start** on the kanban (or the orchestrator
auto-continues after a previous feature finishes).
**What Pi sees:** the system prompt built by `buildCodingSystemPrompt()`
plus a user message containing the feature's title, description, and
acceptance criteria.
**Tools available:** the **full Pi built-in toolset**, with one custom
extension:

```js
tools: ["read", "bash", "edit", "write", "grep", "find", "ls"]
```

Plus a workspace-guard extension (`makeWorkspaceGuardExtension`) that
blocks any `write`/`edit`/`bash` whose target path resolves outside the
project folder. This is a Pi extension, not a fork — it hooks the
`tool_call` event and returns `{ block: true, reason }` when needed.

This is Pi as a **real coding agent** — it reads files, edits source,
runs `npm install`, fixes type errors, etc. The agent runs in a separate
Node.js child process so its long-running work cannot block the Next.js
server.

---

## End-to-end: where Pi sits in the request flow

```
                         ┌──────────────────────────────────────┐
                         │  Browser (Next.js client)            │
                         └──────────────┬───────────────────────┘
                                        │ click Start
                                        ▼
                         ┌──────────────────────────────────────┐
                         │  Next.js API route                   │
                         │  /api/projects/:id/orchestrator/...  │
                         └──────────────┬───────────────────────┘
                                        │
                                        ▼
                         ┌──────────────────────────────────────┐
                         │  lib/agent/orchestrator.ts           │
                         │  • picks ready feature               │
                         │  • spawns child process              │
                         └──────────────┬───────────────────────┘
                                        │ spawn(node, [agent-runner.mjs, …])
                                        ▼
                         ┌──────────────────────────────────────┐
                         │  scripts/agent-runner.mjs (child)    │
                         │  ┌────────────────────────────────┐  │
                         │  │ Pi: createAgentSession({...})  │  │
                         │  │   model + tools + extensions   │  │
                         │  │   .prompt(userPrompt)          │  │
                         │  │   .subscribe(event => …)       │  │
                         │  └─────────────┬──────────────────┘  │
                         │                │ HTTP                │
                         │                ▼                     │
                         │   ┌──────────────────────────────┐   │
                         │   │ LM Studio / Ollama (OpenAI-  │   │
                         │   │ compatible /v1/chat …)       │   │
                         │   └──────────────────────────────┘   │
                         └──────────────┬───────────────────────┘
                                        │ stdout JSON-lines (one per event)
                                        ▼
                         ┌──────────────────────────────────────┐
                         │  orchestrator parses lines →         │
                         │  agent_log rows + SSE broadcast      │
                         └──────────────┬───────────────────────┘
                                        │ SSE
                                        ▼
                         ┌──────────────────────────────────────┐
                         │  Browser activity panel (live)       │
                         └──────────────────────────────────────┘
```

The Pi → orchestrator → SSE chain is the entire reason the activity panel
shows live tool calls — Pi emits `tool_execution_start`, the runner converts
it to a JSON line on stdout, the orchestrator broadcasts it.

---

## How Pi is configured for local models

Pi expects a `Model` object describing a provider, a base URL, an auth
strategy, and capability flags. LocalForge constructs one in two parallel
places — `lib/agent/pi-model-config.ts` (used by the in-process bootstrapper
session) and inline inside `scripts/agent-runner.mjs` (used by the spawned
coding session). Both produce the same shape:

```ts
{
  id: "<model name>",
  name: "<model name>",
  api: "openai-completions",       // LM Studio + Ollama both speak this
  provider: "lm_studio" | "ollama",
  baseUrl: "<base>/v1",            // we always normalise to /v1
  reasoning: false,
  input: ["text"],
  cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
  contextWindow: 128000,
  maxTokens: 16384,
  compat: {
    supportsDeveloperRole: false,
    supportsReasoningEffort: false,
    supportsUsageInStreaming: false,
    supportsStrictMode: false,
    maxTokensField: "max_tokens",
  },
}
```

Two things to notice:

1. **Cost is zero across the board.** Pi's accounting subsystem still runs
   and still tallies "spend", but every line item is multiplied by zero so
   the UI reports `$0.00`. This is the cleanest way to keep Pi's accounting
   on the rails without lying about token counts.
2. **`compat` flags are deliberately conservative.** Local models often
   advertise OpenAI-compatible APIs but quietly drop or mishandle features
   like the `developer` role, `reasoning_effort`, streaming-usage events,
   or strict tool mode. Turning those off prevents cryptic failures on
   first-token.

Auth is satisfied with an in-memory key:

```ts
const authStorage = AuthStorage.inMemory();
authStorage.setRuntimeApiKey(localModel.provider, "localforge");
```

The string `"localforge"` is meaningless to LM Studio (which doesn't check
the key) but Pi requires *some* value, so we hand it a constant. There is
no real credential to leak.

---

## How LocalForge customises Pi at runtime

Pi exposes hook points. LocalForge uses three of them.

### `extensionFactories` — workspace guard

Coding sessions register a single extension (`makeWorkspaceGuardExtension`)
that hooks `tool_call`:

```js
return function workspaceGuardExtension(pi) {
  pi.on("tool_call", (event) => {
    const reason = check(event.toolName, event.input);
    if (!reason) return undefined;
    onDenied?.(event.toolName, reason);
    return { block: true, reason };
  });
};
```

It rejects any `write`, `edit`, or `bash` whose paths escape the project
directory, and it specifically blocks broad process-killers like
`taskkill /F /IM node.exe` that would terminate the LocalForge server
itself. The guard understands MSYS-style paths (`/c/Users/...`) on Windows
so Git-Bash invocations don't bypass it.

### `customTools` + `noTools: "builtin"` — feature CRUD as tools

The bootstrapper route disables Pi's built-in tools entirely and registers
its own. The result is that Pi *cannot* edit files or run shell commands
during feature generation — its only output channel is calling
`create_feature` or `list_features`.

### `subscribe(event => …)` — observability

Pi emits a stream of events:

| Pi event | LocalForge does this |
|---|---|
| `tool_execution_start` | Render "Reading X" / "Editing Y" / "Running: Z" in the activity panel |
| `message_update` (`text_delta`) | Buffer assistant prose for later summarisation |
| `message_end` | Detect `stopReason === "error"` / `"aborted"` |
| `turn_end` | Bump turn counter; abort if `LOCALFORGE_MAX_TURNS` is exceeded |

This is the only reason the UI feels live. There is no separate progress
mechanism inside LocalForge — every "the agent is doing X" line you see is
a Pi event in disguise.

---

## When Pi is *not* invoked

It's just as important to know what does **not** call Pi:

- **Manual feature creation** (typing into the kanban "Add feature" form).
  Plain SQL insert.
- **Bootstrapper chat itself** (the back-and-forth conversation with the
  AI before "Generate features"). That uses a much simpler streaming
  chat-completions call to the same provider — it is not a Pi
  `AgentSession` because there are no tools to call yet.
- **Playwright verification** that runs after a feature finishes. The
  runner imports `chromium` from `@playwright/test` directly and drives the
  browser in process. Pi has already exited by then.
- **Auto-continue between features.** The orchestrator's
  `maybeContinueWithNextFeature()` is plain Node.js — it just calls
  `startOrchestrator(projectId)` again, which spawns a fresh runner, which
  starts a fresh Pi session.

Each Pi session is single-purpose, single-feature, and disposable. There
is no long-lived Pi process.

---

## Lifecycle of a single Pi coding session

A useful way to internalise the integration is to walk one feature through
the system from the moment the user clicks **Start**:

1. **Pick.** `startOrchestrator(projectId)` finds the highest-priority
   ready feature and flips it to `in_progress` (SQLite). No Pi yet.
2. **Persist.** `createAgentSession({...})` creates an `agent_sessions`
   row. Still no Pi.
3. **Spawn.** The orchestrator spawns `node scripts/agent-runner.mjs ...`
   with the feature context written to a temp JSON file. Still no Pi —
   the runner is just a Node script.
4. **Wire Pi up.** Inside the child, `createPiModelRuntime()` builds the
   model registry, `DefaultResourceLoader` loads the system prompt, and
   `createAgentSession({...})` from `@mariozechner/pi-coding-agent` returns
   a real Pi session object. **This is the moment Pi enters the stack.**
5. **Subscribe.** The runner attaches a `subscribe()` listener so every
   tool call and message becomes a stdout JSON line.
6. **Prompt.** `session.prompt(userPrompt, ...)` kicks off the tool loop.
   Pi is now in charge — it queries the local model, gets back tool calls,
   executes them, feeds results back, repeats until the model emits a
   final assistant message with no tool calls.
7. **Dispose.** `session.dispose()` releases Pi's resources. Pi is gone.
8. **Verify (optional).** If `LOCALFORGE_PLAYWRIGHT_ENABLED=true`, the
   runner spins up Chromium directly, navigates to the project's dev
   server, takes a screenshot, and emits a `test_result` log line. **Pi
   does not participate in this phase.**
9. **Report.** The runner writes `{"type":"done","outcome":"success"}` to
   stdout and exits. The orchestrator's `child.on("close")` flips the
   feature to `completed` and broadcasts the final status event. Auto-
   continue may then start the cycle again for the next feature.

The runner can retry the Pi session on transient errors (network blips,
"Failed to generate a valid tool call", 5xx from the provider). Each
retry is a fresh Pi `AgentSession` — Pi has no persistent state between
retries beyond what the prompt rebuilds.

---

## Why split into two Pi sessions instead of one giant one?

Because the two jobs have completely different shapes:

| Job | State surface | Tools needed | Failure consequence |
|---|---|---|---|
| Generate features | Chat transcript + SQLite | `create_feature`, `list_features` (custom) | "Try again" — cheap to redo |
| Implement feature | One backlog row + project folder | `read`, `write`, `edit`, `bash`, `grep`, `find`, `ls` | Re-runs cost minutes of LLM time |

Running them as two distinct Pi sessions means the bootstrapper cannot
accidentally `rm -rf` your project folder, and the coding agent cannot
silently corrupt the backlog by calling a feature CRUD tool. Each session
gets exactly the tools it needs, and nothing more.

---

## Where to look in the code

| Concern | File |
|---|---|
| Pi model config (shared shape) | `lib/agent/pi-model-config.ts` |
| Pi runtime + resource loader (server) | `lib/agent/pi-runtime.ts` |
| Bootstrapper Pi invocation | `app/api/agent-sessions/[id]/generate-features/route.ts` |
| Custom feature CRUD tools | `lib/agent/feature-crud-tools.ts` |
| Coding-agent runner (spawned child) | `scripts/agent-runner.mjs` |
| Orchestrator (spawn + lifecycle) | `lib/agent/orchestrator.ts` |
| Provider config | `lib/agent/providers/` |

---

## Frequently de-mystified

- **"Where does the system prompt come from?"** Two functions:
  `FEATURE_GEN_SYSTEM_PROMPT` (a string constant) for bootstrapping, and
  `buildCodingSystemPrompt(projectDir, coderPrompt, devServerPort, …)` for
  coding. Pi's `DefaultResourceLoader` is told `noSkills`,
  `noPromptTemplates`, `noThemes`, `noExtensions` so it does **not** add
  anything from `~/.pi`. The prompt LocalForge sends is the prompt Pi sees.
- **"Does Pi see my real Anthropic / OpenAI keys?"** No. The auth storage
  is in-memory and the key is the literal string `"localforge"`. Pi only
  ever talks to your local LM Studio / Ollama endpoint.
- **"Why does the activity panel say 'Reading package.json' instead of
  Pi-internal jargon?"** `summariseToolUse(name, input)` in
  `agent-runner.mjs` translates Pi tool events into human-readable lines
  before they hit stdout.
- **"What if I want to see Pi's raw event stream?"**
  `agent-runner-debug.log` at the repo root captures the full event
  trail per session, written by `debugLog()`.
- **"Can the coding agent escape the project folder?"** Not via tools —
  the workspace-guard extension blocks every `write`/`edit`/`bash` with a
  path outside the workspace, including MSYS-form paths on Windows. It
  also blocks broad `node` killers that would take down LocalForge itself.
- **"Where is the Pi session's `cwd`?"** The project folder
  (`project.folderPath`). Both Pi's `DefaultResourceLoader` and the spawned
  child process use this as cwd, which is why relative paths in tool calls
  resolve to the right place.
- **"What ends a Pi session?"** Any of: the model emits a final assistant
  message with no further tool calls; `LOCALFORGE_MAX_TURNS` is exceeded;
  the orchestrator's watchdog fires (`SESSION_TIMEOUT_MS`); the user clicks
  Stop; the runner gets `SIGTERM`/`SIGINT`; `session.abort()` is called for
  any other reason.
