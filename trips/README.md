# Trips

Each trip planned through the Jarvis workflow gets its own folder here, named
`<year>-<month>-<destination-short>` (e.g. `2026-08-japan`).

A trip folder contains:

- `brief.md` — the customer requirements Jarvis gathers before delegating:
  locations, dates, budget, style_tags, travelers.
- `report.md` — Jarvis's final compiled report: day-by-day itinerary woven
  with weather tips and route notes.
- `budget.xlsx` — the financial associate's (Friday's) cost breakdown and
  budget comparison.

Folders are created when a trip is actually planned — there is no template
to copy. The required fields for `brief.md` and `report.md` are defined in
each agent's CONTRACT section under `.claude/agents/tripPlanWorkflow/`.
