# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This repo has no executable code, build system, linter, or test suite — there
are no commands to run. It defines a multi-agent trip-planning workflow as
Claude Code subagents, plus a storage convention for the trips they produce.

## Architecture

Four project-scoped subagents live in `.claude/agents/tripPlanWorkflow/` and
together form the workflow:

- `trip-planner-manager` (Jarvis) — the customer's single point of contact.
  Before interviewing the customer, checks `memory/` for standing
  preferences and past trips, and confirms anything relevant with the
  customer rather than assuming it still applies. Gathers requirements
  (locations, dates, budget, style_tags, travelers), delegates to the
  three associates below, reviews their output against a required-fields
  contract, resolves conflicts between them, and compiles one cohesive
  final report. Never builds routes, checks weather, or does cost math
  itself. After compiling the final report, appends a short entry to
  `memory/history.md`.
- `trip-planner-associate` (Monday) — builds the day-by-day route/itinerary
  from Jarvis's brief.
- `weather-associate` (Fiona) — produces a per-leg weather report (forecast
  vs. seasonal norm, packing/timing tips) from Monday's itinerary. Dispatched
  in parallel with Friday.
- `financial-associate` (Friday) — produces the cost breakdown and budget
  comparison (as an Excel file) from Monday's itinerary. Dispatched in
  parallel with Fiona.
- `trip-planner-consolidator` — a Read-only agent, invoked manually (never
  by Jarvis, never automatically) after a batch of trips. Reads
  `memory/history.md` and drafts candidate planning heuristics with
  supporting evidence for a human to review; it cannot write
  `memory/heuristics.md` itself.

When acting as Jarvis, delegation must go through actual `Agent` tool calls
using these `subagent_type` values — never write an associate's section
yourself under their name:
- Monday → `subagent_type: "trip-planner-associate"`
- Fiona → `subagent_type: "weather-associate"`
- Friday → `subagent_type: "financial-associate"`

Each agent's full SOUL/TONE/CONTRACT/GUARDRAILS/FALLBACK spec lives in its
own file under `.claude/agents/tripPlanWorkflow/` — read the relevant file
before acting as that agent rather than relying on the summary below. In
particular, each agent caps revisions at 2 rounds and has specific fallback
behavior for missing or unavailable data that isn't repeated here.

## Data flow / contract

- Jarvis → associates: `locations`, `dates`, `style_tags`, `budget` (or
  `none provided`), `travelers`.
- Monday → Jarvis: day-by-day itinerary (day number, date, location,
  activities, transitions). Every input location must appear in the
  itinerary or be explicitly flagged as dropped/merged.
- Fiona → Jarvis: per-leg conditions, a `source_type` label (`forecast` vs.
  `seasonal_norm`), flagged risks, packing/timing tips.
- Friday → Jarvis: cost breakdown by category with `firm`/`estimated` labels
  on every line item, subtotals, grand total, and budget variance if a
  budget was given.

Jarvis sends an associate's output back for completion, rather than
forwarding it, if it's missing any required field above.

## Customer memory

`memory/` at the repo root holds standing customer context that persists
across trips — separate from any per-trip data in `trips/`:

- `memory/preferences.md` — standing travel preferences (pace, lodging,
  dietary/accessibility, transit, budget tendencies).
- `memory/history.md` — a running log of past trips, appended to by
  Jarvis after each final report is compiled, including a `Revisions:`
  outcome signal per associate.
- `memory/heuristics.md` — planning heuristics distilled from
  `history.md` by the `trip-planner-consolidator` agent and promoted by
  a human. Unlike the other two files, Jarvis applies these without
  customer confirmation, since they're process lessons rather than
  customer-specific facts.

Only Jarvis reads or writes `memory/`. Monday, Fiona, and Friday never
see it directly — Jarvis folds anything relevant into the `style_tags`/
`budget` fields it already passes downstream, so the associates keep
working from one consistent, filtered brief. Jarvis always confirms
memory-derived facts with the customer rather than applying them
silently, and treats a missing or empty `memory/` as normal, not an
error.

## Self-growing loop

The workflow closes the loop from raw experience to better planning in
four steps, only the first of which existed before this addition:

1. **Logging** — Jarvis appends an entry to `memory/history.md` after
   every trip.
2. **Evaluation** — that entry includes a `Revisions:` line: how many
   rounds each associate needed and whether the 2-round cap was hit.
   This is Jarvis's own existing tracking, just persisted instead of
   discarded.
3. **Consolidation** — run `trip-planner-consolidator` manually (it is
   never invoked automatically or by Jarvis) after enough trips have
   accumulated. It reads `memory/history.md` and drafts candidate
   heuristics with the evidence behind each one. It has no `Write` tool
   and cannot persist anything — its output is a draft only.
4. **Policy update** — a human reviews that draft and hand-writes the
   approved heuristics into `memory/heuristics.md`. Jarvis reads this
   file on every subsequent trip and folds applicable heuristics
   directly into the brief, with no customer confirmation step (unlike
   preferences/history facts, which are always confirmed first).

## Trip storage

Each planned trip gets its own folder under
`trips/<year>-<month>-<destination-short>/` (e.g. `trips/2026-08-japan/`) —
see `trips/README.md` for the convention:
- `brief.md` — the customer requirements Jarvis gathered
- `report.md` — Jarvis's final compiled report
- `budget.xlsx` — Friday's financial breakdown

No trip folders exist yet in this repo; they're created the first time a
trip is actually planned.
