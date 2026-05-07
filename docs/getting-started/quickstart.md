# Quickstart

This takes about 10 minutes after Node.js and a local model provider are installed.

## Prerequisites

| Dependency | Required version | Verify |
|------------|------------------|--------|
| Node.js | 20 or newer | `node --version` |
| npm | Bundled with Node.js | `npm --version` |
| LM Studio or Ollama | Any version that can serve an OpenAI-compatible local endpoint | Open the provider UI or CLI |

LM Studio is the default provider. LocalForge expects it at `http://127.0.0.1:1234` unless you change settings.

## 1. Install dependencies

From the repository root:

```bash
npm install
```

Expected result: `node_modules/` exists and npm exits with code 0.

## 2. Start a local model server

For LM Studio:

1. Download a chat or coding model.
2. Load the model.
3. Start the local server.
4. Confirm the server URL is `http://127.0.0.1:1234`.

For Ollama, set the provider to `ollama` in LocalForge settings and use the default URL `http://127.0.0.1:11434`.

## 3. Apply database migrations

```bash
npm run db:migrate
```

Expected result: SQLite database files appear under `data/`, and Drizzle reports that migrations completed.

## 4. Start LocalForge

```bash
npm run dev
```

Expected output includes a local URL on port 7777:

```text
http://localhost:7777
```

Open `http://localhost:7777` in a browser.

## 5. Create a project

Use the sidebar to create a project. LocalForge writes:

| Artifact | Location | Purpose |
|----------|----------|---------|
| Project row | `data/localforge.db` | Durable project metadata |
| Project folder | `projects/<project-slug>/` by default | Workspace where the generated app is built |
| Pi model file | `projects/<project-slug>/.pi/models.json` | Provider config Pi can use from that folder |

## 6. Create features

Use either path:

| Path | When to use it | What happens |
|------|----------------|--------------|
| Manual feature creation | You already know the feature list | `POST /api/projects/:id/features` creates rows directly |
| AI bootstrapper | You want help shaping the idea | Chat transcript is converted into 6-15 feature rows through Pi custom tools |

## 7. Start agents

Click the project run control. LocalForge starts one or more coding agents, depending on `max_concurrent_agents`.

What happened:

1. The orchestrator found a ready backlog feature.
2. It marked that feature `in_progress`.
3. It spawned `scripts/agent-runner.mjs` as a child process.
4. The runner drove a Pi coding-agent session inside the generated project folder.
5. Logs streamed back into the LocalForge UI through SSE.

## Next steps

Read [project lifecycle](../concepts/project-lifecycle.md) to understand each state transition. Read [agent runner](../concepts/agent-runner.md) to understand how the coding process is isolated and verified.
