# BUILD_PLAN — Step-by-step Cursor prompts

## How to run this plan in Cursor (workflow)
1. **One step = one fresh Cursor chat.** Fresh context stays sharp; the md files carry memory between chats.
2. Use **Plan mode** first for Steps 3–8 (agent proposes plan -> you approve -> Agent mode executes).
3. Start each chat by pasting the step prompt. The `Read first` line makes the agent load the guardrails.
4. Use `@` to attach files named in the prompt (`@OBJECTIVE.md`, `@docs/ARCHITECTURE.md`, `@skills/...`).
5. After each step: run the **Verify** checklist yourself, then `git commit`. If broken, fix in the same chat before moving on.
6. If the agent drifts: reply "Re-read OBJECTIVE.md and AGENTS.md rule N, then revert the out-of-scope change."
7. Suggested models: strongest reasoning model for Steps 3, 4, 6, 7, 8; a fast model for UI Steps 9–13 and polish.

## Timeline (~20h)
| Steps | Focus | Hours |
|---|---|---|
| 0–2 | Setup, backend skeleton, API clients | 0–4 |
| 3–8 | Planner, collector, processing, verifier, coverage loop, runner | 4–11 |
| 9–13 | Frontend | 11–18 |
| 14 | Templates, seed data, polish, demo | 18–20 |

---

## STEP 0 — Repo bootstrap and guardrails
**Goal:** empty repo with the kit installed and both apps scaffolded.

**Do manually first:**
```
mkdir datascout && cd datascout && git init
# copy the kit into the repo root: OBJECTIVE.md AGENTS.md PROGRESS.md docs/ skills/ .cursor/
```
Create `backend/.env` from the keys: `PERPLEXITY_API_KEY`, `ANTHROPIC_API_KEY`, `MOCK_MODE=true`.

**Prompt:**
```text
Read first: @OBJECTIVE.md @AGENTS.md @PROGRESS.md @docs/ARCHITECTURE.md

STEP 0: Bootstrap the repository. Do NOT implement features yet.

1. Create /backend as a Python 3.11 project: pyproject.toml (or requirements.txt) with fastapi, uvicorn[standard], sqlmodel, pydantic-settings, httpx, rapidfuzz, dateparser, openpyxl, python-dotenv, pytest, pytest-asyncio, ruff, perplexity SDK (package name: verify on docs.perplexity.ai), anthropic SDK.
2. Create the backend/app package layout EXACTLY as listed in AGENTS.md (empty modules with docstrings are fine), plus backend/app/prompts/ and backend/tests/fixtures/.
3. backend/app/main.py: FastAPI app with CORS for http://localhost:3000 and GET /api/health returning {"status":"ok","mock_mode":<bool>}.
4. backend/app/config.py: pydantic-settings Settings (PERPLEXITY_API_KEY, ANTHROPIC_API_KEY, MOCK_MODE default true, DB_URL default sqlite:///./data.db, CACHE_DIR, PLANNER_MODEL, EXTRACTOR_MODEL, VERIFIER_MODEL as env-configurable strings, CONCURRENCY default 4).
5. backend/.env.example with all variables (no real secrets). Add .gitignore for python, node, .env, .cache, data.db.
6. Create /frontend with `npx create-next-app@latest` (TypeScript, Tailwind, App Router, ESLint, src dir = no). Init shadcn/ui and add: button, card, input, textarea, badge, tabs, table, sheet, dialog, dropdown-menu, select, switch, progress, skeleton, tooltip, sonner. Install @tanstack/react-query @tanstack/react-table recharts lucide-react.
7. Frontend .env.local.example with NEXT_PUBLIC_API_URL=http://localhost:8000.
8. Root README.md with run instructions for both apps.

Definition of done: `uvicorn app.main:app --reload` serves /api/health; `npm run dev` shows the default page; `ruff check` passes. Update PROGRESS.md (mark Step 0 complete). Summarize what you did.
```
**Verify:** health endpoint works, both dev servers start. Commit.

---

## STEP 1 — Database models and CRUD skeleton
**Skill:** `skills/backend-runner`

**Prompt:**
```text
Read first: @OBJECTIVE.md @AGENTS.md @PROGRESS.md @docs/ARCHITECTURE.md @skills/backend-runner/SKILL.md

STEP 1: Implement the database layer and basic (non-AI) API.

1. backend/app/models.py: SQLModel tables EXACTLY as in docs/ARCHITECTURE.md (workflows, runs, steps, sources, records, evidence, events, query_log). Use JSON columns (sa_column=Column(JSON)) for spec_json, plan_json, stats_json, data_json, payload_json. Add sensible indexes (records.run_id, records.dedupe_key, evidence.record_id, events.run_id).
2. backend/app/db.py: engine, get_session dependency, create_all on startup.
3. backend/app/schemas.py: Pydantic Spec, FieldSpec, QuerySpec, Plan (copy from ARCHITECTURE.md verbatim) plus API response models: WorkflowOut, RunOut, RecordOut, EvidenceOut, SourceOut, Paginated[T].
4. Routers (backend/app/routers/): workflows.py (GET list, GET detail, PATCH, POST clone, POST template), templates.py (GET /templates), runs.py (GET /runs/{id}), records.py (GET /runs/{id}/records with pagination, sorting, text search `q`, `min_trust`, `needs_review`, `diff_status`, and `f.<field>` filters; GET /records/{id} with evidence; PATCH /records/{id}), and GET /runs/{id}/sources. Do NOT create POST /plans or the run-start endpoint yet.
5. backend/app/seed.py + a CLI `python -m app.seed` that inserts one demo workflow (job_posting spec), one completed run and ~15 realistic records with evidence rows, so the frontend can be built later against real-shaped data. Make seeding idempotent.
6. Tests: pytest for records filtering/pagination/sorting and clone workflow, using an in-memory SQLite.

Constraints: do not deviate from table/field/endpoint names in ARCHITECTURE.md. No AI calls in this step.
Definition of done per AGENTS.md. Update PROGRESS.md.
```
**Verify:** `python -m app.seed`, then open `/docs` (Swagger) and query `/api/runs/1/records?q=...`.

---

## STEP 2 — LLM adapter and Perplexity client (with mock mode + cache)
**Skill:** `skills/perplexity-collector`

**Prompt:**
```text
Read first: @OBJECTIVE.md @AGENTS.md @PROGRESS.md @docs/ARCHITECTURE.md @skills/perplexity-collector/SKILL.md @skills/planner-and-prompts/SKILL.md

STEP 2: Build the two external-service wrappers. IMPORTANT: verify the real API signatures from current docs (use web/docs tools: docs.perplexity.ai for Search API + Sonar structured outputs + search filters; docs.anthropic.com for the Messages API) or from the installed SDK source. Do not code from memory. Put verified signatures in a header comment.

1. backend/app/services/llm.py: provider-agnostic `LLMClient` with `async def complete_json(prompt: str, schema_model: type[BaseModel], *, model: str, system: str|None=None, retries: int=2) -> BaseModel`. Implementation uses the Anthropic SDK. It requests JSON only, strips code fences, validates with Pydantic, and on ValidationError retries by appending the error to the prompt. Also `async def complete_text(...)`. MOCK_MODE returns fixtures from tests/fixtures/llm_<name>.json keyed by an optional `mock_key` argument.
2. backend/app/services/prompts.py: `render_prompt(name, **vars)` loading backend/app/prompts/<name>.md and replacing {{var}} placeholders (raise on missing var).
3. backend/app/services/perplexity.py per the skill: `search(...)`, `extract(...)`, disk cache with TTL, semaphore concurrency, exponential backoff, `RunBudget` with `BudgetExceeded`, timeouts, and MOCK_MODE using fixtures. `extract` must dynamically build a JSON Schema from Spec.fields (function `spec_to_json_schema(spec)`), with per-field {value, source_url, snippet, confidence}.
4. Create realistic fixtures in backend/tests/fixtures/ for a "AI startups hiring ML interns in Hyderabad" scenario: 3 search result files, 3 extraction result files (some duplicate companies, some messy dates/URLs), and llm fixtures for planner/gap analysis/verifier. Include 1–2 deliberately bad/hallucination-like rows for later testing.
5. A tiny script `python -m app.scripts.smoke_perplexity "<query>"` that runs a real search+extract when MOCK_MODE=false and prints results, so I can test my keys.
6. Tests: cache hit/miss, budget exceeded, spec_to_json_schema correctness, retry-on-validation-error logic (with a fake client), mock mode determinism.

Definition of done per AGENTS.md. Update PROGRESS.md incl. the verified API notes.
```
**Verify:** run the smoke script with real keys once (`MOCK_MODE=false`), confirm a real search works, then flip back to mock.

---

## STEP 3 — Planner (prompt -> Spec + Plan) and plan preview endpoints
**Skill:** `skills/planner-and-prompts`

**Prompt:**
```text
Read first: @OBJECTIVE.md @AGENTS.md @PROGRESS.md @docs/ARCHITECTURE.md @skills/planner-and-prompts/SKILL.md

STEP 3 (use Plan mode first): Implement the Planner.

1. backend/app/prompts/planner.v1.md: a high-quality system+user prompt that turns a natural-language business requirement into a Spec and Plan JSON (schemas in docs/ARCHITECTURE.md). Include: rules from the skill; 3 diverse few-shot examples (job openings, sponsor opportunities, sales leads — each with different fields/key_fields/seed queries); instruction to always add a `source_url` url-type field; a `warnings` array for unsafe/impossible requests (private data, login-only, personal sensitive info); seed queries diversified across synonyms, geographies, and source types with a `purpose` each.
2. backend/app/services/planner.py: `async def plan_from_prompt(prompt: str) -> PlannerResult` (Spec, Plan, warnings). Validate that Spec has source_url, key_fields ⊆ fields, 4–12 fields, 6–12 seed queries; if not, auto-repair once via LLM retry.
3. Router: POST /api/plans {prompt} -> creates a Workflow (draft) with spec_json/plan_json and returns WorkflowOut + warnings. Also PATCH /workflows/{id} must validate edited spec/plan with Pydantic (already exists — extend validation).
4. GET /api/workflows/{id} returns a `plan_preview` block: human-readable summary (entity, fields table, queries, source_strategy, limits, estimated max cost from limits).
5. Tests using MOCK_MODE fixtures + a param test for validation/repair logic. Add 3 fixture prompts.

Do NOT start runs in this step. Definition of done per AGENTS.md. Update PROGRESS.md.
```
**Verify (with real keys, optional):** try 3–4 varied prompts in Swagger and eyeball plan quality. Tune `planner.v1.md`; this prompt is your product's brain.

---

## STEP 4 — Collector and compliance layer
**Skill:** `skills/perplexity-collector`

**Prompt:**
```text
Read first: @OBJECTIVE.md @AGENTS.md @PROGRESS.md @docs/ARCHITECTURE.md @skills/perplexity-collector/SKILL.md

STEP 4: Implement collection for ONE round (no loop yet).

1. backend/app/prompts/extractor.v1.md per the skill's extractor rules (verbatim snippets, null for unknown, one record per entity, confidence rubric).
2. backend/app/services/compliance.py: `classify_source(url)`, `check_allowed(url, workflow_allow, workflow_deny, global_deny)`, `robots_status(domain)` (cached robots.txt via urllib.robotparser, async-safe, returns allowed/disallowed/unknown), and data/source_types.json (editable heuristics: official = company career/product domains + .gov/.edu/.org; news; aggregator e.g. linkedin/indeed/glassdoor/crunchbase; social). Include a default global deny list of login/paywall-heavy domains.
3. backend/app/services/collector.py: `async def collect_round(run_id, spec, plan, queries, session, emit) -> list[CandidateRecord]`. For each query with bounded concurrency: search -> drop denied hits -> insert `sources` rows (with source_type, robots_status, round) -> extract via Perplexity.extract -> attach evidence -> insert `query_log` row (results_count, cache_hit) -> `emit(...)` events (log/source). One failed query must not fail the round; log it and continue. Respect RunBudget.
4. CandidateRecord model: fields dict[str, FieldValue{value, source_url, snippet, confidence}], record-level source_url.
5. `emit` for now can be a simple callable that writes to the `events` table (SSE comes in Step 8).
6. Tests with MOCK_MODE: candidates returned, denied domains skipped, failed query isolated, query_log & sources persisted.

Definition of done per AGENTS.md. Update PROGRESS.md.
```
**Verify:** a temporary script/test runs `collect_round` on the fixture plan and prints candidates.

---

## STEP 5 — Processing: normalize, validate, dedupe/merge, persist
**Skill:** `skills/data-quality-pipeline`

**Prompt:**
```text
Read first: @OBJECTIVE.md @AGENTS.md @PROGRESS.md @docs/ARCHITECTURE.md @skills/data-quality-pipeline/SKILL.md

STEP 5: Implement processing.py and dedupe.py as PURE, heavily unit-tested functions, plus persistence.

1. processing.py: normalizers (string, date via dateparser incl. relative dates against a `fetched_at` arg, URL normalization with tracking-param stripping, email, number/range, enum), `validate_record(candidate, spec) -> ValidatedRecord` (invalid -> null + validation_issues, completeness ratio, reject records lacking all key_fields), and `build_dedupe_key(record, spec)`.
2. dedupe.py: exact-key merge, fuzzy merge using rapidfuzz token_set_ratio thresholds from the skill, per-field merge policy (non-null > higher confidence > more authoritative source), evidence union, `source_count` by distinct domains, and a `MergeReport`.
3. `persist_records(session, run_id, merged, existing) -> PersistStats` : upsert records (by run_id + dedupe_key) and evidence rows; return counts (new_unique, merged_duplicates, rejected).
4. Trust score function in verifier.py per docs/ARCHITECTURE.md formula (pure function, no LLM yet) with `needs_review` rule.
5. Tests (at least 25 cases): messy dates, URL variants, "Acme Inc" vs "ACME Technologies Pvt Ltd" merges, near-miss non-merges ("Acme Labs" vs "Acme Labs India" edge case documented), evidence preservation, trust score boundaries.
6. Wire fixtures: run collect_round (mock) -> process -> persist and assert the expected unique count.

Definition of done per AGENTS.md. Update PROGRESS.md.
```
**Verify:** the pytest output shows the merge cases pass; the DB shows records + multiple evidence rows per merged record.

---

## STEP 6 — Verifier (cross-check) and gap analysis
**Skills:** `data-quality-pipeline`, `planner-and-prompts`

**Prompt:**
```text
Read first: @OBJECTIVE.md @AGENTS.md @PROGRESS.md @skills/data-quality-pipeline/SKILL.md @skills/planner-and-prompts/SKILL.md

STEP 6: Implement verification and gap analysis.

1. prompts/verifier.v1.md: given ONE record and its evidence (field, value, snippet, url), return per-field {agrees: bool, reason}. Strict rule: a field agrees ONLY if the snippet explicitly supports the value. Output JSON only.
2. verifier.py: `async def verify_records(session, run_id, spec, *, limit)` — choose up to `limit` records prioritizing lowest confidence / single-source; call LLM with VERIFIER_MODEL (different from EXTRACTOR_MODEL by config); batch several records per call to save cost; update evidence.verified, record.needs_review, recompute trust_score with verifier signal.
3. prompts/gap_analysis.v1.md + coverage helper `async def propose_queries(spec, plan, dataset_summary, past_queries) -> list[QuerySpec]`: analyzes what is over/under-represented in the dataset (e.g., locations, company sizes, source types) and proposes 3–8 NEW queries. Hard-filter out any query that normalizes to a past query.
4. `build_dataset_summary(session, run_id, spec)`: compact stats (top values per categorical field, counts, nulls) small enough for a prompt.
5. Tests with mock fixtures: disagreement flags needs_review; trust recompute; duplicate query rejection; summary size cap.

Definition of done per AGENTS.md. Update PROGRESS.md.
```

---

## STEP 7 — Coverage loop
**Skill:** `skills/data-quality-pipeline`

**Prompt:**
```text
Read first: @OBJECTIVE.md @AGENTS.md @PROGRESS.md @docs/ARCHITECTURE.md @skills/data-quality-pipeline/SKILL.md

STEP 7 (Plan mode first): Implement coverage.py, the Coverage Loop, exactly per the skill pseudocode.

`async def run_coverage_loop(run_id, workflow, session, emit, control) -> CoverageResult`
- Each round: collect_round -> process/dedupe/persist -> verify_records -> compute new_unique & yield -> append {round, cumulative_queries, total_unique, new_unique} to stats_json.coverage_curve -> emit `coverage` event -> stop check (target_count reached OR yield < min_new_yield OR max_rounds OR RunBudget exhausted OR cancelled) -> else propose_queries and continue.
- `control` object exposes `await checkpoint()` that awaits when paused and raises RunCancelled if cancelled; call it between queries and between stages.
- Record the stop_reason in stats_json.
- Update stats_json live (queries, sources, raw_records, unique_records, duplicates_merged, flagged, cost_estimate_usd).
- Tests with a FAKE collector that returns scripted candidates per round to prove: stops on low yield, stops at target, never repeats queries, respects max_rounds/budget, cancel mid-round keeps partial data.

Definition of done per AGENTS.md. Update PROGRESS.md.
```
**Verify:** the mock run shows a coverage curve that rises then flattens.

---

## STEP 8 — Runner, SSE, controls, refresh diff, export
**Skill:** `skills/backend-runner`

**Prompt:**
```text
Read first: @OBJECTIVE.md @AGENTS.md @PROGRESS.md @docs/ARCHITECTURE.md @skills/backend-runner/SKILL.md

STEP 8 (Plan mode first): Make runs executable and manageable end-to-end.

1. services/runner.py: RunManager (per skill): start_run, pause/resume/cancel, retry, startup cleanup of stale running runs. Steps table entries per stage. Progress calc. Final stats. A failed run keeps partial results and an `error` message.
2. emit(): writes `events` row and pushes to per-run subscriber queues.
3. Routers: POST /api/workflows/{id}/runs; POST /api/runs/{id}/pause|resume|cancel|retry; GET /api/runs/{id}/events as SSE (replay stored events after ?after=<id>, then live, 15s heartbeat).
4. services/diff.py + GET /api/runs/{id}/diff?against=: compare with previous run by dedupe_key -> new/changed(with field-level old->new)/removed; set records.diff_status on completion of a run that has a predecessor.
5. services/exporter.py + GET /api/runs/{id}/export?format=csv|json|xlsx per the skill (Evidence sheet in XLSX).
6. Insights + chat services: GET /api/runs/{id}/insights (LLM summary + chart data: counts by top categorical fields, records by date, trust distribution) and POST /api/runs/{id}/chat: translate the question into filters over stored records (LLM returns a filter spec or record ids), answer ONLY from those records, return cited record ids. Prompts in prompts/dataset_chat.v1.md and insights.v1.md.
7. E2E test in MOCK_MODE: POST /plans -> POST run -> consume SSE until `done` -> records + evidence exist -> second run -> diff shows expected changes -> export CSV/XLSX opens.

Definition of done per AGENTS.md. Update PROGRESS.md with a curl cheat-sheet for all endpoints.
```
**Verify:** backend feature-complete. Try the whole flow through Swagger and `curl -N .../events`.

---

## STEP 9 — Frontend foundation, New Task and Plan Preview
**Skill:** `skills/frontend-dashboard`

**Prompt:**
```text
Read first: @OBJECTIVE.md @AGENTS.md @PROGRESS.md @docs/ARCHITECTURE.md @skills/frontend-dashboard/SKILL.md

STEP 9: Frontend foundation and first two screens.

1. lib/types.ts (mirror ARCHITECTURE.md types), lib/api.ts (typed fetch helpers for all endpoints), QueryClientProvider, sonner Toaster, dark mode toggle, app shell: sidebar (New Task, History, Templates) + top bar with mock-mode badge from /api/health.
2. `/` New Task screen: large textarea, 5 clickable example prompts (job openings, sponsors for a hackathon, sales leads, competitor pricing, market data), template gallery cards (GET /templates), "Generate plan" button -> POST /plans -> loading skeleton with "AI is designing your workflow…" -> redirect to /plan/[id]. Show planner `warnings` in an alert.
3. `/plan/[workflowId]`: PlanEditor — editable Spec fields table (name, type, required, description, add/remove row), key_fields multi-select, query list with purpose (add/edit/remove, per-query domain allow/deny chips), limits (max_queries, max_sources, max_rounds, target_count) as inputs/sliders, source_strategy text, estimated cost hint. Buttons: Save changes (PATCH), "Run workflow" (save then POST run -> redirect to /runs/[runId]).
4. Polished, data-dense visual design per the skill. Loading/empty/error states.

Definition of done: tsc --noEmit + eslint clean; works against the backend in MOCK_MODE. Update PROGRESS.md.
```

---

## STEP 10 — Task Monitor with live progress and Coverage Curve
**Prompt:**
```text
Read first: @OBJECTIVE.md @AGENTS.md @PROGRESS.md @skills/frontend-dashboard/SKILL.md @docs/ARCHITECTURE.md

STEP 10: Build /runs/[runId] header + Monitor tab.

1. lib/sse.ts useRunEvents(runId): EventSource with reconnect using ?after=<lastEventId>, dedupes events, exposes {events, latestByType, connected}.
2. Run header: workflow name, run #, status badge, progress bar, stats chips (queries, sources, unique records, duplicates merged, flagged for review, est. cost), controls (Pause/Resume/Cancel/Retry) wired to the API with optimistic UI.
3. RunTimeline: steps with status icons, durations, expandable logs; live log console (auto-scroll, filter by level).
4. CoverageChart (Recharts): x = cumulative queries, y = unique records, with a marker showing stop_reason when finished; animate points as SSE `coverage` events arrive. Add a small "records discovered" live ticker.
5. Tabs scaffold: Monitor · Results · Sources · Diff · Insights · Chat (only Monitor implemented now; others show a placeholder).
6. When the run completes, toast + switch to Results tab automatically.

Definition of done per AGENTS.md. Update PROGRESS.md.
```

---

## STEP 11 — Results Explorer, Evidence Drawer, Source Inspector
**Prompt:**
```text
Read first: @OBJECTIVE.md @AGENTS.md @PROGRESS.md @skills/frontend-dashboard/SKILL.md @docs/ARCHITECTURE.md

STEP 11: The core value screens.

1. ResultsTable (TanStack Table) with columns generated dynamically from spec.fields; server-side pagination/sort; global search (debounced) -> `q`; per-column filter popovers for string/enum/date/number; column visibility menu; sticky header; "needs review" and "min trust" filter toggles; trust badge column (green/amber/red) + source_count column + diff badge column when present; row click opens the drawer; selectable rows + bulk "mark reviewed".
2. EvidenceDrawer (shadcn Sheet): for each field show value, source domain favicon-less chip + link, the verbatim snippet with the value highlighted, confidence bar, verified/disagrees badge; a section "All sources for this record"; buttons: "Edit value" (PATCH, keeps original in history note), "Mark reviewed". This is the hero interaction — make it feel great.
3. SourceInspector tab: table of sources (domain, URL, type badge, robots status badge, records contributed, round, status), group-by-domain toggle, summary cards (total domains, % official, % unknown robots), an "Audit log" accordion listing every query from query_log (round, query, results, cache hit, new records). Allow/deny toggle per domain that PATCHes workflow plan lists (used on next run).
4. Export menu (CSV/JSON/XLSX) in the results toolbar.

Definition of done per AGENTS.md. Update PROGRESS.md.
```

---

## STEP 12 — History, clone/re-run, Refresh Diff
**Prompt:**
```text
Read first: @OBJECTIVE.md @AGENTS.md @PROGRESS.md @skills/frontend-dashboard/SKILL.md

STEP 12: Workflow and dataset history.

1. `/history`: list of workflows (name, prompt excerpt, entity type, #runs, last run status/time, last unique count) with search; expand row to show its runs with status, duration, counts, and actions: Open, Re-run (new run), Clone & edit (-> /plan/[newId]), Save as template, Compare with previous.
2. In `/runs/[runId]` header add a "Refresh" button (starts a new run of the same workflow with cache bypass) and a run selector to switch between runs.
3. DiffView tab: summary cards (New / Changed / Removed / Unchanged counts), three sections; Changed shows field-level old -> new with strikethrough/green styling; clicking a row opens the EvidenceDrawer. "Compare against" dropdown to pick any earlier run.
4. Templates page/gallery (`GET /templates`) with "Use template" -> clones into the plan editor.

Definition of done per AGENTS.md. Update PROGRESS.md.
```

---

## STEP 13 — Insights, Dataset Chat, compliance polish
**Prompt:**
```text
Read first: @OBJECTIVE.md @AGENTS.md @PROGRESS.md @skills/frontend-dashboard/SKILL.md

STEP 13: Intelligence layer UI.

1. InsightsPanel: AI summary card + "3 things worth noticing", Recharts charts from /insights (top categorical fields bar charts, records over time, trust distribution histogram, source-type donut).
2. DatasetChat tab: chat UI against POST /runs/{id}/chat; suggestion chips generated from the spec ("Which have the highest trust score?"); assistant answers show cited records as clickable chips that open the EvidenceDrawer; a visible note "Answers come only from this dataset".
3. Compliance badge/panel in the run header: shows "N permitted sources · X robots unknown · 0 denied hits used" with a popover listing the policy (public sources only, allow/deny lists, audit log link).
4. Global empty/error/loading polish sweep across all screens; keyboard shortcuts (/ focuses search, Esc closes drawer); responsive check at tablet width.

Definition of done per AGENTS.md. Update PROGRESS.md.
```

---

## STEP 14 — Templates, seed demos, hardening, demo script
**Prompt:**
```text
Read first: @OBJECTIVE.md @AGENTS.md @PROGRESS.md @docs/ARCHITECTURE.md

STEP 14: Ship-readiness. No new features beyond this list.

1. Four workflow templates seeded via `python -m app.seed --templates`: Job Radar, Sponsor Finder, Lead Gen, Competitor Pricing — each with a tuned Spec + Plan (fields, key_fields, seed queries with purposes).
2. Demo seeding: `python -m app.seed --demo` creates 2 finished workflows with 2 runs each (so Diff has real data), covering ~60 and ~40 records with varied trust scores, some needs_review rows, multi-source evidence, and coverage curves. Cache warm-up script `python -m app.scripts.warm_cache "<prompt>"` to pre-run real tasks and store cached Perplexity responses for offline demo reliability.
3. Hardening: global exception handler, request timeouts, input length limits on prompts, sanitize filenames in export, rate-limit POST /plans (simple in-memory), consistent error JSON, log redaction of keys.
4. README: architecture diagram (mermaid), setup, env vars, how MOCK_MODE works, demo script (exact 3-minute walkthrough: prompt -> plan edit -> live run -> coverage curve -> evidence drawer -> compliance panel -> export -> refresh diff -> dataset chat), and a "Why not just Perplexity?" section.
5. Run the full test suite, ruff, tsc, eslint; fix everything. Update PROGRESS.md with final status and known limitations.
```

---

## Cross-step recovery prompts (copy when needed)
**Agent drifted:**
```text
Stop. Re-read @OBJECTIVE.md and @AGENTS.md. List which of your changes violate the golden rules or the current step's scope, revert them, and continue only with the step requirements.
```
**Contract change needed:**
```text
Propose the change to docs/ARCHITECTURE.md as a diff with justification and list every file affected. Do not code until I approve.
```
**Something is broken:**
```text
Reproduce the failure first (show the exact command and error). Find the root cause, propose the minimal fix, apply it, add a regression test, and confirm all tests pass. Don't refactor unrelated code.
```
**Context is long/messy:** open a new chat, paste the current step prompt again with "Continue Step N. Check git diff and PROGRESS.md to see what's already done."
**Before final demo:**
```text
Audit the repo against the Requirements checklist in @OBJECTIVE.md. For each item, cite the file/feature that satisfies it, and list gaps. Do not fix yet, just report.
```
