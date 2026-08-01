# Failure recovery — gate failure diagnosis and routing

This standard gives the orchestrator a structured decision process for handling gate failures.
Read it whenever a gate returns `verdict: fail` before routing findings to owning agents.

## Step 1 — Classify the failure type

Before routing, classify each finding into one of five categories:

| Category | Signals | Example |
|---|---|---|
| **Local bug** | Single file/function, clear required_fix, no design change needed | Off-by-one in a validator |
| **Design flaw** | Multiple files affected, requires a structural change, architect's spec is wrong | Wrong data model chosen |
| **Plan/approach wrong** | Gate fails repeatedly (≥ 2 retries) on the same root cause, or findings span unrelated areas | Security scan fails because the wrong framework was chosen |
| **Gate/tooling error** | Gate reports fail but evidence is missing, contradictory, or the tool itself errored | Scanner crashes; test runner not installed |
| **Infrastructure/capacity-transient** | The failure is not a gate finding at all — a non-gate infrastructure or capacity outage (API error, session/context limit, write/classifier outage) interrupted a stage or subagent mid-run | A council-deliberate lens cut off by an API error; a write-safety classifier outage delaying a trail write |

### Infrastructure/capacity-transient → retry, don't route

This class is not a gate failure and must not be treated as one:
- **Retry-with-backoff and resume-from-checkpoint.** Re-dispatch the interrupted stage/lens (or
  resume it inline if it was near-complete) rather than routing to an owning agent or planner.
- **Does not consume a gate retry slot** and is not a "local bug"/"design flaw"/"plan wrong" —
  routing it to an owning agent misattributes an infrastructure hiccup as a defect in their work.
- **Re-verify before proceeding.** After the retry/resume, confirm the stage's artifacts are
  actually complete on disk (not just that the subagent returned) before treating the stage as
  done — a half-applied multi-file edit or a partially-written deliberation must never be treated
  as finished because the process resumed. When multiple deliberation lenses were in flight, confirm
  **all** of them produced complete output before invoking `council-synthesize`.

## Step 2 — Route by category

### Local bug → owning agent + re-run gate

Route each finding to its `owner` field. The most common mappings:

| `owner` | Route when |
|---|---|
| `engineer` / `frontend` / `mobile` | Code defect in application logic |
| `data-engineer` | Pipeline, ETL, or data-quality defect |
| `ml-engineer` | Model, eval, or inference defect |
| `devops` | Build script, CI config, or dependency CVE |
| `writer` | Missing or wrong doc, LICENSE absent |
| `sre-operate` | Observability gap, missing runbook |
| `dba` | DB tuning, backup/restore, replication gap |

After each owner fixes its findings, re-run only the failed gate (not the full pipeline).
Require the gate to mark each prior `finding-id` as `resolved` or `still-open`.

### Design flaw → architect (+ possibly data-architect) + re-run from build

Signs of a design flaw:
- Findings cluster around an interface or data model that the architect defined
- The `required_fix` says "redesign" or "change the schema"
- `qa` and `reviewer` both surface the same structural issue

Action: route to `architect` (and `data-architect` if schema-related) with the relevant findings
as context. After architect produces a revised `02-design.md` / `03-tasks.md`, re-run from build
(engineer → qa → reviewer + security). Do **not** re-run analyst/planner unless the PRD itself is wrong.

### Plan/approach wrong → planner for a revised plan

Signs the plan (not just the code) is wrong:
- Same gate fails on the **same root cause** after 2+ retries
- Multiple findings owned by different agents but all traceable to one architectural or sequencing mistake
- `BLOCKED.md` would contain the same first finding regardless of more retries

Action: set `replan: true` and delegate to `planner` with the open findings and gate history.
Planner writes a revised `01b-plan.md`, explaining what changed and why the prior plan was wrong.
Then re-run from the earliest affected stage. Cap re-plan routing at 1 re-plan per gate; a second
failure after re-planning means `BLOCKED.md` and an honest stop.

### Gate/tooling error → fix the gate, then re-run it

If the gate's `evidence` is missing, contradictory, or the tool itself failed (non-zero exit from
a crash, not a test failure):
1. Check whether the tool is installed/configured (read `08-devops.md` if present).
2. Route to `devops` if the tool isn't set up.
3. Re-run the gate once the tool is available. This does **not** consume a retry slot.

## Step 3 — Pattern recognition across retries

After each retry, before routing again, check for patterns:

```
same gate, same finding root cause across ≥ 2 attempts → "plan/approach wrong" (→ planner)
different gates failing on the same file or module     → "design flaw" (→ architect)
only one gate failing, different findings each time    → "local bugs" (continue routing to owners)
gate passes intermittently (flaky)                     → "gate/tooling error" (→ devops to fix flakiness)
```

Persist your classification in `STATUS.md`'s `note` column alongside the retry count.

## Step 4 — Hard stop criteria

Write `BLOCKED.md` and stop without shipping when any of the following is true:
- A gate has consumed its 3 retry slots and still fails
- A re-plan was issued and the same gate still fails after it
- The `risk-analyst`'s `irreversible_actions` list includes an action that the human hasn't approved

`BLOCKED.md` must include:
- The gate that blocked
- All open `finding-id` entries with their `severity`, `owner`, and `defect`
- The classification from Step 1
- What the orchestrator already tried (which agents were re-run, how many times)
- A recommended next human action (fix X manually, revise the request, or approve the risky action)

## Reference: retry budget per gate

| Gate | Max retries | Re-plan allowed? |
|---|---|---|
| `qa` | 3 | Yes (after 2 same-root-cause failures) |
| `perf` | 3 | Yes |
| `reviewer` | 3 | Yes (design flaw → architect, not re-plan) |
| `accessibility` | 3 | No (route to frontend) |
| `security` | 3 | Yes (after 2, route to devops for tool/dep fix or to architect for design) |
| `red-team` | 3 | Yes |
| `sre-readiness` | 3 | Yes (missing infrastructure → devops; missing docs → writer) |
