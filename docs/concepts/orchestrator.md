# Orchestrator

The orchestrator is the server-side scheduler that turns ready backlog features into running coding-agent child processes.

## What it solves

LocalForge needs a coordinator that understands project state, dependency order, process handles, retries, UI updates, and server restarts. The orchestrator owns that coordination while leaving actual file editing to `scripts/agent-runner.mjs`.

## Core responsibilities

`lib/agent/orchestrator.ts` owns these responsibilities:

| Responsibility | Behavior |
|----------------|----------|
| Feature selection | Choose the highest-priority ready backlog feature |
| Session creation | Insert an `agent_sessions` row for each coding run |
| Process spawn | Start `scripts/agent-runner.mjs` with provider, model, project, and feature context |
| Event parsing | Convert runner stdout JSON lines into persisted logs and SSE events |
| Finalization | Mark features completed, failed, demoted, or returned to backlog |
| Auto-continue | Fill available agent slots after success or failure |
| Recovery | Reap orphan session rows and reset orphaned `in_progress` features |

## Start flow

`startOrchestrator(projectId)` performs this sequence:

1. Load the project row.
2. Close orphaned coding sessions that are in the database but not in the in-process running map.
3. Reset orphaned `in_progress` features that no live child process owns.
4. Check the configured per-project concurrency limit.
5. Find the next ready feature.
6. Mark the feature `in_progress`.
7. Create an `agent_sessions` row.
8. Append and broadcast a startup log.
9. Write a temporary prompt JSON file.
10. Spawn the runner with Node.js.
11. Attach stdout, stderr, error, close, and watchdog handlers.

## Concurrency

The hard cap is `MAX_CONCURRENT_AGENTS_HARD_CAP`, currently 3. The effective project limit comes from the `max_concurrent_agents` setting and is clamped to the valid range.

`startAllAgents(projectId)` fills every available slot by repeatedly calling `startOrchestrator()` until no slot or no ready feature remains.

## Event model

The orchestrator stores events in SQLite and also emits them in memory.

| Event type | Meaning | UI subscriber |
|------------|---------|---------------|
| `log` | Agent or runner output | Activity panel and agent pods |
| `status` | Session or feature state changed | Kanban refresh and slot refresh |
| `project_completed` | All features completed | Celebration screen |

Two SSE routes expose these events:

| Route | Scope |
|-------|-------|
| `GET /api/agent/events` | All orchestrator events for global notifications and project view updates |
| `GET /api/agent/stream/:sessionId` | One session, with stored log replay on connect |

## Finalization

`finalizeSession()` is idempotent. It removes the session from the running map, clears the watchdog, deletes the temporary prompt file, closes the session row, broadcasts final status, and then decides whether to continue.

| Runner outcome | Feature result | Session result | Next action |
|----------------|----------------|----------------|-------------|
| `success` | `completed` | `completed` | Check project completion, then maybe continue |
| `failed` | Back to `backlog` with lower priority | `failed` | Maybe continue with other ready work |
| `terminated` | Back to `backlog` without demotion | `terminated` | Stop auto-continue |

## Why the singleton exists

Next.js development mode can reload modules while child processes and SSE listeners are still active. The orchestrator stores its runtime state on `globalThis` under a symbol so hot reloads do not instantly lose the running map and event emitter.

That singleton is runtime-only. Durable state still belongs in SQLite.
