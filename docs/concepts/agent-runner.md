# Agent runner

The agent runner is the isolated execution process that drives Pi coding-agent for exactly one LocalForge feature.

## What it solves

The Next.js server should coordinate work, not perform long-running file edits and model/tool loops inside a request handler. The runner separates execution from orchestration: it gets a project folder, feature context, local provider config, and environment variables, then reports progress through stdout.

## Process contract

The orchestrator starts the runner like this:

```text
node scripts/agent-runner.mjs --session-id <id> --feature-id <id> --feature-title <title> --prompt-file <path> --project-dir <path> --base-url <url> --provider <provider> --model <model>
```

The runner emits one JSON object per stdout line:

```json
{"type":"log","message":"Editing src/App.tsx","messageType":"action"}
```

```json
{"type":"done","outcome":"success"}
```

The orchestrator treats non-JSON stdout as an `info` log so unexpected output is still visible in the UI.

## Prompt construction

The runner receives long feature context through a temporary prompt file. The file contains:

| Field | Source |
|-------|--------|
| `id` | Feature id |
| `title` | Feature title |
| `description` | Feature description |
| `acceptanceCriteria` | Feature acceptance criteria |
| `coderPrompt` | Effective project/global coder prompt |

The system prompt tells the agent:

1. Work only on the assigned feature.
2. Read before editing.
3. Modify real files on disk.
4. Run project checks when available.
5. Keep the app's dev server on the configured project port.
6. Avoid broad process killers such as killing all Node.js processes.

## Workspace guard

The runner enforces the project boundary with a Pi extension. File mutation tools and shell commands are checked so writes stay inside the generated project folder.

Read/search/list tools are allowed outside the same mutation path because the guard focuses on preventing destructive or accidental writes above the project workspace.

## Local model runtime

The runner registers the active model with Pi in memory. It maps LocalForge provider IDs to Pi provider names:

| LocalForge provider | Pi provider |
|--------------------|-------------|
| `lm_studio` | `lm_studio` |
| `ollama` | `ollama` |

Base URLs are normalized to include `/v1`, and the model is registered as an OpenAI completions-compatible text model with zero cost metadata.

## Tool event streaming

The runner subscribes to Pi session events and summarizes tool starts into short log lines:

| Pi tool | Example LocalForge log |
|---------|------------------------|
| `read` | `Reading package.json` |
| `write` | `Writing src/App.tsx` |
| `edit` | `Editing src/App.tsx` |
| `bash` | `Running: npm run build` |
| `grep` | `Searching for useState` |
| `find` or `ls` | `Finding files` or `Listing files` |

Assistant text is also truncated and emitted as `info` so users can see the agent's own summary.

## Retry behavior

The runner retries transient model/provider failures. Current transient patterns include invalid tool-call generation, overloads, rate limits, timeouts, connection resets, and 502/503/529-style errors.

The retry controls are environment variables:

| Variable | Default |
|----------|---------|
| `LOCALFORGE_MAX_RETRIES` | `3` |
| `LOCALFORGE_RETRY_DELAY_MS` | `5000` |
| `LOCALFORGE_MAX_TURNS` | `1000` |

## Playwright verification

Playwright verification is opt-in through settings. When disabled, a successful Pi session is enough for the runner to emit success.

When enabled, the runner:

1. Writes a generated spec under the project folder's `tests/` directory.
2. Launches Chromium through `@playwright/test`.
3. Navigates to the configured project dev server URL.
4. Captures a screenshot under LocalForge's `screenshots/` directory.
5. Emits a `test_result` log such as `npx playwright test completed: 1 passed, 0 failed (639ms)`.
6. Emits a `screenshot` log if a screenshot file exists.

Connection refused and empty page title are treated as non-fatal in specific cases because they can mean the target app is not currently serving HTML even though the coding session itself completed.

## Exit outcomes

| Runner condition | Emitted outcome | Orchestrator result |
|------------------|-----------------|---------------------|
| Pi succeeds and verification is disabled | `success` | Feature completed |
| Pi succeeds and Playwright passes or is non-fatal | `success` | Feature completed |
| Pi fails | `failure` | Feature demoted to backlog |
| Playwright fails with a fatal error | `failure` | Feature demoted to backlog |
| SIGTERM/SIGINT | `failure` plus process exit | Session usually becomes terminated or failed depending on close signal |
