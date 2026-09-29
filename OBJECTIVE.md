# OBJECTIVE — DataScout (working name)

> This file is the north star. Every agent session MUST read it first.
> If a task conflicts with this file, STOP and ask the human. Do not silently deviate.

## One-liner
Describe the data you need in plain English → AI designs a collection workflow → collects from permitted sources via Perplexity APIs → cleans, validates, dedupes → results appear in a dashboard with **per-cell source evidence**, and the workflow can be **re-run and diffed**.

## The problem (from the problem statement)
Businesses need specific web data (job openings, sales leads, sponsor opportunities, market data). Building a separate scraper per need is slow and unscalable. We build ONE generic, prompt-driven platform.

## What we are NOT building
- NOT a chatbot or an answer engine. Perplexity answers questions; we produce **managed datasets**.
- NOT per-use-case scrapers. One generic pipeline driven by an LLM-generated JSON spec + plan.
- NOT a system that scrapes protected/login/paywalled/robots-disallowed content.

## Core loop (must always work end-to-end)
1. **Prompt** → Planner LLM → `Spec` (entity, typed fields, filters, target count) + `Plan` (queries, source strategy, limits)
2. **Plan preview** → user can edit/approve (human in the loop)
3. **Run** → Collector fans out Perplexity Search/Sonar calls → structured extraction → rows
4. **Process** → normalize → validate → dedupe/merge → verify (cross-check) → trust score
5. **Coverage Loop** → find what's missing, generate new queries, repeat until yield flattens
6. **Dashboard** → live monitor, results table, evidence drawer, source inspector
7. **Manage** → history, clone/re-run, refresh diff (new/changed/removed), export CSV/JSON/XLSX

## Requirements checklist (mirrors the problem statement)
- [ ] Understand data requirements from natural language
- [ ] Dynamically design + execute workflows (no hardcoded per-domain code)
- [ ] Collect from multiple permitted sources
- [ ] Clean, structure, validate, deduplicate
- [ ] Source-backed, traceable data (every field value has URL + snippet)
- [ ] Interactive dashboard: search, filter, sort, export
- [ ] Monitor and manage tasks (progress, logs, pause/cancel/retry)
- [ ] Workflow + dataset history

## Differentiators (protect these — they are the pitch)
1. **Coverage Loop** with live coverage curve (records vs queries)
2. **Per-cell provenance** + Trust Score (multi-source agreement + verifier + source authority)
3. **Cross-model verification** (extractor vs verifier disagree → flagged for review)
4. **Plan preview/edit** (glass box, not black box)
5. **Refresh with diff** (new / changed / removed)
6. **Compliance panel** (permitted sources, robots/terms flag, allow/deny lists, audit log)
7. **Chat over the dataset** (answers from stored data, not fresh web guesses)
8. **Workflow templates** (Sponsor Finder, Job Radar, Lead Gen, Competitor Pricing)

## Tech stack (fixed — do not swap without asking)
- Backend: Python 3.11+, FastAPI, SQLModel/SQLAlchemy, SQLite, asyncio tasks (NO Celery/Redis)
- Data collection: Perplexity APIs (Search API for discovery, Sonar w/ structured output for extraction)
- Planner/Verifier LLM: Anthropic Claude via a provider-agnostic `llm.py` adapter
- Frontend: Next.js (App Router) + TypeScript + Tailwind + shadcn/ui + TanStack Table + Recharts
- Live updates: Server-Sent Events
- Tests: pytest (backend), a few Playwright/vitest smoke tests optional

## Success criteria for the demo
A judge types: "Find AI startups hiring ML interns in Hyderabad" and within ~2 minutes sees a
deduped, evidence-backed table, a coverage curve, a source list, can click any cell to see its
source snippet, export to CSV, and re-run the workflow to see a diff.

## Scope discipline
Priority order: core loop > provenance > coverage loop > refresh diff > everything else.
No auth, no multi-user, no payments, no deployment infra unless explicitly requested.
