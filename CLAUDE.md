# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This repo has no executable code, build system, linter, or test suite — there
are no commands to run. It defines a multi-agent trip-planning workflow as
Claude Code subagents, plus a storage convention for the trips they produce.

## Architecture

Five project-scoped subagents live in `.claude/agents/tripPlanWorkflow/`;
four of them form the per-trip workflow, and a fifth runs out-of-band:

- `trip-planner-manager` (Jarvis) — the customer's single point of contact.
  Before interviewing the customer, checks `memory/` for standing
  preferences and past trips, and confirms anything relevant with the
  customer rather than assuming it still applies (heuristics excepted —
  see Customer memory). Gathers requirements
  (locations, dates, budget, style_tags, travelers), delegates to the
  three associates below, reviews their output against a required-fields
  contract, resolves conflicts between them, and compiles one cohesive
  final report. Never builds routes, checks weather, or does cost math
  itself. After compiling the final report, creates a new page under
  `memory/history/`.
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
  `memory/history/` and drafts candidate planning heuristics with
  supporting evidence for a human to review; it cannot write
  `memory/heuristics/` itself.

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
across trips — separate from any per-trip data in `trips/`. It's
structured as a small wiki, not a pile of flat logs, so it stays cheap to
read as trips accumulate:

- `memory/INDEX.md` — the entry point. One line per page in `history/`
  and `heuristics/`, each carrying its `destination`/`tags` so a reader
  can tell which pages are relevant to the trip at hand without opening
  all of them. Always read this before anything else under `memory/`.
- `memory/preferences.md` — standing travel preferences (pace, lodging,
  dietary/accessibility, transit, budget tendencies). One file, always
  read in full — it's small and always relevant.
- `memory/history/` — one page per past trip (`<year>-<month>-<slug>.md`),
  appended to by Jarvis after each final report is compiled. Each page's
  frontmatter carries `destination`, `style_tags`, and a `revisions`
  outcome signal per associate; see `memory/history/README.md` for the
  exact format.
- `memory/heuristics/` — one page per planning heuristic, distilled from
  `history/` by the `trip-planner-consolidator` agent and promoted by a
  human. Unlike the other two, Jarvis applies these without customer
  confirmation, since they're process lessons rather than
  customer-specific facts. See `memory/heuristics/README.md` for the
  exact format.

Every page under `history/` and `heuristics/` carries YAML frontmatter
(`type`, plus `destination`/`tags`/`style_tags` as applicable) — that's
the schema `INDEX.md` filtering depends on. Free-text content below the
frontmatter is unstructured prose, same as before.

Jarvis is the only agent that writes `memory/`: it creates new pages
under `history/` and updates `INDEX.md`'s History section, but never
touches `heuristics/` or `preferences.md`. Besides Jarvis, the only
other agent that reads it is the read-only `trip-planner-consolidator`
(see Self-growing loop below); Monday, Fiona, and Friday never see it
directly — Jarvis folds anything relevant into the `style_tags`/`budget`
fields it already passes downstream, so the associates keep working
from one consistent, filtered brief. Jarvis always confirms
memory-derived facts with the customer rather than applying them
silently (with the exception of heuristics, which are applied without
confirmation), and treats a missing or empty `memory/` as normal, not an
error.

## Self-growing loop

The workflow closes the loop from raw experience to better planning in
four steps, only the first of which existed before this addition:

1. **Logging** — Jarvis creates a page under `memory/history/` after
   every trip and adds a one-line entry to `memory/INDEX.md`.
2. **Evaluation** — that page's frontmatter includes a `revisions` field:
   how many rounds each associate needed and whether the 2-round cap was
   hit. This is Jarvis's own existing tracking, just persisted instead of
   discarded.
3. **Consolidation** — run `trip-planner-consolidator` manually (it is
   never invoked automatically or by Jarvis) after enough trips have
   accumulated. It reads `memory/INDEX.md` and every page under
   `memory/history/` and drafts candidate heuristic pages with the
   evidence behind each one. It has no `Write` tool and cannot persist
   anything — its output is a draft only.
4. **Policy update** — a human reviews that draft and hand-writes the
   approved heuristics as new pages under `memory/heuristics/`, adding an
   entry to `memory/INDEX.md` for each. Jarvis discovers these via the
   index on every subsequent trip and folds applicable ones directly into
   the brief, with no customer confirmation step (unlike
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
