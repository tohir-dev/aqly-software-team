# Definition of Done

A unit of work is **done** only when ALL of these hold. Agents must not report "done" otherwise.

## Code
- [ ] Implements the acceptance criteria it claims to (mapped explicitly).
- [ ] Builds / runs with documented commands from a clean checkout.
- [ ] No dead code, no commented-out blocks, no leftover debug prints.
- [ ] Errors handled — no silent failures; meaningful messages; no unhandled rejections/panics on expected paths.
- [ ] Input validated at trust boundaries.
- [ ] No secrets, keys, or tokens in source. Config via env.

## Tests
- [ ] Unit tests for core logic; integration tests for boundaries; a smoke test for the main happy path.
- [ ] Tests actually run green — output included in the QA report (not asserted from memory).
- [ ] Meaningful coverage on critical paths (not a coverage-percentage game).
- [ ] **Flake-detection proof.** Any gate (engineer, qa, reviewer, orchestrator) that certifies a test
      suite green pastes evidence of repeated runs under load — at least 20 sequential runs AND one
      parallel/oversubscribed batch — not a single idle run. A single "exit 0" or a claim of "no
      flakiness" / "exit 0 every time" with no pasted stress-run count is **not** sufficient proof; a
      pass on that basis alone is treated as unverified, not green.

## Quality gates
- [ ] Lint/format/type-check pass (whatever the chosen stack provides).
- [ ] Reviewer verdict = pass (no open blocker/high findings).
- [ ] Security verdict = pass (scan clean or findings triaged & justified).
- [ ] A **real dependency + SAST scan actually ran** and its command output is pasted as evidence. A
      security pass qualified only on manual manifest review (no scanner available) is **not** sufficient
      for final ship: the missing-scanner gap must be resolved and the scan re-run, or ship is blocked.
      The same "no scanner installed" gap carried forward across runs escalates from a medium finding to a
      **ship-blocking** one — it may not be indefinitely re-qualified as an accepted medium.

## Cross-environment proof (dialect- & authorization-sensitive changes)
- [ ] Any change touching dialect/type-specific behavior (e.g. SQLite vs PostgreSQL) or authorization/
      RLS/tenant-isolation is **not done** until its gate proofs run on BOTH the production dialect AND
      the least-privilege production DB/runtime role — or the untested dialect/role is recorded as an
      **explicit blocking finding**, never silently assumed to hold.
- [ ] A green result obtained on a non-production dialect (e.g. SQLite standing in for Postgres) or under
      a superuser/table-owner role that bypasses the control under test is **never** sufficient proof for
      an RLS, isolation, or type control. Prove it where and how production runs it.

## Safety-critical guardrail proof
- [ ] Any probe that verifies a safety-critical behavioral guardrail (e.g. a refusal contract, an
      abuse/harm boundary, an approval gate) states its **strength tier**: (a) an **independent live
      run** — a separate session or dispatch actually exercising the guardrail under genuine
      multi-agent conditions — or (b) a **single-session self-synthesis** by the same session that
      authored the guardrail's shape.
- [ ] Tier (b) evidence caps the claim at **shape-verified**, never **behavior-verified** — the same
      session that wrote the contract producing and scoring its own probe answers proves the contract
      is well-formed, not that independent behavior honors it.
- [ ] When only tier (b) evidence exists, the readiness gate must carry the un-run genuine
      end-to-end check as a **named open item**, and recommend running it (via a separate human or
      session) before any monetization or execution wiring that depends on the guardrail holding.

## Production readiness (MUST-HAVE subset, scaled to the program)
- [ ] Runs reproducibly (pinned deps / lockfile; documented runtime version).
- [ ] Config separated from code; sane defaults; example `.env.example` if needed.
- [ ] Basic observability: structured logs on important paths; clear error surfaces.
- [ ] A rollback/uninstall or "how to revert" note if it deploys or installs anything.
- [ ] Build/run automation exists (script, Makefile, or CI workflow).

## Docs
- [ ] `README` covers: what it is, install, run, test, configure, and limitations.
- [ ] `CHANGELOG` entry for this version.
- [ ] Any non-obvious decision recorded (in build report or ADR).

> Scale the bar to the artifact: a 50-line CLI doesn't need multi-AZ redundancy. But auth, secrets,
> input validation, tests, error handling, and reproducible runs are **never** optional.
