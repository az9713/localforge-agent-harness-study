# Common issues

## LocalForge starts but the bootstrapper cannot reply

**Cause:** The configured local provider is not reachable, the wrong provider is selected, or the model name does not match a loaded model.

**Fix:**

1. Open Settings in LocalForge and confirm `provider`, provider URL, and `model`.
2. For LM Studio, confirm the local server is running at `http://127.0.0.1:1234`.
3. For Ollama, confirm the server is running at `http://127.0.0.1:11434`.
4. Restart the LocalForge dev server after changing local provider software.

```bash
npm run dev
```

**If that does not work:** Use the provider health routes from a browser or HTTP client: `/api/lm-studio/health` for LM Studio and `/api/providers/health` for the configured provider layer.

## Feature generation finishes without creating features

**Cause:** The Pi feature-generation session completed without calling `create_feature`, often because the conversation did not describe enough app behavior or the local model failed to follow the tool-only instruction.

**Fix:**

1. Continue the bootstrapper conversation with concrete users, screens, data, and behaviors.
2. Click "Generate feature list" again.
3. If it repeats, use a stronger local model or add features manually from the kanban.

**If that does not work:** Check the 502 response details in the browser network panel. The route returns tool call count and assistant summary when no rows were created.

## Clicking start says no ready features exist

**Cause:** The backlog is empty or every backlog feature is blocked by incomplete dependencies.

**Fix:**

1. Open the kanban and confirm there are cards in Backlog.
2. Open blocked feature details and inspect dependencies.
3. Complete prerequisite features or remove incorrect dependency links.

**If that does not work:** Check `feature_dependencies` in SQLite for a dependency shape that the UI does not make obvious.

## A feature is stuck in progress after a restart

**Cause:** The Next.js server restarted while a runner child process was active. SQLite still shows an `in_progress` session or feature, but the in-memory process handle is gone.

**Fix:**

1. Start the LocalForge server.

```bash
npm run dev
```

2. Click start on the project again. `startOrchestrator()` reaps orphaned active sessions and resets orphaned `in_progress` features to `backlog` before starting new work.

**If that does not work:** Stop any stale project dev server from the project header and retry.

## The generated app dev server will not start

**Cause:** The generated project folder has no `package.json`, dependencies were not installed, or the app's dev script does not accept the configured port.

**Fix:**

1. Let the coding agent create the project foundation feature first.
2. Confirm the generated folder has `package.json`.
3. Confirm the app can run on the effective `dev_server_port`.

```bash
npm run dev -- --port 3000
```

**If that does not work:** Open the project settings and verify `dev_server_port`. LocalForge's dev-server manager starts generated apps with `npm run dev -- --port <port>`.

## Playwright verification fails after the agent says it succeeded

**Cause:** Playwright verification is enabled and the generated app did not serve a usable page at `http://localhost:<dev_server_port>`, or Chromium found a fatal browser error.

**Fix:**

1. Start the generated app dev server from the project header.
2. Open the generated app URL from the header.
3. If the app is API-only or not ready for browser smoke tests, disable Playwright verification in project settings.
4. If browser verification is required, add a backlog feature that creates a minimal page with a non-empty HTML title.

**If that does not work:** Check the latest `test_result` and `screenshot` logs in the feature detail modal.

## `npm run db:migrate` fails

**Cause:** SQLite cannot create or write the database path, or a migration file conflicts with the current schema state.

**Fix:**

1. Confirm the repo root is writable.
2. Confirm `data/` can be created.
3. Use a fresh local database when you do not need existing data.

```bash
Remove-Item -LiteralPath .\data\localforge.db -Force
npm run db:migrate
```

**Warning:** Removing `data/localforge.db` deletes LocalForge project metadata, features, sessions, logs, chat messages, and settings stored in that database. It does not delete generated project folders.
