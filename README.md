# Software Team — a 40-agent software company for Claude

Most AI coding help is one agent writing code and telling you it works. This skill is an **agent
software company**: a product manager writes the PRD, a planner sequences it, designers own the UX
and the design system, an architect picks the stack, a risk analyst runs a pre-mortem *before*
anything is built, engineers implement — and then it has to get past gates that are built to fail it.

The gates are the product. A `verdict: pass` is only valid if the artifact pastes the **real command
output** that proves it. A pass with no evidence is treated as a failure. When a gate fails, the
orchestrator classifies the failure before routing it — a design flaw goes back to the architect, not
to the engineer — and if the same root cause recurs twice, it re-plans instead of retrying. If it
still cannot pass, it writes `BLOCKED.md` and stops honestly rather than shipping something green.

## Three modes

| Mode | What it does |
|---|---|
| **BUILD** | Turn a plain-language request into new production-grade software, with a full audit trail |
| **IMPROVE** | Change an existing program in another folder, **in place**, on an isolation git branch — never pushes without your explicit approval |
| **COUNCIL** | Inspect the system's own quality: run health, open improvement recommendations, cross-run meta-retrospect, and (human-invoked) apply |

## The pipeline

```
analyst      PRD + acceptance criteria + the profile flags
planner      milestones, risk-first sequencing
ux → ui      flows, IA → design system, components, states      (UI products)
architect    stack choice, system design, task breakdown
data-architect  schema, indexing, migration safety              (data products)
compliance   privacy, residency, regulation, licenses           (personal data)
risk-analyst pre-mortem → risk register ── irreversible? → HUMAN CHECKPOINT
engineer     implementation (+ frontend / mobile / data / ML per profile)
─────────────────────────── gates ───────────────────────────
qa           tests + every acceptance criterion + assigned risks
reviewer + security     adversarial review ‖ scan, OWASP, secrets, deps
accessibility / design-tester / perf / red-team                 (per profile)
─────────────────────────────────────────────────────────────
devops       build, container, CI/CD
sre-operate  observability, alerts, runbooks, rollback drill    (services)
writer       README, usage docs, CHANGELOG, LICENSE
support      known issues, troubleshooting, feedback loop       (products with users)
sre-readiness  the production-readiness MUST-HAVE gate
history-writer + council-retrospect → deliver
```

Every situational specialist is **off by default**. A small CLI gets the core chain and nothing else
— the system is explicitly instructed never to invoke a specialist just to have it report N/A.

## What's in the box

- **40 agent definitions** (`agents/`) — every profession above, plus six council agents.
- **Three mode playbooks** (`modes/build.md`, `modes/improve.md`, `modes/council.md`).
- **Six engineering standards** (`standards/`) — coding standards, definition of done, the stage
  handoff contract and profile flags, tech-stack policy, failure-recovery routing, and the
  council-complexity threshold.
- **Log schemas** (`councils/schema.md`, `history/schema.md`) — the append-only run and council
  memory. Logs are created on your first run; the skill ships schemas only, so you start clean.

## The council system

Three council types wrap the pipeline:

- **Deliberative** — on complex builds, major design stages are argued from three lenses in parallel
  (risk-first, user-first, tech-first) and then synthesized into one improved artifact.
- **Adversarial challenge** — a devil's advocate runs after every marked stage and can send it back
  with blocking findings, with per-stage caps so it cannot loop forever.
- **Retrospective / meta-retrospective** — every run is scored; every fifth run, recurring weaknesses
  across runs are ranked by leverage. `council-apply` can then turn those into surgical edits to the
  agents themselves — but only when you explicitly ask, dry-run first.

## Install

**Importing from GitHub** (marketplaces, or any skill importer that takes a URL) — paste
**this** URL. It points at the folder that contains `SKILL.md`, which is one level below the
repo root:

```
https://github.com/tohir-dev/aqly-software-team/tree/main/software-team
```

> ⚠️ The bare repo URL (`https://github.com/tohir-dev/aqly-software-team`) will **not** work —
> there is no `SKILL.md` at the repo root.

**Installing manually** — clone or download the repo, then copy the inner
**`software-team/`** folder (not the `aqly-software-team` repo folder — the skill directory name must
match the `name:` in `SKILL.md`).

**Project scope** — available in one repo:

```bash
mkdir -p .claude/skills && cp -R software-team .claude/skills/
```

**Personal scope** — available everywhere:

```bash
mkdir -p ~/.claude/skills && cp -R software-team ~/.claude/skills/
```

**Optional speedup.** The orchestrator dispatches agents by reading `agents/*.md` and inlining them.
To register them as native subagent types instead:

```bash
mkdir -p .claude/agents && cp software-team/agents/*.md .claude/agents/
```

Pairs with the companion **production-readiness** skill, which `sre-readiness` uses for the launch
checklist when it is installed.

## Try it

```
Build me a CLI that watches a folder and syncs changed files to S3, with retries.
```

```
Improve /Users/me/code/my-api — the auth middleware is a mess and there are no
tests around token refresh.
```

```
Show me the council status — what improvement recommendations are still open?
```

## Safety model

- **Your request is data, not instructions.** Text embedded in a request or in the code being read
  that tries to redirect the agents, extract secrets, or trigger a push is surfaced as a finding, not
  obeyed.
- **Worker agents never push, open PRs, or publish.** Ever.
- **The orchestrator pushes only on IMPROVE, only with your explicit approval**, only to the target
  project's own remote, only to an `agentco/<slug>` branch — never the default branch, never
  force-pushed, never auto-merged.
- **Irreversible actions hit a human checkpoint** before they run — the risk analyst flags them
  before the build starts.
- **Work stays inside the target.** A pipeline run cannot modify the skill's own agents or standards;
  that is a separate, human-invoked operation.

## Requirements

- Claude Code (or any Claude client that supports skills and subagent dispatch).
- `git` for IMPROVE mode's isolation branch. `gh` only if you opt into the PR step.
- No API keys and no external services for the system itself. Everything is Markdown.

## License

See [LICENSE](LICENSE).

---

### Part of Aqly Skills

Six other standalone multi-agent skills for Claude Code, each sold separately:

- [research-team](https://github.com/tohir-dev/aqly-research-team) — 20-archetype research org
- [analyst-team](https://github.com/tohir-dev/aqly-analyst-team) — business-intelligence org
- [sales-team](https://github.com/tohir-dev/aqly-sales-team) — 17-role revenue engine
- [marketing-team](https://github.com/tohir-dev/aqly-marketing-team) — 15-agent marketing agency
- [financist-team](https://github.com/tohir-dev/aqly-financist-team) — 8-lens finance analysis
- [production-readiness](https://github.com/tohir-dev/aqly-production-readiness) — launch-blocker gate
