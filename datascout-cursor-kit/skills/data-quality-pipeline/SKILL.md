---
name: data-quality-pipeline
description: Use when implementing normalization, validation, deduplication/merging, verification, trust score, coverage loop, or run-vs-run diff.
---

# Data Quality Pipeline

## Normalize (`processing.py`) — pure functions, unit-tested
- Strings: trim, collapse whitespace, unicode NFKC. Company/org names: strip suffixes (Inc, Pvt Ltd, LLC, Technologies…) ONLY for key building, keep display value original.
- Dates -> ISO-8601 (`dateparser`), relative dates ("2 days ago") resolved against fetch time.
- URLs -> strip tracking params (utm_*, gclid), lowercase host, drop fragment; must be http(s).
- Emails -> lowercase, regex-valid. Numbers -> float; salary ranges -> min/max when applicable.
- Enums -> map to nearest allowed value or null.

## Validate
- Per-field validators from `FieldSpec.type`. Invalid value -> set to null, keep original in evidence with `confidence=0`, add `validation_issues` list on the record.
- Reject records missing ALL `key_fields`. Compute `completeness` = required fields present / required total.

## Dedupe & merge (`dedupe.py`)
- `dedupe_key` = normalized join of `Spec.key_fields`.
- Exact key match -> merge. Otherwise fuzzy: `rapidfuzz.fuzz.token_set_ratio >= 90` on key string (and same domain of `source_url` optionally) -> merge candidate; 80–90 -> merge only if another field (e.g. url or location) matches.
- Merge policy per field: prefer non-null; prefer higher confidence; prefer more authoritative source; keep ALL evidence rows from all sources. `source_count` = distinct source domains.
- Return `MergeReport` (merged pairs + reason) for the audit log/UI.

## Verify & trust (`verifier.py`)
- Cross-check: a different model than the extractor (configurable) receives the record + its evidence snippets and returns per-field `agrees` + reason. Disagree -> `needs_review = true`, `evidence.verified=false`.
- Only verify records above a cost budget (`verify_limit`), prioritizing low-confidence ones.
- Trust score formula lives in `docs/ARCHITECTURE.md` — implement exactly, as a pure function with tests.

## Coverage loop (`coverage.py`)
```
round = 1; queries = plan.seed_queries
while round <= max_rounds and budget_ok:
    candidates = collect_round(queries)
    new_unique = process_and_merge(candidates)      # count records not seen before
    yield_ = new_unique / max(1, total_unique)
    record coverage point {round, cumulative queries, total_unique}; emit SSE `coverage`
    if total_unique >= target_count or yield_ < min_new_yield: stop
    queries = gap_analysis(spec, dataset_summary, past_queries)   # LLM proposes NEW queries only
    round += 1
```
Never repeat a past query (normalize + compare). Stop reason stored in stats.

## Diff (`diff.py`)
Compare run N vs previous run by `dedupe_key`: `new`, `removed`, `changed` (which fields differ, old->new), `unchanged`. Persist `diff_status` on records of run N; removed records are returned from the previous run's rows.

## Testing
Unit tests with tiny hand-made fixtures for: URL normalization, date parsing, fuzzy merge edge cases, trust score, coverage stop conditions (fake collector), diff.
