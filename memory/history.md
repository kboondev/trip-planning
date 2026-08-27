# Trip History

A running log of past trips planned through the Jarvis workflow. Jarvis
appends one entry here after compiling each trip's final report — a short
factual record, an outcome signal (how many revision rounds each
associate needed), plus any lesson worth carrying into future planning
(what worked, what didn't, anything that should adjust future briefs).

Jarvis reads this file before gathering requirements for a new trip,
alongside `preferences.md`, and surfaces anything relevant back to the
customer to confirm — it is never applied silently.

Newest entries at the top. Format:

```
## <year>-<month> — <destination>

- Dates: <start> to <end>
- Style: <style_tags used>
- Revisions: Monday <n>, Fiona <n>, Friday <n> (cap hit: yes/no)
- Takeaway: <1-2 lines — what worked, what to repeat or avoid>
```

No trips have been planned yet — this file will fill in as trips complete.
