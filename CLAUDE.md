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
  Gathers requirements (locations, dates, budget, style_tags, travelers),
  delegates to the three associates below, reviews their output against a
  required-fields contract, resolves conflicts between them, and compiles
  one cohesive final report. Never builds routes, checks weather, or does
  cost math itself.
- `trip-planner-associate` (Monday) — builds the day-by-day route/itinerary
  from Jarvis's brief.
- `weather-associate` (Fiona) — produces a per-leg weather report (forecast
  vs. seasonal norm, packing/timing tips) from Monday's itinerary. Dispatched
  in parallel with Friday.
- `financial-associate` (Friday) — produces the cost breakdown and budget
  comparison (as an Excel file) from Monday's itinerary. Dispatched in
  parallel with Fiona.

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

## Trip storage

Each planned trip gets its own folder under
`trips/<year>-<month>-<destination-short>/` (e.g. `trips/2026-08-japan/`) —
see `trips/README.md` for the convention:
- `brief.md` — the customer requirements Jarvis gathered
- `report.md` — Jarvis's final compiled report
- `budget.xlsx` — Friday's financial breakdown

No trip folders exist yet in this repo; they're created the first time a
trip is actually planned.
