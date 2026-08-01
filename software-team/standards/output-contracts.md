# Output contracts — how agents hand off work

All stage state for a build lives in `projects/<name>/.pipeline/`. Each agent reads the prior
artifacts it needs and writes exactly one artifact of its own (the engineer and frontend write
**separate** files — never the same one). Keep artifacts terse and structured — they are
machine-and-human readable handoffs, not essays.

## Pipeline-path resolution (BUILD mode vs IMPROVE mode)

The pipeline directory is **resolved from the run type**, and every stage artifact MUST be written under
the resolved path — never a stray one:
- **BUILD mode** → `projects/<name>/.pipeline/` (inside the run's project folder).
- **IMPROVE mode** → `<target>/.agentco/.pipeline/` (inside the authorized target's `.agentco/` trail).

An artifact written outside the run's resolved pipeline path (e.g. a stray root `.pipeline/` on an
IMPROVE mode run) is a wiring defect: the orchestrator MUST **reject or relocate** it into the resolved path
before the stage is accepted. The `projects/<name>/.pipeline/` paths below are the BUILD mode layout;
substitute `<target>/.agentco/.pipeline/` verbatim for an IMPROVE mode run.

## Pipeline state directory

```
projects/<name>/
  .pipeline/
    00-request.md        # orchestrator: the original request, verbatim
    STATUS.md            # orchestrator: live pipeline status (snapshot)
    RUN-LOG.md           # orchestrator: append-only run trace
    01-prd.md            # analyst (sets the profile hints)
    01b-plan.md          # planner: milestones + risk-first sequence
    01c-ux-spec.md       # ux: flows, IA, interaction (UI products only)
    01d-ui-spec.md       # ui: design system, components, states (UI products only)
    01e-localization.md  # localization: i18n architecture (multi-locale only)
    01f-design-direction.md # design-lead: art direction + design critique (UI products only)
    01g-motion-spec.md   # motion-designer: motion/animation spec (has_motion only)
    01h-content-design.md # ux-writer: product microcopy + voice & tone (UI products only)
    02d-product-strategy.md  # product-strategist: vision, roadmap, prioritization (has_strategy; early in ① Frame)
    02-design.md         # architect (includes has_ui flag)
    02c-api-design.md    # api-designer: API contract (API products only)
    03-tasks.md          # architect: ordered task list with owners
    03a-data-design.md   # data-architect: schema, indexes, migration (data products only)
    03c-compliance.md    # compliance: privacy, data-residency, regulations (personal-data only)
    03b-risk-register.md # risk-analyst: pre-mortem + reversibility  [pre-build gate]
    04-build-report.md   # engineer (backend/fullstack)
    04b-frontend-report.md # frontend (only if has_ui)
    04c-mobile-report.md   # mobile (only if has_mobile)
    04d-data-eng-report.md # data-engineer (only if has_pipelines)
    04e-ml-report.md       # ml-engineer (only if has_ml)
    05-qa-report.md      # qa            [gate]
    05b-perf.md          # perf          [gate] (perf-critical only)
    06-review.md         # reviewer      [gate]
    06b-accessibility.md # accessibility [gate] (needs_a11y only)
    06c-design-qa.md     # design-tester [gate] (UI products only)
    07-security.md       # security      [gate]
    07c-redteam.md       # red-team      [gate] (high-security only)
    08-devops.md         # devops
    08b-operate.md       # sre-operate: observability/alerts/runbooks (deployed services)
    08c-dba.md           # dba: tuning, backup/restore, replication (operational DB only)
    08d-finops.md        # finops: cost estimate + optimization (cloud infra only)
    10-docs.md           # writer
    10b-support.md       # support: known-issues, troubleshooting (products with users)
    09-readiness.md      # sre-readiness [gate]  (runs last)
    BLOCKED.md           # orchestrator: only if a gate exhausts its retries
  <actual program source...>
```

> Stage order note: `writer` (docs/LICENSE) runs before `sre-readiness` so the readiness gate can
> verify docs exist. The `09`/`10` numbers reflect gate identity, not run order.

## Orchestrator-owned files

### STATUS.md (snapshot, rewritten each stage)
A table, one row per stage: `stage | agent | status (pending|running|pass|fail|done) | verdict | attempt | last-update`.
The `attempt` column is the **source of truth** for retry counts (don't rely on in-context memory).

### RUN-LOG.md (append-only)
One line per stage start/end and per retry: `timestamp · stage · agent · model · event · verdict · attempt · note`.
This is the audit/debug trace; failures recorded here feed the eval set (`evals/`).

## Universal `consequences` block (every agent)

Every agent ends its artifact with this block — its own forward-looking prediction of what its work
causes. This is consequence-awareness baked into every stage; the `risk-analyst` independently verifies it.
```yaml
consequences:
  impact: <what this change enables or affects downstream>
  reversibility: reversible | reversible-with-effort | irreversible
  risky_assumptions: [<assumptions that, if wrong, break this>]
```

## Artifact schemas

Each artifact starts with a YAML header then a short body, and ends with the `consequences` block above.

**Reading rule (consumers MUST honor "Consumed by").** Each agent reads its listed inputs **plus any
optional/specialist artifact present for this build that its "Consumed by" line names** — e.g. an
implementer reads `02c-api-design.md` / `01e-localization.md` / `03a-data-design.md` if they exist; the
gates read **all** present `04*` implementation reports (not just `04-build-report.md`) and the relevant
specs they verify against. The orchestrator tells each agent which optional artifacts exist. A produced
spec that its implementer doesn't read is a wiring bug, not a feature.

### 01-prd.md (analyst)
```yaml
stage: prd
summary: <one line>
profile:                      # hints that route the optional/specialist agents
  has_ui: true | false        #   → design-lead, ux, ui, ux-writer, frontend, design-tester
  has_motion: true | false    #   → motion-designer
  has_mobile: true | false    #   → mobile
  has_data: true | false      #   → data-architect
  has_pipelines: true | false #   → data-engineer
  has_ml: true | false        #   → ml-engineer
  handles_personal_data: true | false  # → compliance
  is_service: true | false    #   deployed/long-running → sre-operate
  has_users: true | false     #   → support
  # situational specialists (default false; set only when the product needs them):
  has_api: true | false       #   → api-designer
  multi_locale: true | false  #   → localization
  perf_critical: true | false #   → perf
  needs_a11y: true | false    #   → accessibility
  high_security: true | false #   → red-team
  operational_db: true | false #  → dba
  cloud_infra: true | false   #   → finops
  has_strategy: true | false  #   → product-strategist (product vision/roadmap, early in ① Frame)
blocker: none | <a genuine ambiguity that changes the whole shape of the work>
```
- **Problem** — what & why, who uses it.
- **Scope / non-scope** — explicit in and out.
- **User stories** — `As a <persona>, I want <capability> so that <outcome>`.
- **Acceptance criteria** — numbered, testable, `Given/When/Then`. These become QA's checklist.
- **Constraints & assumptions** — decide and document; only set `blocker` for a genuine shape-changer.
  If `blocker` is set, the orchestrator pauses and asks the human before continuing.

### 01b-plan.md (planner)
```yaml
stage: plan
replan: false | true       # true when this is a revised plan after a repeated gate failure
```
- **Milestones** — small vertical slices, each with AC-refs and its own definition-of-done.
- **Risk-first sequence** — ordered steps; the riskiest/most-uncertain first (spike). Note dependencies.
- **Parallelism** — which steps can run concurrently (and warrant a worktree).
- **Checkpoints** — where to verify progress / where a human decision may be needed.
- On a re-plan: a short "what changed and why" (diagnose why the prior plan was wrong).
- *Consumed by:* architect, risk-analyst, orchestrator.

### 01c-ux-spec.md (ux) — UI products only
```yaml
stage: ux-spec
```
- **Users & goals**, **user flows** (incl. unhappy paths: empty/error/loading/edge), **information
  architecture / sitemap**, **interaction patterns**, **flow-level accessibility**, and **UX acceptance
  criteria** the QA gate can check. *Consumed by:* ui, architect, frontend, qa.

### 01d-ui-spec.md (ui) — UI products only
```yaml
stage: ui-spec
```
- **Design tokens** (color incl. semantic, type scale, spacing, radii, elevation), **component inventory
  with all states**, **layout/grid + responsive**, **visual accessibility** (contrast, visible focus),
  and **mockup links** if a Figma/mockup tool was used. *Consumed by:* frontend, reviewer.

### 01f-design-direction.md (design-lead) — UI products only
```yaml
stage: design-direction
```
- **Art direction** (visual language, mood, references, anti-patterns), **product design principles**,
  and — after ux/ui produce their specs — a **critique** section with structured notes (screen · issue ·
  fix). *Consumed by:* ux, ui, motion-designer, ux-writer, frontend, design-tester.

### 01g-motion-spec.md (motion-designer) — has_motion only
```yaml
stage: motion-spec
```
- **Motion tokens** (durations, easings), **per-component transitions/micro-interactions**, **loading/
  progress motion**, **choreography**, and **reduced-motion fallbacks** (WCAG 2.3.3). *Consumed by:*
  frontend, design-tester.

### 01h-content-design.md (ux-writer) — UI products only
```yaml
stage: content-design
```
- **Voice & tone** guide, a **string catalog** keyed by screen/state (labels, buttons, empty/loading/
  success/error, confirmations), the **error-message set** (human + actionable), and a **terminology
  glossary**. i18n message keys when `multi_locale`. *Consumed by:* frontend, design-tester, localization.

### 02-design.md + 03-tasks.md (architect)
```yaml
stage: design
has_ui: true | false        # drives whether the orchestrator runs the frontend agent
frontend_tasks: [ ... ]      # task ids that belong to the frontend (empty if has_ui:false)
```
- **Stack decision** — language, framework, datastore, with a one-line *why* each (per `tech-stack-policy.md`).
- **Architecture** — components, data flow, key interfaces/contracts, error strategy.
- **Tasks** (`03-tasks.md`) — ordered, each: `id · title · owner(engineer|frontend) · depends-on · AC-refs`.
- *Consumed by:* engineer, frontend, qa, reviewer, devops.

### 03a-data-design.md (data-architect) — data products only
```yaml
stage: data-design
```
- **Data model** (entities/relationships/ER), **schema sketch** (DDL: tables/keys/constraints/types),
  **normalization rationale**, **indexing** for real query patterns, **storage/engine choice** (with a
  one-line why), **data flow + retention/PII handling**, and a **migration strategy**
  (expand→backfill→switch→contract, rollback). Mark any destructive/irreversible migration explicitly.
- *Consumed by:* engineer, risk-analyst, security, sre-readiness.

### 03b-risk-register.md (risk-analyst) — pre-build gate
```yaml
stage: risk-register
checkpoint_required: true | false   # true if any irreversible action needs human approval
```
- **Ranked risks** — each block:
  ```yaml
  - id: risk-1
    scenario: <failure mode / consequence predicted by the pre-mortem>
    likelihood: low | med | high
    blast_radius: <what/who it affects>
    reversibility: reversible | reversible-with-effort | irreversible
    mitigation: <how to prevent or limit it>
    owner: <any fixing agent in the org chart (SKILL.md) — e.g. planner | architect | engineer | frontend |
            mobile | data-architect | data-engineer | ml-engineer | design-lead | ux-writer |
            product-strategist | devops — the agent that owns the fix>
    verify_at: qa | perf | reviewer | accessibility | security | red-team | sre-readiness  # gate that confirms it
  ```
- **irreversible_actions** — list of destructive/irreversible operations, each `requires_human_ok: true`.
- *Consumed by:* the orchestrator (for the human checkpoint) and every downstream gate (to verify its
  assigned `verify_at` risks were actually mitigated).

### 04-build-report.md (engineer) / 04b-frontend-report.md (frontend)
- **Implemented** — bullet list mapped to task ids.
- **Key decisions** — anything non-obvious + why.
- **Files** — paths created/changed.
- **How to run locally** — exact commands.
- **Known gaps** — honest list of what's not done.
- *Consumed by:* qa, reviewer, security, devops, writer.

### 05-qa-report.md (qa) — GATE
```yaml
stage: qa
verdict: pass | fail
evidence: |
  <the actual test command + real output: counts, pass/fail, exit code>
```
- **Tests added** — files + what they cover (unit/integration/e2e).
- **AC verification** — each acceptance criterion → met / not-met + evidence.
- **Findings** (on fail) — structured, see below.
- A `pass` without real `evidence` output is invalid and treated as `fail` by the orchestrator.

### 06-review.md (reviewer) + 07-security.md (security) — GATE
```yaml
stage: review | security
verdict: pass | fail
```
- **Findings** — structured (see below). `fail` if any blocker/high remains.
- security includes the actual scanner command + output as evidence; "no scanner available" is itself
  at least a medium finding and cannot yield an unqualified pass.

### 09-readiness.md (sre-readiness) — GATE
```yaml
stage: readiness
verdict: pass | fail
```
- The production-readiness MUST-HAVE checklist, each item checked with evidence or marked N/A with reason.
- `fail` if any MUST-HAVE launch-blocker is unmet, with structured findings naming the owning agent.

### 03c-compliance.md (compliance) — personal/regulated data only
```yaml
stage: compliance
routes_back: true | false    # true if a hard requirement (e.g. data-localization) forces a design change
```
PII inventory + privacy (consent/retention/data-subject rights), **data-residency** requirement,
applicable regulations with sources, OSS-license review, required legal docs (ToS/Privacy Policy), and
each requirement's `verify_at` gate. When `routes_back: true`, the orchestrator re-runs `architect`
(and `data-architect` if storage location changes), bumps a `compliance-rework` counter (cap 2), then
re-runs `compliance` to confirm. *Consumed by:* architect (on routes_back), risk-analyst, security,
writer (drafts the ToS/Privacy Policy text), sre-readiness.

### 04c/04d/04e — specialist implementation reports
`04c-mobile-report.md` (mobile), `04d-data-eng-report.md` (data-engineer), `04e-ml-report.md`
(ml-engineer) follow the `04-build-report.md` shape, each adding its specialty's evidence — mobile:
build + store-release notes; data-engineer: pipeline run + data-quality results; ml-engineer: **eval
method + results + failure cases**. *Consumed by:* qa, reviewer, security, devops.

### 08b-operate.md (sre-operate) — deployed services
`stage: operate`. Observability (signals + a dashboard), alerting (SLO/symptom + severities), runbooks
for the top failures, incident-response process, and **rollback/restore drill evidence**. *Consumed by:*
sre-readiness (verifies it exists), writer.

### 10b-support.md (support) — products with users
`stage: support`. Known issues/limitations (pulled from the trail), troubleshooting guide, FAQ,
issue-reporting + triage/escalation, and the feedback→backlog loop. Plus a `SUPPORT.md`/`TROUBLESHOOTING.md`
in the product. *Consumed by:* the human.

### Situational specialist artifacts (each runs only when its profile flag is set)
- **`02c-api-design.md`** (api-designer) — `stage: api-design`. The API contract: resources/endpoints,
  request/response schemas, status codes + one error format, pagination, versioning (breaking changes
  flagged), auth, DX. *Consumed by:* engineer, reviewer, writer.
- **`01e-localization.md`** (localization) — `stage: localization`. i18n architecture: string
  externalization + catalog format, locale/RTL handling, locale-aware formatting, pluralization, and the
  translation workflow. *Consumed by:* frontend, engineer.
- **`05b-perf.md`** (perf) — `stage: perf`, `verdict: pass|fail`. Performance budgets vs measured numbers,
  load-test output (evidence), bottlenecks → fixes. A **gate**.
- **`06b-accessibility.md`** (accessibility) — `stage: accessibility`, `verdict: pass|fail`. WCAG audit +
  checker output, findings with WCAG refs → frontend. A **gate**.
- **`06c-design-qa.md`** (design-tester) — `stage: design-qa`, `verdict: pass|fail`. Build-vs-design
  verification: visual fidelity, responsive, motion, content accuracy; evidence + findings → frontend. A **gate**.
- **`07c-redteam.md`** (red-team) — `stage: redteam`, `verdict: pass|fail`. Abuse cases attempted against
  this build, each with repro + impact + fix. A **gate** (exploitable blocker/high fails).
- **`08c-dba.md`** (dba) — `stage: dba`. Query/index tuning, backup + **tested-restore evidence**,
  replication/HA, DB monitoring.
- **`08d-finops.md`** (finops) — `stage: finops`. Cost estimate (with cited pricing + date), drivers,
  optimizations (with savings), budget alerts, unit economics. Advisory.

## Strategy artifact (① Frame, has_strategy)
- **`02d-product-strategy.md`** (product-strategist) — `stage: product-strategy`. Product vision + north-star, a prioritization framework applied to scope, a now/next/later roadmap, MVP-vs-defer cuts, product-led differentiation, per-bet success metrics. *Consumed by:* analyst, planner, architect, the human.

## Structured finding schema (every gate `fail`)
Each defect a gate reports is one structured block so the orchestrator can route and re-verify it:
```yaml
- id: <gate>-<n>            # e.g. qa-1, sec-2
  severity: blocker | high | medium | low
  owner: <the fixing agent — any agent in the org chart (SKILL.md): a code finding → engineer / frontend /
          mobile; a UI-design finding → design-lead / ux-writer / frontend; a schema finding →
          data-architect; a docs finding → writer>
  location: <file:line or area>
  defect: <what is wrong>
  required_fix: <the concrete change>
  ac_ref: <acceptance-criterion id, if applicable>
```

## Council artifacts

### council-challenge-<stage>.md (council-challenge)
```yaml
stage: <stage name>
verdict: pass | challenges-found
```
On `challenges-found`, a list of structured challenges:
```yaml
challenges:
  - id: chg-<stage>-<n>
    severity: blocker | major | minor
    angle: correctness | completeness | honesty | forward-risk | security
    location: <artifact section or file:line>
    finding: <what is wrong or missing>
    required_action: <concrete fix>
    ac_ref: <AC id if applicable>
```
Only `blocker` and `major` block the pipeline. Written to `.pipeline/council-challenge-<stage>.md`.
*Consumed by:* orchestrator (routing), stage agent (addressing challenges).

### 11-retrospect.md (council-retrospect)
Human-readable summary of the retrospective council's scoring and recommendations.
The structured record goes to `councils/retrospects.jsonl`.
*Consumed by:* the human (post-run review) and future `council-retrospect` runs (for trend analysis).

## Verdict rule

Any artifact with `verdict: fail` blocks the pipeline. The orchestrator reads `verdict`, routes each
structured finding to its named `owner`, increments the gate's `attempt` in STATUS, and re-runs from
the owner through the failed gate. On re-run the gate must mark each prior finding `id` resolved or
still-open. A `pass` lacking the required evidence output is treated as `fail`.

**Risk verification.** Each gate also confirms the `03b-risk-register.md` risks assigned to it
(`verify_at: <this gate>`) were actually mitigated — an unmitigated predicted risk is a finding.
**Re-plan.** If a gate exhausts its retries because the *plan or approach* was wrong (not a local code
bug), the orchestrator routes back to `planner` for a revised plan (`replan: true`), not just the engineer.
