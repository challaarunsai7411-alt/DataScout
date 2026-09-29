---
name: perplexity-collector
description: Use when writing or changing anything that calls Perplexity (Search API, Sonar), the collector, caching, cost caps, or the compliance layer.
---

# Perplexity Collector

## FIRST: verify the API
Do not code from memory. Read https://docs.perplexity.ai (Search API, Sonar/chat completions, structured outputs, search filters) or inspect the installed `perplexity` SDK. Confirm: client init, `search.create` params, chat completion params for JSON schema output, filter params (domain, recency, location), response shape (citations/search_results). Record verified signatures in a comment at the top of `services/perplexity.py`.

## Two-tool strategy
1. **Discovery** — Perplexity Search API: ranked results (title, url, snippet) for each `QuerySpec`. Use `max_results` ~10. Apply `domain_allow/deny` and recency filters where supported.
2. **Extraction** — Sonar with structured JSON output constrained to the Spec's field schema (build a JSON Schema from `Spec.fields` dynamically). Give it the query + the candidate URLs/snippets as context and the `extraction_instructions`. Ask for evidence (snippet + url) per field.
Use the returned citations/search_results to populate `source_url` and to validate that each cited URL was actually returned by the search.

## Wrapper requirements (`services/perplexity.py`)
- `async def search(query, *, max_results, domain_allow, domain_deny, recency) -> list[SearchHit]`
- `async def extract(spec, plan, query, hits) -> list[CandidateRecord]`
- Disk cache: `.cache/perplexity/<sha256(request)>.json`; return `cache_hit` flag; TTL configurable (default 24h; refresh runs may bypass).
- Concurrency limit via `asyncio.Semaphore` (default 4) + exponential backoff on 429/5xx (max 3 tries).
- Cost caps: increment counters on a `RunBudget` object; raise `BudgetExceeded` when `max_queries` reached; record estimated cost in stats.
- `MOCK_MODE`: load `tests/fixtures/search_*.json` / `extract_*.json` chosen by hash of query so results are deterministic.
- Never log API keys. Timeouts on every request (30s).

## Compliance (`services/compliance.py`)
- `classify_source(url) -> source_type` (official / news / aggregator / social / other) via simple domain heuristics + TLD + a small editable JSON list.
- `check_allowed(url, workflow) -> Allowed|Denied(reason)` using global + workflow allow/deny lists.
- `robots_status(domain)`: if we ever fetch a page directly, fetch and cache robots.txt (urllib.robotparser) and respect it; otherwise `"unknown"` and say so in UI.
- Skip login/paywall domains from a deny list; never attempt to bypass.
- Every query -> `query_log`; every source -> `sources`. This is the audit trail.

## Collector flow
`collect_round(run, queries) -> list[CandidateRecord]`: for each query (bounded concurrency): search -> filter via compliance -> extract -> attach evidence -> emit SSE events (`log`, `source`, `record`) -> return candidates. Failure of one query must not fail the round.
