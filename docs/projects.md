# High-Impact Projects

> **Status.** A curated list of six projects this codebase + docs make
> uniquely well-positioned to undertake. Each is sized for one engineer,
> produces a publishable artefact (not just code), and leverages both
> the working harness specimen *and* the
> [blueprint](architecture/agent-harness-blueprint.md) +
> [analysis](architecture/agent-harness-analysis.md) pair.
>
> **Audience.** Anyone deciding what to build on top of this repo —
> contributors, researchers, hobbyists with hardware, or the maintainer
> revisiting priorities.

---

## How this list was built

The six projects below were selected against three criteria:

1. **Solo-feasible.** Each is sized for one engineer in 2-8 weeks.
2. **Publishable.** Each produces an external artefact (data, paper,
   reusable tool, leaderboard) — not just internal code.
3. **Leverages this repo specifically.** Each benefits from having
   *both* a working harness specimen (LocalForge) *and* a vendor-
   neutral blueprint (the `docs/architecture/` pair). A project that
   could be done equally well from any other starting point isn't on
   this list.

What's deliberately excluded: cosmetic / small-quality-of-life
improvements to LocalForge (those live in the analysis doc's tier
recommendations); pure research projects with no code component;
projects that need infrastructure beyond a single workstation.

---

## Project 1 — Harness Conformance Test Suite

> **Highest impact.** *"What if SWE-bench, but for the harness instead of the model?"*

### What you build

A vendor-neutral test harness that grades any agent-harness
implementation against the
[blueprint's](architecture/agent-harness-blueprint.md) MUST/SHOULD
requirements. Black-box: feed the target harness a small project
brief, observe its event stream, score it on the 16 components in
[§IV of the blueprint](architecture/agent-harness-blueprint.md#iv-component--maturity-matrix).

LocalForge is the reference subject. Stretch targets: Claude Agent
SDK, OpenAI Agents SDK, LangGraph Deep Agents.

### What you publish

- A `harness-conformance/` repo with ~30 scenarios spanning T0-T3
  capability levels.
- A runner CLI that drives any harness via a thin adapter.
- A public leaderboard scoring 3-5 known harnesses.
- A write-up: *"What we found grading 5 production agent harnesses
  against the same spec."*

### Why high impact

There are eval suites for *models* (SWE-bench, HumanEval, LiveCodeBench)
but no public eval suite for the *harnesses* that drive them — and the
blueprint already gives you the rubric. First mover gets cited every
time someone says "is harness X any good?"

### Tiers advanced

Blueprint §VII T3 (verification) across the field. Analysis Tier 11a
(Generator-Evaluator) and Tier 13 (eval-driven development).

### Estimate

4-6 weeks. Bulk of the work is scenario authoring and per-harness
adapters; the rubric exists already.

### Risks

- Some harnesses need cloud API keys for the model layer.
- Adapter maintenance burden grows with each new SDK; pin versions and
  refresh quarterly rather than continuously.

---

## Project 2 — Local-Model Long-Running Coding Benchmark ("Chase Bench")

> **Highest novelty.** *Zero* public data exists on this question.

### What you build

A reproducible benchmark that measures how far a *local* model can
drive an autonomous coding run. LocalForge is the harness; the
variables are the model (Qwen Coder, DeepSeek Coder, Llama 4, Mixtral,
Gemma 4, Phi, …) and a fixed scenario set:

1. Build a CRUD-with-auth web app from a one-paragraph brief.
2. Fix a known bug in a real OSS repo (curated from issue trackers).
3. Port a small library across languages (Python → Rust, e.g.).
4. Implement a non-trivial algorithm with passing tests.

### What you publish

A leaderboard with the same metrics OpenAI / Anthropic published in
their long-running posts:

- Hours uninterrupted before the watchdog or task fires.
- LOC produced.
- % features completed.
- % tests passing on completion.
- Cost-equivalent: GPU-hours × electricity-rate.

Updated quarterly. Each result reproducible from a published config.

### Why high impact

OpenAI's [25h / 30k LOC Codex run](https://developers.openai.com/blog/run-long-horizon-tasks-with-codex)
and Anthropic's
[16-parallel C-compiler](https://www.anthropic.com/engineering/building-c-compiler)
are the only public benchmarks for long-running coding agents — and
both used frontier hosted models. There is **zero** public data on
what local models can do at this task. Filling that gap is publishable
on its own; it's also the data
[Chase explicitly says the field needs](https://www.youtube.com/watch?v=c-fsL0gsmo0)
("open models good enough to drive a harness — but how good?").

### Tiers advanced

Validates the entire blueprint as a measurement instrument, not just
as a design reference.

### Estimate

6-8 weeks. The slow part is *running* the benchmark, not building it.
Each scenario × model combination may take hours to days of compute.

### Risks

- Needs ~RTX 4090 / Mac M3 Max class hardware to be credible against
  current frontier numbers.
- Model availability shifts faster than the benchmark ships; pin a
  date-stamped model set.
- Scenario authorship is the make-or-break — bad scenarios produce
  meaningless rankings. Reuse existing scenarios from SWE-bench-style
  work where possible.

---

## Project 3 — Tier 1-3 Reference Implementation on LocalForge

> **Highest concrete value to LocalForge itself.** Turns the analysis from "audit" into "applied audit".

### What you build

Implement the [analysis doc's](architecture/agent-harness-analysis.md)
Tier 1, 2, 3, and 13 in one clean PR series on LocalForge:

- **Tier 1 — durable Pi sessions.** Replace `SessionManager.inMemory()`
  with on-disk persistence under `data/sessions/<id>/`. Persist the
  tool-call event log per the blueprint's
  [§V.9 spec](architecture/agent-harness-blueprint.md#v9-tool-call-event-log-artefact).
  Resume on restart.
- **Tier 2 — writable agent memory.** Implement the four-file durable-
  memory pattern (`Prompt.md` / `Plan.md` / `Implement.md` /
  `Documentation.md`) plus `CHANGELOG.md` with a "failed approaches"
  section, per [§V.1-V.5](architecture/agent-harness-blueprint.md#v-concrete-artefact-specifications).
- **Tier 3 — real verification.** Run the project's own test / lint /
  type-check after each session. Add a Generator-Evaluator pair per
  [§VI.5 contract](architecture/agent-harness-blueprint.md#vi5-evaluator--generator).
  Add the "mergeable between sessions" hard invariant.
- **Tier 13 — eval suite.** A `tests/evals/` directory with 5-10
  scenarios, run on CI as a regression gate.

### What you publish

A v2 LocalForge with measurable improvement on a fixed scenario set.
Before/after numbers. A blog post: *"We took a Tier-0 harness to
Tier-3. Here's what changed and what it cost."*

### Why high impact

The blueprint's value compounds when readers can see the same project
at two maturity levels. It's also the obvious next step for anyone
using LocalForge in anger — and it makes LocalForge a usable Tier-3
reference, not just a demo.

### Tiers advanced

Analysis Tiers 1, 2, 3, 13 — all green.

### Estimate

3-5 weeks. Tier 1 (durable sessions) is the hardest; everything else
builds on it.

### Risks

- Pi's `SessionManager` interface is what it is. If clean disk
  persistence is hard, you may need to fork or PR upstream. Confirm
  feasibility in week 1 before committing to the rest.

---

## Project 4 — Generator-Evaluator Measured Experiment

> **Highest publish-ability per week.** Anthropic published the recipe; nobody published the measurement.

### What you build

Implement the
[§VI.5 evaluator-as-peer-agent contract](architecture/agent-harness-blueprint.md#vi5-evaluator--generator)
on LocalForge. Run a fixed 50-scenario set with three configurations:

- **A.** Agent self-evaluation only (LocalForge today).
- **B.** Deterministic verification only (test/lint/type pass = done).
- **C.** Both + Generator-Evaluator peer agent.

Measure: false-pass rate, true-pass rate, time-to-completion, regret
rate (features marked passing that fail on inspection), token cost
per feature.

### What you publish

A short paper / blog post with a data table answering: *"Does the
GAN-inspired Generator-Evaluator pattern actually work for coding
agents — and at what cost?"*

[Anthropic's harness-design post](https://www.anthropic.com/engineering/harness-design-long-running-apps)
prescribes the pattern. No one has published controlled-experiment data
on whether it earns its cost.

### Why high impact

The harness-engineering field is full of recipes and short on data.
A clean controlled before/after on one specific pattern, on one
controlled testbed, with reproducible methodology, is the kind of
result the field is hungry for. Cite-friendly.

### Tiers advanced

Validates blueprint MAY component Y4. Provides the public data needed
to support promoting Generator-Evaluator from MAY to SHOULD.

### Estimate

2-3 weeks total: ~1 week implementation, ~1 week running scenarios,
~½ week analysis + write-up.

### Risks

- Local models may not be strong enough evaluators. May need to mix:
  generator on a small local model, evaluator on a stronger
  cloud model — which itself becomes a finding worth reporting.
- The 50-scenario set design matters more than the implementation.
  Borrow from existing benchmarks where possible.

---

## Project 5 — MCP Integration + Per-Tool Permissions

> **Highest leverage.** Adds ~250 community-built tools to local harnesses with one config primitive.

### What you build

- **MCP host adapter.** Add a `mcpServers` setting to LocalForge plus
  a harness-side adapter that registers MCP tools into Pi's tool
  registry. Both stdio and HTTP MCP transports.
- **`permissions.json`.** Per-project policy file with allow / deny /
  ask rules per tool name + argument-shape pattern. Default-allow
  today; default-ask for `bash` becomes a one-line config change.
- **Approval-gate UI.** Reuse the existing SSE channel to surface
  "agent wants to run X — approve?" prompts when policy says ask.

### What you publish

A working LocalForge that can plug in `mcp-server-filesystem`,
`mcp-server-postgres`, `mcp-server-puppeteer`, `mcp-server-github`,
etc. without harness code changes. Plus the policy file format and a
write-up showing 3-4 real third-party tools wired in.

### Why high impact

MCP is the closest thing the field has to a tool-interop standard.
Adding MCP to LocalForge instantly multiplies the toolset available to
local-model harnesses by ~250 community servers. Per-tool permissions
on top makes it safe to actually use them. Validates the blueprint's
M2 (tool layer at T5) and M4 (permissions at T3-T6) requirements with
real third-party tools.

### Tiers advanced

Analysis Tier 5 (broaden tool surface). Blueprint M4 to T3 / T6,
M2 to T5.

### Estimate

2-3 weeks. Pi already supports custom tool registration via
`defineTool`; the work is the MCP adapter and the policy evaluator.

### Risks

- Some MCP servers assume a Node/TS host; minor compatibility work.
- Permission policy granularity is hard — too coarse and the policy
  isn't useful; too fine and authoring it is a chore. Start coarse
  and refine from observed traffic.

---

## Project 6 — `SKILL.md` Starter Library for Local Harnesses

> **Highest educational and community-building value.** No public skill library exists for local-LLM workflows.

### What you build

A curated repo of 30-50 `SKILL.md` bundles for the local-LLM coding
workflow, each in the manifest format specified at
[blueprint §V.6](architecture/agent-harness-blueprint.md#v6-capability-manifest-eg-skillmd):

- name, version, description (with explicit *use when* and
  *don't use when* sections)
- required_tools
- procedure
- example invocation
- supporting scripts / templates as separate files

Coverage areas: Next.js patterns, Drizzle migrations, Playwright
debugging, dev-server reset, dependency upgrades, common Tailwind
fixes, common TypeScript-error rescues, git workflow patterns,
testing idioms.

Wire them into LocalForge by re-enabling Pi's `noSkills: false`.

### What you publish

- An `awesome-local-harness-skills` repo, curated and versioned.
- A small ablation study: same scenarios with vs without the skill
  library. Borrow Glean's
  ["73% → 85% accuracy" methodology](https://developers.openai.com/blog/skills-shell-tips)
  as the reporting template.
- A write-up: *"What 50 skills look like for a local-LLM coding
  harness, and what the model can do once it has them."*

### Why high impact

OpenAI and Anthropic have shipped `SKILL.md`-style formats but their
published skills are tied to hosted runtimes. There is no public skill
library for local-LLM workflows. A good starter set lowers the bar to
entry for everyone running local harnesses. The repo also becomes a
natural contribution surface — outside contributors can add a skill
without understanding the whole harness.

### Tiers advanced

Blueprint S3 (skills library) to T3 / T6. Analysis Tier 2 #6.

### Estimate

4-6 weeks of curation + measurement. Trivially parallelisable across
contributors once the manifest format is fixed.

### Risks

- Curation discipline. The trap is ~200 marginally-useful skills
  instead of 30 sharp ones. Per [§V.6 anti-pattern](architecture/agent-harness-blueprint.md#v6-capability-manifest-eg-skillmd):
  any skill description that reads like marketing copy gets cut.
- Skills written for one model family may not trigger correctly on
  others. Test triggering across at least two local models before
  publishing.

---

## How to choose

| If you optimise for… | Pick |
|---|---|
| Single biggest external contribution to the field | **#1 Conformance Suite** or **#2 Chase Bench** |
| Maximum concrete impact on LocalForge itself | **#3 Tier 1-3 Reference** |
| Cleanest publish-by-month-end paper | **#4 Generator-Evaluator Experiment** |
| Most leverage per week of work | **#5 MCP + Permissions** |
| Building a contributor community | **#6 `SKILL.md` Starter Library** |

## Suggested sequencing if you want to do all six

```
#3 (3-5 wk)  →  #5 (2-3 wk)  →  #4 (2-3 wk)  →  #6 (4-6 wk)  →  #1 (4-6 wk)  →  #2 (6-8 wk)
LocalForge     LocalForge       publishable     community         field-level    field-level
goes T0→T3     gets MCP +       experiment      contribution       contribution    contribution
                permissions     on Tier-3        builds on T3       (LocalForge    (LocalForge
                                LocalForge       LocalForge          as one of N    as testbed)
                                                                     subjects)
```

This order:

1. Builds capability into LocalForge first (#3, #5) — every later
   project benefits from a Tier-3 specimen with MCP support.
2. Produces a publishable result on top of that capability (#4) —
   small experiment, fast turn-around.
3. Turns the project into a contributor magnet (#6) — once others
   are showing up, you can sustain larger work.
4. Ends with two field-level contributions (#1, #2) that lean on
   everything before. Both want a strong Tier-3 LocalForge as
   either the reference subject or the testbed.

Total: roughly 21-31 person-weeks. Splits cleanly across two
contributors as `(#3+#4+#6)` and `(#5+#1+#2)`.

---

## A note on what *not* to do

The temptation when sitting on a comprehensive blueprint is to try to
implement every tier on every project. Resist this. The projects above
are the high-leverage subset — they each move *one specific thing*
forward and then stop. Smaller LocalForge-internal upgrades belong in
the [analysis doc's tier list](architecture/agent-harness-analysis.md#13-recommendations--making-localforge-a-real-long-running-self-evolving-harness),
not as standalone projects.

Each project on this list answers a question someone outside this repo
is currently asking. That's what makes them high-impact.
