---
name: planner-and-prompts
description: Use when designing or editing the Spec/Plan schema, the planner, extractor, verifier or gap-analysis prompts, or any LLM structured-output logic.
---

# Planner & Prompts

## Principles
- The LLM outputs **data** (Spec + Plan JSON), never code. The engine is fixed.
- All LLM output is parsed by Pydantic. On failure: retry up to 2x, passing the validation error back. Then fail the item gracefully.
- Prompts are markdown files in `backend/app/prompts/`, with `{{placeholders}}`, loaded via a tiny `render_prompt(name, **vars)` helper. Version them (`planner.v1.md`).
- Temperature low (0–0.2) for planner/extractor/verifier.

## Prompt files to maintain
| File | Input | Output |
|---|---|---|
| `planner.v1.md` | user prompt | `Spec` + `Plan` JSON |
| `extractor.v1.md` | Spec, Plan.extraction_instructions, query | array of candidate records, each field as `{value, source_url, snippet, confidence}` |
| `verifier.v1.md` | one record + its evidence snippets | per-field `{agrees: bool, reason}` |
| `gap_analysis.v1.md` | Spec, current dataset summary, past queries | 3–8 new `QuerySpec`s targeting missing coverage |
| `dataset_chat.v1.md` | question, filtered records | answer + cited record ids |
| `insights.v1.md` | dataset stats | summary + "3 things worth noticing" |

## Planner rules
- Always include a `source_url` field (type url) in Spec.fields.
- 4–12 fields. Prefer fields findable on public pages. Mark only truly essential ones `required`.
- `key_fields` = the minimal set uniquely identifying a record (e.g. company+title for jobs; org name for sponsors).
- Seed queries: 6–12, diverse (synonyms, sub-geographies, source types: official sites, news, directories). Each has a `purpose`.
- `source_strategy` is human-readable, 2–4 sentences, shown in plan preview.
- Refuse/flag requests needing private data, login-only content, or personal sensitive data; return a `warnings` list instead of pretending.

## Extractor rules (anti-hallucination)
- Extract only what the source text supports. Unknown → `null`. Never guess.
- `snippet` must be a verbatim excerpt (<=300 chars) from the source supporting the value.
- One record per real-world entity. Include `source_url` per record.
- Confidence: 1.0 explicit statement, 0.7 clearly implied, 0.4 weak, otherwise null.
