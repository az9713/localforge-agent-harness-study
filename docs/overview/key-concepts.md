# Key concepts

**Agent log** - A persisted event from a bootstrapper or coding session. Example: an `action` log records that the agent edited a file; a `test_result` log records the latest Playwright pass/fail summary.

**Agent runner** - The Node.js child process started by the orchestrator for one coding feature. It lives in `scripts/agent-runner.mjs`.

**Agent session** - A database row representing one bootstrapper conversation or one coding run. Session types are `bootstrapper` and `coding`; statuses are `in_progress`, `completed`, `failed`, and `terminated`.

**Bootstrapper** - The chat workflow that helps a user describe an app and then converts that transcript into LocalForge feature rows.

**Dependency** - A relationship where one feature cannot start until another feature is completed. Dependencies are stored in `feature_dependencies`.

**Feature** - One work item in a project backlog. A feature has a title, optional description, optional acceptance criteria, status, priority, and category.

**Feature category** - A feature classification used by the UI and generation prompt. Current values are `functional` and `style`.

**Feature status** - The kanban state for a feature. Current values are `backlog`, `in_progress`, and `completed`.

**Local model provider** - A local inference server that exposes a compatible API. Current provider IDs are `lm_studio` and `ollama`.

**Orchestrator** - The server-side coordinator that chooses ready features, starts runner processes, handles process completion, updates feature/session state, and broadcasts events.

**Pi coding-agent** - The `@mariozechner/pi-coding-agent` runtime used to drive file reads, edits, shell commands, and model calls inside a generated project folder.

**Project** - A LocalForge-managed app idea plus a generated folder on disk. The project row stores name, description, folder path, and lifecycle status.

**Project folder** - The filesystem workspace where the coding agent builds the user's app. LocalForge writes `.pi/models.json` there and the runner enforces that mutations stay inside it.

**Provider config** - The effective `{ provider, baseUrl, model }` tuple used by bootstrapper and coding sessions. Project settings override global settings.

**Server-Sent Events** - The streaming mechanism used for live bootstrapper replies and live agent/orchestrator activity.

**Settings row** - A key-value row in SQLite. Global settings use `project_id = NULL`; project overrides use the project id.

**Working directory** - The root folder where new project folders are created. By default it is `projects/` under the LocalForge repo root.
