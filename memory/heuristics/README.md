# Planning Heuristics

One page per distilled heuristic, named for the rule (e.g.
`heuristics/long-haul-transit-buffer.md`). Promoted here by a human after
reviewing output from the `trip-planner-consolidator` agent — never
written by Jarvis or the consolidator directly (neither has a path to
persist here; the consolidator has no Write tool at all).

## Page format

```
---
type: heuristic
tags: [<tag>, <tag>]
based_on: [<history-page-slug>, <history-page-slug>]
confidence: <low/medium/high>
---

<rule, phrased as guidance Jarvis can fold directly into a brief, e.g.
"Add a buffer day after any leg combining long-haul transit (>6h) with a
hiking/activity day.">
```

`based_on` names the `history/` pages (by filename, without `.md`) that
support this rule — keeps the evidence trail intact so a human reviewing
a refinement later can trace it back.

Unlike `preferences.md` and pages under `history/`, these are process
lessons distilled across trips, not customer-specific facts — Jarvis
applies them without confirming with the customer first. Jarvis finds
relevant heuristics via `tags` in `memory/INDEX.md` (matched against the
current trip's `style_tags`/destination), then reads those pages
directly, and folds applicable ones into the brief and delegation.

No heuristics exist yet — this folder fills in as
`trip-planner-consolidator` output gets reviewed and promoted by hand.
