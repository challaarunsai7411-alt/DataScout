---
name: backend-runner
description: Use for FastAPI routers, SQLModel tables, migrations-free DB setup, the async run orchestrator, SSE events, pause/cancel/retry, and exports.
---

# Backend & Runner

## Setup
FastAPI + SQLModel + SQLite (file `backend/data.db`), `create_all` at startup (no Alembic). CORS allow `http://localhost:3000`. Settings via pydantic-settings from `.env`. Follow `docs/ARCHITECTURE.md` exactly.

## Runner (`services/runner.py`)
- `RunManager` singleton holds `dict[run_id, RunHandle]` with an `asyncio.Task`, a `pause_event`, and a `cancel_flag`.
- `start_run(workflow_id)` creates a `Run`, spawns the task, returns immediately.
- Steps recorded in `steps` table: `plan_loaded -> round_1 -> process_1 -> verify_1 -> ... -> finalize -> diff`. Each has status + logs + timestamps.
- Check `pause_event`/`cancel_flag` between steps and between queries. Pause = await event; cancel = raise `RunCancelled`, mark `cancelled`, keep partial results.
- `retry` = new run for the same workflow (or resume failed step if trivial). Never delete old runs.
- Progress = weighted estimate (rounds done / max_rounds, or unique/target_count, whichever is greater), capped at 99 until finalize.
- On startup: mark any `running` runs from a previous process as `failed` ("server restarted").

## Events & SSE
- Every important action calls `emit(run_id, type, message, payload)` which (1) inserts into `events`, (2) pushes to per-run `asyncio.Queue`s of subscribers.
- `/runs/{id}/events` streams via `StreamingResponse(media_type="text/event-stream")`; on connect, first replay stored events (support `?after=<event_id>`), then live. Send a heartbeat comment every 15s.

## Records API
- Pagination + sorting + text search over `data_json` (SQLite JSON functions or in-Python filter for MVP — prefer correctness over cleverness; page size default 50).
- `f.<field>=value` equality/contains filters, `min_trust`, `needs_review`, `diff_status`.
- `GET /records/{id}` returns record + evidence grouped by field.

## Export (`exporter.py`)
CSV/JSON/XLSX (`openpyxl`). Columns = Spec fields + `trust_score`, `source_count`, `needs_review`, plus a `sources` column (semicolon-joined URLs). Include a second XLSX sheet "Evidence" (record key, field, value, source_url, snippet). `Content-Disposition` filename `<workflow-name>_run<N>.<ext>`.

## Quality bar
pytest with `httpx.AsyncClient` + `MOCK_MODE`. One end-to-end test: create plan -> run -> wait done -> records exist with evidence -> export CSV parses.
