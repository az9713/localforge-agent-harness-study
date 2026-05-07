# Runtime settings

LocalForge settings are key-value rows in SQLite. Global rows have `project_id = NULL`; project overrides use a concrete `project_id`.

## Global settings

Defaults live in `DEFAULT_GLOBAL_SETTINGS` in `lib/settings.ts`.

| Key | Default | Scope | Meaning |
|-----|---------|-------|---------|
| `provider` | `lm_studio` | Global and project | Active local model provider |
| `lm_studio_url` | `http://127.0.0.1:1234` | Global and project | LM Studio base URL |
| `ollama_url` | `http://127.0.0.1:11434` | Global and project | Ollama base URL |
| `model` | `google/gemma-4-31b` | Global and project | Model id sent to Pi/provider |
| `working_directory` | `<repo>/projects` | Global only | Parent folder for new project folders |
| `coder_prompt` | Empty string | Global and project | Extra instructions appended to the coding-agent system prompt |
| `dev_server_port` | `3000` | Global and project | Port generated project dev servers must use |
| `max_concurrent_agents` | `1` | Global and project | Number of coding agents LocalForge can run for one project |
| `playwright_enabled` | `false` | Global and project | Whether the runner performs post-session browser verification |
| `playwright_headed` | `false` | Global and project | Whether verification opens visible Chromium when not in CI |

## Project overrides

Project-specific settings can override every key except `working_directory`. The working directory is global-only because it determines where new project folders are created before a project exists.

Effective provider config is resolved by `getEffectiveProviderConfig(projectId)`:

1. Load global settings.
2. Load project overrides when a project id is provided.
3. Pick each override when present, otherwise use the global value.
4. Normalize the provider to `lm_studio` if an invalid provider somehow appears.

## Validation

| Setting | Validation |
|---------|------------|
| `provider` | Must be `lm_studio` or `ollama` |
| Provider URLs | Must be non-empty `http` or `https` URLs |
| `model` | Must be non-empty |
| `working_directory` | Must be non-empty |
| `max_concurrent_agents` | Must be `1`, `2`, or `3` |
| `playwright_enabled` | Must be `true` or `false` |
| `playwright_headed` | Must be `true` or `false` |

## Environment variables

| Variable | Default | Read by | Meaning |
|----------|---------|---------|---------|
| `LOCALFORGE_DB_PATH` | `data/localforge.db` | `lib/db/index.ts` | SQLite database path |
| `LOCALFORGE_LOG_SQL` | Off | `lib/db/index.ts` | Enables Drizzle SQL logging when `1` or `true` |
| `LOCALFORGE_LOG_DB_CONNECT` | Off | `lib/db/index.ts` | Logs database connection path when `1` or `true` |
| `LOCALFORGE_SESSION_TIMEOUT_MS` | `1800000` | `lib/agent/orchestrator.ts` | Runner watchdog timeout |
| `LOCALFORGE_MAX_TURNS` | `1000` | `scripts/agent-runner.mjs` | Pi coding-agent turn limit |
| `LOCALFORGE_MAX_RETRIES` | `3` | `scripts/agent-runner.mjs` | Transient provider retry count |
| `LOCALFORGE_RETRY_DELAY_MS` | `5000` | `scripts/agent-runner.mjs` | Base retry delay |
| `LOCALFORGE_PLAYWRIGHT_TIMEOUT_MS` | `60000` | `scripts/agent-runner.mjs` | Browser verification timeout |
| `CI` | Unset | `scripts/agent-runner.mjs` | Forces Playwright verification to stay headless |

The orchestrator also passes per-session environment variables to the runner:

| Variable | Meaning |
|----------|---------|
| `LOCALFORGE_SESSION_ID` | Current `agent_sessions.id` |
| `LOCALFORGE_FEATURE_ID` | Current `features.id` |
| `LOCALFORGE_DEV_SERVER_PORT` | Effective project dev server port |
| `LOCALFORGE_PLAYWRIGHT_BASE_URL` | `http://localhost:<dev_server_port>` |
| `LOCALFORGE_PLAYWRIGHT_ENABLED` | Effective Playwright enabled flag |
| `LOCALFORGE_PLAYWRIGHT_HEADED` | Effective Playwright headed flag |

## Package scripts

Scripts are defined in `package.json`.

| Command | Purpose |
|---------|---------|
| `npm run dev` | Start LocalForge on port 7777 |
| `npm run build` | Build the Next.js app |
| `npm start` | Start the production Next.js server |
| `npm run lint` | Run the configured Next.js lint command |
| `npm run db:generate` | Generate Drizzle migrations from schema changes |
| `npm run db:migrate` | Apply Drizzle migrations |
| `npm run db:push` | Push schema changes directly with Drizzle |
| `npm run db:studio` | Open Drizzle Studio |
| `npm test` | Run Playwright tests |
| `npm run test:ui` | Run Playwright tests in UI mode |

There is no `typecheck` package script. Use TypeScript directly:

```bash
npx tsc --noEmit
```
