# ARCHITECTURE — Source of Truth (data model + API contract)

## Concepts
- **Workflow**: a saved definition (prompt + Spec + Plan). Shown as a "Task" in the UI.
- **Run**: one execution of a workflow. Records belong to a run. Refresh = new run; diff compares runs.
- **Record**: one row of the dataset. `data_json` holds typed fields per the Spec.
- **Evidence**: per-field proof (value + source_url + snippet + confidence).

## Pipeline
```
prompt -> planner -> Spec+Plan -> [user edits/approves]
run: coverage loop { seed/expanded queries -> Perplexity Search -> candidate sources
     -> Sonar structured extraction -> processing (normalize/validate)
     -> dedupe/merge -> verifier (cross-check) -> trust score -> gap analysis -> new queries }
     until yield < threshold or max_rounds/max_queries hit -> done
```

## Pydantic schemas (LLM-facing)
```python
FieldType = Literal["string","number","date","url","email","bool","enum"]

class FieldSpec(BaseModel):
    name: str                # snake_case
    type: FieldType
    description: str
    required: bool = False
    enum_values: list[str] | None = None

class Spec(BaseModel):
    entity_type: str                     # e.g. "job_posting"
    description: str
    fields: list[FieldSpec]              # 4-12 fields; MUST include a `source_url` field of type url
    key_fields: list[str]                # fields used for dedupe key
    filters: dict[str, Any] = {}         # geography, time_range, keywords, etc.
    target_count: int = 50

class QuerySpec(BaseModel):
    query: str
    domain_allow: list[str] = []
    domain_deny: list[str] = []
    recency: Literal["day","week","month","year","any"] = "any"
    purpose: str                         # why this query exists

class Plan(BaseModel):
    seed_queries: list[QuerySpec]
    source_strategy: str                 # plain-English explanation shown in plan preview
    extraction_instructions: str
    max_queries: int = 30
    max_sources: int = 60
    max_rounds: int = 4
    min_new_yield: float = 0.10          # stop when new-unique-records / total < this
```

## Tables (SQLModel) — JSON columns for dynamic data
| Table | Columns |
|---|---|
| workflows | id, name, prompt, spec_json, plan_json, is_template, created_at, updated_at |
| runs | id, workflow_id, run_number, status(queued/running/paused/completed/failed/cancelled), progress(0-100), stats_json, error, started_at, ended_at |
| steps | id, run_id, name, status, logs, started_at, ended_at |
| sources | id, run_id, url, domain, source_type(official/news/aggregator/social/other), robots_status(allowed/disallowed/unknown), status, fetched_at, records_extracted, round |
| records | id, run_id, dedupe_key, data_json, trust_score(0-100), diff_status(new/changed/unchanged/removed/null), needs_review(bool), source_count |
| evidence | id, record_id, field, value, source_url, snippet, confidence(0-1), verified(bool/null) |
| events | id, run_id, ts, level, type, message, payload_json |
| query_log | id, run_id, round, query, results_count, new_records, cache_hit |

`stats_json` example: `{"queries":12,"sources":34,"raw_records":140,"unique_records":87,"duplicates_merged":53,"flagged":9,"coverage_curve":[{"round":1,"queries":8,"unique":40},...],"cost_estimate_usd":0.42}`

## REST API (prefix /api)
| Method | Path | Purpose |
|---|---|---|
| POST | /plans | body `{prompt}` -> creates draft workflow with Spec+Plan (no run yet) |
| GET | /workflows | list (with last run summary) |
| GET | /workflows/{id} | detail incl. runs |
| PATCH | /workflows/{id} | edit name/spec/plan |
| POST | /workflows/{id}/runs | start a run -> Run |
| POST | /workflows/{id}/clone | duplicate workflow |
| POST | /workflows/{id}/template | mark as template |
| GET | /templates | list templates |
| GET | /runs/{id} | run detail + steps + stats |
| GET | /runs/{id}/events | **SSE** stream of events |
| POST | /runs/{id}/pause, /resume, /cancel, /retry | control |
| GET | /runs/{id}/records | query: q, sort, order, page, page_size, min_trust, needs_review, diff_status, plus `f.<field>=value` filters |
| GET | /records/{id} | record + evidence[] |
| PATCH | /records/{id} | user edit / mark reviewed |
| GET | /runs/{id}/sources | source inspector list |
| GET | /runs/{id}/diff | query: `against=<run_id>` (default previous) -> {new, changed, removed} |
| GET | /runs/{id}/insights | summary text + chart data |
| POST | /runs/{id}/chat | body `{message}` -> answer + cited record ids |
| GET | /runs/{id}/export | query: format=csv|json|xlsx |

## SSE event shape
`{"type":"step|log|progress|record|coverage|source|done|error","ts":"...","message":"...","payload":{}}`

## Trust score (0-100)
`50*agreement + 25*verifier + 15*source_authority + 10*completeness` where
agreement = min(1, (independent_sources-1)/2 + 0.5 if >=1 source), verifier = 1 if cross-check agrees / 0.5 unchecked / 0 disagree,
authority = official 1.0 / news .7 / aggregator .5 / social .3 / other .4, completeness = required fields filled ratio.
`needs_review = trust_score < 50 or verifier disagrees`.

## Compliance layer
- Perplexity Search/Sonar = discovery + extraction from public web
- `robots_status` flagged per domain (fetch robots.txt when we fetch anything directly; otherwise "unknown")
- allow/deny lists per workflow and global
- audit log: every query + source recorded in `query_log` + `sources`
