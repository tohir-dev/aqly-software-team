# Run history schema

`history/runs.jsonl` is an **append-only log**. Each line is one JSON object written by the
`history-writer` agent immediately after a run completes (delivered, blocked, or abandoned).
Never edit or delete lines; this is an audit trail.

**JSON-safety rule.** `history-writer` MUST build the record as a data structure, serialize it
with a JSON library, self-validate the emitted line, and re-parse the whole file after appending
— see `agents/history-writer.md` § "How to emit the JSON line safely" for the full
protocol. One malformed line breaks the file for every downstream reader (COUNCIL mode ` status`,
`council-meta-retrospect`, `jq` queries), so this is not optional.

## Fields

```json
{
  "run_id":          "build-todo-app-20260628T143022Z",
  "type":            "build | improve",
  "project":         "todo-app",
  "target":          null,
  "request_summary": "A simple todo app with React frontend and localStorage persistence",
  "started_at":      "2026-06-28T14:30:22Z",
  "completed_at":    "2026-06-28T14:52:07Z",
  "duration_min":    22,
  "status":          "delivered | blocked | abandoned",
  "profile": {
    "has_ui": true,
    "has_data": false,
    "is_service": false,
    "has_users": true
  },
  "gates": {
    "qa":           "pass | fail | skipped",
    "perf":         "skipped",
    "reviewer":     "pass | fail | skipped",
    "accessibility":"skipped",
    "security":     "pass | fail | skipped",
    "red-team":     "skipped",
    "sre-readiness":"pass | fail | skipped"
  },
  "retries": {
    "qa": 0,
    "security": 1
  },
  "blocked_reason":  null,
  "pipeline_path":   "projects/todo-app/.pipeline",
  "branch":          null
}
```

### Field notes

| Field | Notes |
|---|---|
| `run_id` | `<type>-<project>-<YYYYMMDDTHHmmssZ>` (UTC) |
| `type` | `build` = greenfield; `improve` = existing project |
| `project` | kebab-case project name (or slug from IMPROVE mode request) |
| `target` | absolute path for IMPROVE mode runs; `null` for BUILD mode |
| `request_summary` | ≤ 1 sentence, extracted from `00-request.md` |
| `status` | `delivered` = all gates green; `blocked` = gate exhausted retries; `abandoned` = human stopped it |
| `profile` | copy of `01-prd.md` profile flags that were `true` |
| `gates` | one entry per gate that ran; `skipped` if flag was false |
| `retries` | gate name → total retry count (from STATUS.md attempt column) |
| `blocked_reason` | first open finding from `BLOCKED.md` if `status=blocked`; else `null` |
| `pipeline_path` | relative path to `.pipeline/` dir for cross-referencing artifacts |
| `branch` | `agentco/<slug>` for IMPROVE mode runs; `null` for BUILD mode |

## Querying history

Since `runs.jsonl` is newline-delimited JSON, query it with standard tools:

```bash
# All delivered builds
grep '"status":"delivered"' history/runs.jsonl | jq .

# Failed at a specific gate
jq 'select(.gates.qa == "fail")' history/runs.jsonl

# All improve runs
jq 'select(.type == "improve")' history/runs.jsonl

# Projects with retries
jq 'select(.retries | to_entries | map(.value) | add > 0)' history/runs.jsonl
```
