# Trip History

One page per planned trip, named `<year>-<month>-<destination-short>.md`
(matching the corresponding folder under `trips/`, e.g.
`history/2026-08-japan.md`). Jarvis creates one here after compiling each
trip's final report, then adds a one-line entry to `memory/INDEX.md`
pointing to it.

Find trips via `memory/INDEX.md`, not by browsing this folder — the index
carries the `destination`/`tags`/`style_tags` needed to tell which pages
are relevant to a new trip without opening all of them.

## Page format

```
---
type: history
destination: <slug>
dates: <start> to <end>
style_tags: [<tag>, <tag>]
revisions: {monday: <n>, fiona: <n>, friday: <n>, cap_hit: <yes/no>}
---

<1-2 line takeaway: what worked, what to repeat or avoid>
```

The `revisions` field is Jarvis's own existing round-count tracking,
persisted instead of discarded — it's the primary evidence
`trip-planner-consolidator` looks for recurring patterns in.

Jarvis reads pages here (selected via `destination`/`style_tags` overlap
in `memory/INDEX.md`) before gathering requirements for a new trip, and
surfaces anything relevant back to the customer to confirm — a history
fact is never applied silently. `trip-planner-consolidator` reads all
pages in this folder (via the index) to look for cross-trip patterns.

No trip pages exist yet — this folder fills in as trips complete.
