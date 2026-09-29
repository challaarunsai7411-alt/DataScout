---
name: frontend-dashboard
description: Use for any Next.js/React UI work — screens, components, API client, SSE hooks, tables, charts, and visual design of the dashboard.
---

# Frontend Dashboard

## Stack & structure
Next.js App Router + TypeScript (strict) + Tailwind + shadcn/ui + TanStack Query + TanStack Table + Recharts. Backend base URL from `NEXT_PUBLIC_API_URL`. API types in `lib/types.ts` mirror `docs/ARCHITECTURE.md`; fetch helpers in `lib/api.ts`. SSE hook `useRunEvents(runId)` in `lib/sse.ts` (EventSource, reconnect, dedupe by event id).

## Routes
| Route | Screen |
|---|---|
| `/` | New Task: prompt box, example chips, template gallery |
| `/plan/[workflowId]` | Plan preview/edit -> Run button |
| `/runs/[runId]` | Task Monitor (timeline, progress, logs, coverage curve, controls) with tabs: Results · Sources · Diff · Insights · Chat |
| `/history` | Workflows + runs, clone / re-run / compare |

## Design direction
Clean, data-dense, professional (think Linear/Vercel). Neutral palette + one accent. Dark mode via Tailwind `dark:`. Trust badges: green >=75, amber 50–74, red <50. Generous empty/loading/error states (skeletons, not spinners). Responsive down to tablet.

## Key components
- `PlanEditor`: editable schema table (name/type/required), query list (add/remove/edit), limits sliders, source_strategy text.
- `RunTimeline`: steps with status icons + collapsible logs; `ProgressBar`; `RunControls` (pause/resume/cancel/retry).
- `CoverageChart`: Recharts line of unique records vs cumulative queries, updates live via SSE.
- `ResultsTable`: TanStack Table, dynamic columns from Spec, column visibility toggle, global search, per-field filters, sort, server pagination, trust badge column, row-click opens drawer.
- `EvidenceDrawer`: per-field list: value, source domain + link, verbatim snippet (highlight the value), confidence, verified badge, "mark reviewed / edit value".
- `SourceInspector`: table of domains/URLs with type, robots status, records contributed, round; allow/deny toggles.
- `DiffView`: New / Changed (old -> new) / Removed sections.
- `ExportMenu`, `DatasetChat`, `InsightsPanel`.

## Rules
- No hardcoded columns: everything derives from `spec.fields`.
- Every async view has loading, empty and error states.
- Keep components small; colocate hooks; no global state library beyond TanStack Query.
- Accessibility: focus states, aria labels on icon buttons, keyboard-closable drawer.
