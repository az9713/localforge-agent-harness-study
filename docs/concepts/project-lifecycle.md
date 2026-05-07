# Project lifecycle

A LocalForge project is a database row plus a real filesystem folder that coding agents are allowed to modify.

## What it solves

The project lifecycle keeps app generation durable. If the Next.js server restarts, the database still knows which projects, features, sessions, and logs exist. If a coding session dies, the orchestrator can recover orphaned `in_progress` features back to `backlog`.

## Creation

Project creation is handled by `createProject()` in `lib/projects.ts`.

| Step | Source of truth | Behavior |
|------|-----------------|----------|
| Validate name | `MAX_PROJECT_NAME_LENGTH` and `ProjectValidationError` | Rejects blank or too-long names |
| Choose folder | `getWorkingDirectory()` plus slugified name | Uses `projects/` by default and adds numeric suffixes for collisions |
| Write model config | `writePiSettingsFile()` | Creates `.pi/models.json` using effective provider config |
| Insert database row | `projects` table | Stores name, description, folder path, status, timestamps |
| Roll back on failure | `fs.rmSync()` cleanup | Removes the new folder if the DB insert fails |

## Feature backlog

Features are the units agents build. `lib/features.ts` validates titles, descriptions, categories, statuses, priorities, and dependency edges.

| Field | Meaning |
|-------|---------|
| `status` | `backlog`, `in_progress`, or `completed` |
| `priority` | Lower number means earlier in the backlog |
| `category` | `functional` or `style` |
| `acceptanceCriteria` | Optional details passed into the runner prompt |

Dependencies are stored separately in `feature_dependencies`. LocalForge rejects self-dependencies, cross-project dependencies, duplicates, and cycles.

## Readiness

`findNextReadyFeatureForProject()` chooses the next feature by:

1. Loading backlog features for the project.
2. Sorting by `priority`, then `id`.
3. Building a set of completed feature ids.
4. Returning the first backlog feature whose prerequisites are all completed.

This is why dependency order matters: a high-priority feature can still be skipped if it depends on unfinished work.

## Running

When the orchestrator starts a feature, it changes the feature from `backlog` to `in_progress` before creating the coding session. That order keeps the UI consistent if a reload happens between writes.

Active coding work is represented twice:

| Runtime state | Durable state |
|---------------|---------------|
| In-process `running` map in `lib/agent/orchestrator.ts` | `agent_sessions` row with status `in_progress` |

The in-process map holds child-process handles. The database holds history and user-visible state.

## Completion and retry

When a runner succeeds, LocalForge marks the feature `completed`. When a runner fails, LocalForge demotes the feature back to `backlog` and pushes its priority after the current project maximum so other ready work can run first.

Manual termination is different from failure. A terminated feature returns to `backlog` without priority demotion because the user explicitly stopped the run.

## Project completion

`markProjectCompletedIfAllDone()` flips a project to `completed` only when:

1. The project exists.
2. It is not already completed.
3. It has at least one feature.
4. Every feature status is `completed`.

On transition, the orchestrator broadcasts a `project_completed` event. The celebration UI listens for that project-level signal instead of polling for completion.
