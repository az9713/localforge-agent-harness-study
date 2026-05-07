# Bootstrapper

The bootstrapper turns a conversational app idea into validated LocalForge feature rows.

## What it solves

Users often start with an app concept, not a dependency-ordered backlog. The bootstrapper keeps the early conversation lightweight, then runs a feature-generation pass that writes structured feature data through safe tools.

## Chat phase

Bootstrapper sessions are created by `POST /api/projects/:id/bootstrapper-session`. The route is idempotent: if a project already has an active bootstrapper session, it returns the existing session instead of creating a duplicate.

Messages are handled by `app/api/agent-sessions/[id]/messages/route.ts`.

| Event | Storage | Streaming behavior |
|-------|---------|--------------------|
| User sends a message | User row is inserted into `chat_messages` first | Client receives a `user` SSE event |
| Provider streams a reply | Assistant text is accumulated in memory | Client receives repeated `delta` events |
| Reply finishes | Assistant row is inserted into `chat_messages` | Client receives `assistant`, then `done` |
| Provider fails | User row remains persisted | Client receives `error`; no assistant row is saved |

The chat uses the effective provider config for the project. It does not use the Pi coding-agent tools during normal conversation; it streams chat completion text through the local provider helper.

## Feature-generation phase

The "Generate feature list" action calls `POST /api/agent-sessions/:id/generate-features`.

That route creates a Pi AgentSession with:

| Runtime setting | Value |
|-----------------|-------|
| Current working directory | The project folder |
| Built-in tools | Disabled with `noTools: "builtin"` |
| Custom tools | LocalForge feature CRUD tools |
| Context files | Disabled with `noContextFiles: true` |
| Turn cap | 40 turns |

This means the feature-generation agent cannot edit files or run shell commands. It can only call validated tools such as `list_features`, `create_feature`, and `add_dependency`.

## Tool boundary

The custom tools live in `lib/agent/feature-crud-tools.ts`.

| Tool | Purpose |
|------|---------|
| `list_features` | Inspect the existing project backlog before creating new rows |
| `create_feature` | Add a feature with title, description, category, and optional dependencies |
| `update_feature` | Change feature metadata |
| `delete_feature` | Remove a feature |
| `add_dependency` | Add one prerequisite edge |
| `remove_dependency` | Remove one prerequisite edge |
| `set_dependencies` | Replace all prerequisites for a feature |

Every mutation still passes through `lib/features.ts`, so the same validation rules apply to UI, API, and AI-generated backlog changes.

## Completion behavior

After the Pi session finishes, the route counts features before and after generation.

| Outcome | HTTP behavior | Session status |
|---------|---------------|----------------|
| At least one feature was created | Returns `count`, `total`, `projectId`, `toolCalls`, and `summary` | Bootstrapper session is closed as `completed` |
| No features were created | Returns 502 with agent summary and tool count | Session remains available for retry |
| Pi/provider error | Returns 502 with error message | Session remains available for retry |

This keeps failed generation recoverable: the user can clarify the idea and try again without losing the transcript.
