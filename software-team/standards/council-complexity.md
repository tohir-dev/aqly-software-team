# Council complexity threshold

This standard defines when a build qualifies as "complex" for the purpose of activating the
**deliberative council** (three-perspective debate + synthesis). The retrospective council always
runs regardless of complexity; the adversarial challenge always runs **except** where a same-stage
deliberation subsumes it (see below).

## Complexity checklist

A build is **complex** if it meets **any** of the following criteria:

| Criterion | Signal |
|---|---|
| Task count | `03-tasks.md` contains ≥ 5 distinct tasks |
| Data layer | `has_data: true` in the PRD profile |
| ML/AI component | `has_ml: true` in the PRD profile |
| External API contract | `has_api: true` in the PRD profile |
| Multi-locale | `multi_locale: true` in the PRD profile |
| Service (deployed) | `is_service: true` in the PRD profile |
| High security | `high_security: true` in the PRD profile |
| Personal data | `handles_personal_data: true` in the PRD profile |
| Mobile target | `has_mobile: true` in the PRD profile |
| Product strategy | `has_strategy: true` in the PRD profile |
| Multi-implementation agent | Both `has_ui: true` and any of the above |

## How the orchestrator applies this

After `analyst` produces `01-prd.md`, the orchestrator checks this checklist against the profile
flags and the task count from the planner's `01b-plan.md` (once available). If **any criterion
is met**, set `council_mode: deliberate` for the run and apply the deliberative council to all
eligible stages.

Record the complexity decision in `STATUS.md`'s notes column after the analyst stage.

## Eligible stages for deliberative council

Even in a complex build, deliberative council only runs on **design-decision stages** — not
implementation or gate stages:

| Stage | Eligible | Rationale |
|---|---|---|
| `analyst` | Yes | PRD shapes everything; wrong scope propagates everywhere |
| `planner` | Yes | Bad sequence → cascading gate failures |
| `architect` | Yes | Stack/design wrong → full rebuild |
| `data-architect` | Yes (if has_data) | Schema mistakes are hardest to reverse |
| `risk-analyst` | Yes | Risk register gaps go undetected until gates fail |
| `product-strategist` | Yes (if has_strategy) | Roadmap/prioritization errors compound across releases |
| `engineer` | No | Implementation is owned by the engineer; debate adds noise here |
| `qa` / gate agents | No | Gates must be independent; council debate would compromise their verdict |
| `writer` / `support` | No | Too late in the pipeline; wrong docs → challenge, not debate |

## When deliberation subsumes the same-stage challenge

A stage that actually ran the full 3-lens deliberation (`council-deliberate` × 3 +
`council-synthesize`) does **not** additionally need `council-challenge` on that **same** stage —
synthesis already reconciles three adversarial perspectives, a superset of one challenger's pass.
Skip the same-stage challenge only when **both** hold:

- deliberation **actually ran** for that stage on this build (complexity-eligible is not enough —
  check `STATUS.md`), and
- `council-synthesize`'s output was independently verified (spot-checked against the raw lens
  outputs, not merely accepted) and the skip plus its reason is recorded in `STATUS.md`.

The challenge still runs on every stage that did **not** run deliberation — including a
complexity-eligible stage where deliberation was skipped or cost-limited for that build.

## Per-stage council intensity

Not every complexity-eligible stage warrants the same weight. Right-size per stage and record the
choice in `STATUS.md`:

- **Full 3-lens deliberation** on the highest-uncertainty / highest-consequence design stages —
  PRD framing (`analyst`) and guardrail/architecture shape (`architect`, and `data-architect` when
  `has_data`).
- **Challenge-only** suffices on stages whose inputs are largely pre-encoded by an upstream
  deliberation — e.g. `planner` sequencing after a deliberated PRD, or `risk-analyst` after a
  deliberated design — since the challenge alone still catches stage-local defects (including a
  consequence-critical gap) without re-litigating decisions the upstream deliberation already made.

This right-sizing only applies to stages already eligible under the checklist above; it narrows
*intensity*, not *eligibility*.

## Re-challenge cap by stage

The adversarial challenge cap (max re-challenges before escalating) varies by stage complexity:

| Stage | Default cap | Rationale |
|---|---|---|
| `analyst` | 2 | PRD ambiguity is resolved via blocker flag, not re-challenges |
| `planner` | 2 | Re-planning is handled by the failure-recovery standard |
| `architect` | 3 | Architectural challenges may need multiple passes on deep structural issues |
| `data-architect` | 3 | Schema challenges often surface layered problems |
| `risk-analyst` | 2 | Risk gaps should escalate to human checkpoint, not re-challenge loop |
| `engineer` / `frontend` | 2 | Code fixes should be fast; escalate to qa gate if not |
| `ux` / `ui` | 2 | Design challenges escalate to architect if structural |
| `writer` | 1 | Doc gaps are quick fixes; 1 re-challenge is sufficient |
| `sre-operate` | 2 | Observability gaps → devops or sre-readiness gate |
| `devops` | 2 | CI/infra issues → devops owns them; escalate to gate if stuck |
| `product-strategist` | 2 | Roadmap/strategy challenges resolve fast or escalate |
