# Tech-stack policy — how the architect chooses

There is **no mandated default language**. The architect selects the stack per project and records
the choice with a one-line rationale each. Optimize for: fit to the problem, low operational burden,
maturity, and the ability to ship production-grade quickly.

## Selection heuristics

| Problem shape | Reasonable default |
|---|---|
| CLI / scripting / automation / data | Python (or Go for a single static binary) |
| Web API / backend service | TypeScript+Node, Python (FastAPI), or Go |
| Web frontend / fullstack | TypeScript + a mainstream framework |
| Performance-critical / systems | Go or Rust |
| Library for an existing ecosystem | match the ecosystem's language |

## Rules
- **Boring by default.** Pick mature, well-documented, widely-used tools. Novelty must earn its place.
- **Least operational burden.** Prefer managed/serverless and a single deployable until scale forces otherwise.
  Do **not** reach for Kubernetes/microservices for a small program.
- **One datastore until you need more.** Default to SQLite (local/simple) or Postgres (real persistence).
- **Lockfiles always.** Reproducible installs are non-negotiable.
- **Match the host.** If the program plugs into an existing project, follow that project's stack, not this policy.
- **Prefer verifiable stacks.** Favor ecosystems the `qa` and `security` agents can actually test and
  scan on this machine (a dependency scanner exists: e.g. `pip-audit`, `npm audit`, `osv-scanner`,
  `trivy`). A stack whose code can't be scanned silently weakens the security/QA gates.

## Record the decision
In `02-design.md`, list: language, framework, datastore, key libraries — each with a one-line *why*.
If a choice is reversible and cheap, don't over-deliberate; if it's load-bearing (datastore, auth),
state the trade-off you accepted.
