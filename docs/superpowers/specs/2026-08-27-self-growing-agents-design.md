# Self-Growing Agents: Evaluation, Consolidation, Policy Update

Date: 2026-08-27

## Problem

The trip-planning workflow (Jarvis/Monday/Fiona/Friday) currently has one
of the four pieces a self-improving system needs: experience logging
(`memory/history.md`). It has no evaluation (no signal on whether a plan
was actually good), no consolidation (no distillation of recurring
patterns across trips), and no policy update (the agents don't plan any
better on trip 51 than trip 1). This spec adds the missing three pieces
while keeping the existing four-agent architecture and its contracts
intact.

## Non-goals

- No automatic, unsupervised policy changes. Every promoted heuristic
  passes through a human glance before it can affect live behavior.
- No customer-facing satisfaction surveys or star ratings. The signal
  comes from data Jarvis already has (revision counts), not a new
  interview step.
- No change to Monday/Fiona/Friday's contracts, tools, or fallback
  behavior — they remain unaware of `memory/` entirely, per the existing
  design.

## 1. Evaluation: revision counts as the outcome signal

Jarvis's existing CONTRACT already caps revisions at 2 rounds per
associate and already knows, at compile time, how many rounds each of
Monday/Fiona/Friday went through. This is free signal that today is
discarded once the report is compiled.

**Change:** extend the `history.md` entry format (in
`memory/history.md` and the template it documents) to include a
`Revisions:` line recording the round count per associate and whether
any hit the 2-round cap:

```
## <year>-<month> — <destination>

- Dates: <start> to <end>
- Style: <style_tags used>
- Revisions: Monday <n>, Fiona <n>, Friday <n> (cap hit: yes/no)
- Takeaway: <1-2 lines — what worked, what to repeat or avoid>
```

Jarvis's own CONTRACT section is updated to require this line when it
appends a history entry. No new tool calls, no new agent dispatch — this
is purely "log what you already tracked."

## 2. Consolidation: `trip-planner-consolidator` (new, read-only agent)

A fourth associate-tier agent, added at
`.claude/agents/tripPlanWorkflow/trip-planner-consolidator.md`, structured
with the same SOUL/TONE/CONTRACT/GUARDRAILS/FALLBACK sections as the
existing three associates for consistency.

- **Tools:** `Read` only. Deliberately no `Write` — this agent cannot
  persist anything, by construction, so the human-review gate can't be
  skipped by accident or a future prompt change.
- **Input:** `memory/history.md` (now carrying revision signals) and the
  existing `memory/heuristics.md` if one exists, so it can propose
  amendments rather than always starting fresh.
- **Output (its whole job):** a draft list of candidate heuristics, each
  with:
  - the proposed rule, phrased as guidance Jarvis could fold into a brief
  - the evidence: which trips (by history entry) support it, and how many
  - a confidence note if the sample is thin (e.g. "based on 2 trips —
    weak signal, consider deferring")
  - It explicitly does not invent a rule from a single data point.
- **Trigger:** manual only. Invoked the same way Monday/Fiona/Friday are
  invoked — an explicit `Agent` tool call with
  `subagent_type: "trip-planner-consolidator"` — whenever the user
  decides to run it (e.g. after a batch of trips). Jarvis never invokes
  it itself; it is not part of the per-trip workflow.
- **After the draft comes back:** the user reviews it, edits/drops/keeps
  proposed heuristics, and only then are the approved ones written into
  `memory/heuristics.md`. That write is a normal file edit at that point,
  not an autonomous agent action — this is the human gate the earlier
  analysis called for.

## 3. Policy update: `memory/heuristics.md` (new memory tier)

A new file, third alongside `preferences.md` and `history.md`:

```
# Planning Heuristics

Distilled patterns from past trips, promoted here by a human after
reviewing output from the trip-planner-consolidator agent. Jarvis reads
this file before gathering requirements and when drafting delegation
briefs, and folds applicable heuristics into the brief directly —
unlike preferences.md/history.md, these are process lessons, not
customer-specific facts, so they don't need a customer confirmation
step.

Format:

- <rule>, e.g. "Add a buffer day after any leg combining long-haul
  transit (>6h) with a hiking/activity day." (based on N trips)

No heuristics yet — this file fills in as `trip-planner-consolidator`
output gets reviewed and promoted.
```

**Change to Jarvis (`trip-planner-manager.md`):**
- SOUL: mention `memory/heuristics.md` alongside the other two memory
  files as something read before interviewing the customer.
- CONTRACT: reading order becomes preferences → history → heuristics;
  applicable heuristics get folded directly into the brief (locations/
  dates/style_tags/budget/travelers) sent to Monday/Fiona/Friday, with
  no customer confirmation required (contrast explicitly drawn against
  the existing preference-confirmation rule, so future edits don't
  accidentally merge the two behaviors).
- FALLBACK: missing/empty `heuristics.md` is normal, same treatment as
  the other two memory files today.

Monday/Fiona/Friday remain unaware `heuristics.md` exists — Jarvis
still folds everything into the same filtered brief fields it already
passes downstream, per the existing "memory is centralized through
Jarvis" design.

## 4. Documentation updates

- **CLAUDE.md:** add a short "Self-growing loop" subsection under
  Architecture or Customer memory naming the four pieces explicitly
  (logging → evaluation → consolidation → policy update), pointing at
  which file/agent implements each, and stating the human-review gate
  between consolidation and policy update. Add
  `trip-planner-consolidator` to the agent list with its role and its
  Read-only tool grant. Document `memory/heuristics.md` next to the
  existing `preferences.md`/`history.md` bullets.
- **`trips/README.md`:** no change — heuristics are a cross-trip memory
  concern, not a per-trip artifact.

## Data flow (updated)

```
Customer → Jarvis: interview (now also reads heuristics.md)
Jarvis → Monday/Fiona/Friday: brief (heuristics folded in silently)
Jarvis → memory/history.md: trip record + revision-count signal   [evaluation]
                                        │
                         (manual, later, batched)
                                        ▼
                     trip-planner-consolidator (Read-only)
                                        │
                         draft heuristics + evidence
                                        ▼
                         human reviews, edits, approves
                                        ▼
                         memory/heuristics.md updated             [consolidation
                                                                    + policy update]
                                        │
                         (next trip) Jarvis reads heuristics.md
```

## Testing / verification

This repo has no executable code or test suite (per CLAUDE.md) — agents
are prompts, verified by review and by running the workflow. Verification
for this change:
- Read back all edited/new agent files for internal consistency (no
  contradiction between SOUL/CONTRACT/GUARDRAILS).
- Confirm `trip-planner-consolidator` has no `Write` tool listed.
- Confirm CLAUDE.md's agent list and file conventions match the actual
  files on disk after the change.
- Optionally, dry-run: plan one trip through the full workflow, manually
  add a plausible history entry with revisions, invoke the consolidator,
  and confirm its draft output cites the entry as evidence and does not
  write any file itself.
