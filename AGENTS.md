# AGENTS.md — How AI agents work in this repo

## Read order (every session)
1. `OBJECTIVE.md` — what we build and what we do NOT build
2. `AGENTS.md` — this file
3. `PROGRESS.md` — where we are
4. `docs/ARCHITECTURE.md` — data model + API contract (source of truth)
5. The relevant `skills/<name>/SKILL.md`

## Repo layout (target)
```
/backend
  app/
    main.py            # FastAPI app, routers, CORS
    config.py          # env settings (pydantic-settings)
    db.py              # engine, session
    models.py          # SQLModel tables
    schemas.py         # Pydantic API + Spec/Plan schemas
    routers/           # workflows.py runs.py records.py export.py chat.py templates.py
    services/
      llm.py           # provider-agnostic LLM adapter (planner/verifier)
      perplexity.py    # Perplexity Search + Sonar wrapper, cache, cost caps
      planner.py       # prompt -> Spec + Plan
      collector.py     # search fan-out + extraction
      compliance.py    # robots/terms flags, allow/deny, rate limit
      processing.py    # normalize, validate
      dedupe.py        # keys, fuzzy match, merge
      verifier.py      # cross-check + trust score
      coverage.py      # coverage loop
      runner.py        # orchestrator, pause/cancel/retry, events
      diff.py          # run-vs-run diff
      exporter.py      # csv/json/xlsx
      dataset_chat.py  # chat over stored data
    prompts/           # *.md prompt templates (versioned, loaded from disk)
  tests/
  .env.example
/frontend              # Next.js App Router
/docs  /skills  OBJECTIVE.md  AGENTS.md  PROGRESS.md
```

## Golden rules
1. **Stay on objective.** If a task conflicts with OBJECTIVE.md, stop and ask.
2. **One generic pipeline.** Never write use-case-specific scraping code (no "jobs scraper"). Behavior comes from the LLM-generated Spec/Plan.
3. **Provenance is mandatory.** Any extracted value must carry `{value, source_url, snippet, confidence}`. No value without evidence. Never invent values; unknown → `null`.
4. **Contracts are frozen.** Don't rename tables, fields, or endpoints from `docs/ARCHITECTURE.md`. If a change is needed, propose it, update the doc first, then code.
5. **Verify external APIs.** Do NOT rely on memory for the Perplexity or Anthropic SDKs. Read the current docs (docs.perplexity.ai, docs.anthropic.com) or the installed package source, then code against what you verified. Note the verified signature in a comment.
6. **Small steps.** Work only on the current step from `docs/BUILD_PLAN.md`. Don't build ahead.
7. **Mock mode.** Every external call must work with `MOCK_MODE=true` using fixtures in `backend/tests/fixtures/`, so the app runs without keys and tests are deterministic.
8. **Cost & safety caps.** Every run has `max_queries`, `max_sources`, `max_rounds`, and a per-request timeout. Cache all Perplexity responses on disk keyed by request hash.
9. **Compliance-first.** Only public pages/APIs. Respect robots.txt flags, allow/deny lists, and rate limits. Log every source touched. Never bypass logins, paywalls, or CAPTCHAs.
10. **Typed and validated.** Pydantic for all LLM output; on validation failure retry up to 2 times with the error message, then mark the item failed (don't crash the run).
11. **Secrets.** Only in `.env` (git-ignored). Provide `.env.example`. Never log keys.
12. **Prompts live in files.** `backend/app/prompts/*.md`, loaded at runtime, not inlined in Python.

## Definition of Done (every step)
- [ ] Code runs (`uvicorn` / `npm run dev`) with `MOCK_MODE=true`
- [ ] Tests for new logic pass (`pytest -q`)
- [ ] Lint/type check clean (`ruff check`, `tsc --noEmit` for frontend)
- [ ] No contract drift from `docs/ARCHITECTURE.md`
- [ ] `PROGRESS.md` updated (what was done, decisions, TODOs)
- [ ] Short summary of changes + how to manually verify

## Coding conventions
- Python: type hints everywhere, `async def` for I/O, `httpx.AsyncClient`, small pure functions in `processing`/`dedupe`.
- Errors: raise domain exceptions, convert to HTTP errors in routers only.
- Frontend: TypeScript strict, server state via TanStack Query, no `any`, components in `components/`, API client in `lib/api.ts` generated from the contract types.
- Commits: conventional (`feat:`, `fix:`, `chore:`), one per step minimum.

## Skills (load before working in the area)
| Skill | Use when |
|---|---|
| `skills/planner-and-prompts` | Spec/Plan schema, planner/extractor/verifier prompts |
| `skills/perplexity-collector` | Search/Sonar calls, caching, compliance, collector |
| `skills/data-quality-pipeline` | normalize, validate, dedupe, trust score, coverage loop, diff |
| `skills/backend-runner` | FastAPI routers, DB, async runner, SSE, export |
| `skills/frontend-dashboard` | Any Next.js UI work |

## When unsure
Ask one concise question rather than guessing. Prefer the simplest thing that satisfies the current step.
