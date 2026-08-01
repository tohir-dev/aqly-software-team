# Councils schema

The `councils/` directory holds two append-only logs produced by the council system.

## JSON-safety rule (applies to every log below)

Every log here is JSONL: one JSON object per line, append-only, and read by downstream tooling
(`jq`, COUNCIL mode ` status`, `council-meta-retrospect`, `council-apply`). **One malformed line
breaks the whole file for every reader.** Log-writers (`council-retrospect`,
`council-meta-retrospect`, `council-apply`, `history-writer`, and the orchestrator writing
`challenge-log.jsonl`) MUST:

1. Build the record as a **data structure**, not a string — put prose containing `"`, `\`, or
   newlines into normal string values without pre-escaping.
2. Serialize with a **JSON library** (`python3 -c 'import json,sys; print(json.dumps(json.load(sys.stdin), ensure_ascii=False))'`
   or `jq -c .`). Never write JSON by concatenation, `printf`, `echo`, or heredoc.
3. **Self-validate** the emitted line before appending — round-trip it through the same JSON
   parser; do not append if it fails.
4. Append exactly **one line + one trailing `\n`**; no interior newlines, no BOM, no pretty-print.
5. **Re-parse the whole file** after the append (`python3 -c "import json;[json.loads(l) for l in
   open('<path>') if l.strip()]"`) — this catches append-buffering bugs that a single-line check
   misses.

If any step fails, do not append a broken line; write the intended record to a sidecar
`<name>-unwritten.json` and surface the failure. See each agent's own "How to emit the JSON line
safely" section for the full protocol.

## councils/retrospects.jsonl

Written by `council-retrospect` after **every** pipeline run. One JSON line per run.
See `council-retrospect.md` for the full field spec.

**Key fields:**

| Field | Purpose |
|---|---|
| `run_id` | Links to `history/runs.jsonl` for cross-referencing |
| `stage_scores` | Per-stage quality scores (0–2) on proof, completeness, honesty |
| `failure_patterns` | Systemic issues detected (gate-retry-loop, spec-build-drift, etc.) |
| `recommendations` | Concrete improvement suggestions targeting agents, standards, or pipeline |
| `council_effectiveness` | Whether deliberate/challenge councils helped or added noise |

**Querying:**

```bash
# All high-priority recommendations across all runs
jq 'select(.recommendations[].priority == "high") | .recommendations[]' councils/retrospects.jsonl

# Runs where spec-build-drift was detected
jq 'select(.failure_patterns[].pattern == "spec-build-drift")' councils/retrospects.jsonl

# Average architect completeness score across all runs
jq '[.stage_scores.architect.completeness] | add' councils/retrospects.jsonl | \
  awk '{s+=$1; n++} END {print s/n}'

# Council effectiveness summary
jq '{run_id, deliberate: .council_effectiveness.deliberate_helped, challenge: .council_effectiveness.challenge_helped}' councils/retrospects.jsonl
```

## councils/challenge-log.jsonl

Written by the orchestrator after each `council-challenge` invocation during a build.
Captures the challenge verdict so patterns can be tracked across runs.

```json
{
  "run_id": "<from history>",
  "stage": "engineer",
  "verdict": "pass | challenges-found",
  "blocker_count": 0,
  "major_count": 1,
  "minor_count": 2,
  "re_challenge_count": 0,
  "artifact": ".pipeline/council-challenge-engineer.md"
}
```

**Querying:**

```bash
# Stages that most often have blockers
jq 'select(.blocker_count > 0) | .stage' councils/challenge-log.jsonl | sort | uniq -c | sort -rn

# Re-challenge rate per stage
jq '{stage, re_challenges: .re_challenge_count}' councils/challenge-log.jsonl
```

## councils/meta-retrospects.jsonl

Written by `council-meta-retrospect` after every 5th build. One JSON line per invocation.

```json
{
  "run_at": "2026-06-29T15:00:00Z",
  "runs_analyzed": 5,
  "top_pattern": "spec-build-drift",
  "recommendation_count": 7,
  "high_leverage_count": 3,
  "report_path": "councils/meta-retrospect-20260629.md"
}
```

Full human-readable report lives at the `report_path`. Use it to drive agent/standard improvements.

```bash
# Latest meta-retrospect report path
jq -r '.report_path' councils/meta-retrospects.jsonl | tail -1

# All high-leverage patterns across all meta-retrospects
cat councils/meta-retrospect-*.md | grep -A3 "leverage: high"
```

## councils/applied.jsonl

Written by `council-apply` each time it applies a recommendation from `retrospects.jsonl` or a
meta-retrospect report. One JSON line per applied recommendation. Never deleted or rewritten.

```json
{
  "applied_at": "2026-06-29T15:30:00Z",
  "rec_id": "rec-1",
  "source_run_id": "build-abc-123",
  "source_type": "retrospect | meta-retrospect",
  "target": "agent:architect | standard:output-contracts | pipeline:build",
  "finding": "<what the retrospective found>",
  "change_summary": "<one sentence: what was changed and where>",
  "files_modified": ["<relative path>"]
}
```

**Querying:**

```bash
# All applied recommendations
jq '{rec_id, target, change_summary}' councils/applied.jsonl

# Which agents have been improved most often
jq 'select(.target | startswith("agent:")) | .target' councils/applied.jsonl | sort | uniq -c | sort -rn

# Cross-reference: find retrospect recs that are NOT yet applied
comm -23 \
  <(jq -r '.recommendations[].id' councils/retrospects.jsonl | sort -u) \
  <(jq -r '.rec_id' councils/applied.jsonl | sort -u)
```

## Relationship to history/runs.jsonl

`councils/retrospects.jsonl` supplements `history/runs.jsonl` — history tracks *what* ran and
whether it succeeded; retrospects track *how well* each stage performed and *what to improve*.
Join on `run_id` for a complete picture.
