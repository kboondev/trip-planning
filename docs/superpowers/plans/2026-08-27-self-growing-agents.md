# Self-Growing Agents Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close the gap between "memory-augmented" and "self-improving" in the trip-planning workflow by adding the three missing pieces of the self-growing loop — evaluation, consolidation, and policy update — without changing Monday/Fiona/Friday's contracts or the customer-facing workflow.

**Architecture:** Jarvis's existing revision-round tracking becomes a persisted outcome signal in `memory/history.md` (evaluation). A new, Read-only subagent `trip-planner-consolidator` reads that history and drafts candidate heuristics with evidence, for a human to review (consolidation). Approved heuristics live in a new `memory/heuristics.md` that Jarvis reads and folds into every brief without customer confirmation, since they're process lessons rather than customer facts (policy update).

**Tech Stack:** None — this repo has no executable code, build system, or test suite. Agents are markdown prompt specs under `.claude/agents/tripPlanWorkflow/`; "testing" means reading files back and grepping for required strings/structure, per the repo's own CLAUDE.md.

**Spec:** `docs/superpowers/specs/2026-08-27-self-growing-agents-design.md`

## Global Constraints

- No automatic, unsupervised policy changes — every promoted heuristic passes through a human glance before it can affect live behavior (spec Non-goals).
- No customer-facing satisfaction surveys or new interview steps — the evaluation signal comes only from revision-round counts Jarvis already tracks (spec §1).
- Monday, Fiona, and Friday's contracts, tools, and fallback behavior do not change, and they remain unaware `memory/` exists at all (spec Non-goals, existing CLAUDE.md "Only Jarvis reads or writes `memory/`").
- The consolidator agent gets `tools: Read` only — no `Write`, by construction, so it cannot persist anything itself (spec §2).
- Heuristics are applied silently by Jarvis (no customer confirmation), in explicit contrast to preferences/history facts which still require confirmation (spec §3).

---

### Task 1: Evaluation — persist the revision-count signal in history.md

**Files:**
- Modify: `memory/history.md`
- Modify: `.claude/agents/tripPlanWorkflow/trip-planner-manager.md`

**Interfaces:**
- Produces: the `history.md` entry template now includes a `Revisions:` line in the form `Monday <n>, Fiona <n>, Friday <n> (cap hit: yes/no)`. Task 3's consolidator agent reads this exact field name and format as its evidence source — keep the literal string `Revisions:` so it's grep-able.

- [ ] **Step 1: Update the history.md template to include the revision signal**

Use the Edit tool on `memory/history.md`:

old_string:
```
A running log of past trips planned through the Jarvis workflow. Jarvis
appends one entry here after compiling each trip's final report — a short
factual record plus any lesson worth carrying into future planning
(what worked, what didn't, anything that should adjust future briefs).
```

new_string:
```
A running log of past trips planned through the Jarvis workflow. Jarvis
appends one entry here after compiling each trip's final report — a short
factual record, an outcome signal (how many revision rounds each
associate needed), plus any lesson worth carrying into future planning
(what worked, what didn't, anything that should adjust future briefs).
```

- [ ] **Step 2: Add the Revisions line to the template block**

Use the Edit tool on `memory/history.md`:

old_string:
```
## <year>-<month> — <destination>

- Dates: <start> to <end>
- Style: <style_tags used>
- Takeaway: <1-2 lines — what worked, what to repeat or avoid>
```

new_string:
```
## <year>-<month> — <destination>

- Dates: <start> to <end>
- Style: <style_tags used>
- Revisions: Monday <n>, Fiona <n>, Friday <n> (cap hit: yes/no)
- Takeaway: <1-2 lines — what worked, what to repeat or avoid>
```

- [ ] **Step 3: Update Jarvis's SOUL section to log the revision signal**

Use the Edit tool on `.claude/agents/tripPlanWorkflow/trip-planner-manager.md`:

old_string:
```
After you compile the final report, append a short entry to `memory/history.md` (destination, dates, a one-to-two-line takeaway) so future trips can draw on it.
```

new_string:
```
After you compile the final report, append a short entry to `memory/history.md` (destination, dates, the revision-round count you already tracked for each associate and whether any hit the 2-round cap, and a one-to-two-line takeaway) so future trips can draw on it.
```

- [ ] **Step 4: Verify the edits landed and stay consistent**

Run: `grep -n "Revisions:" "memory/history.md" "docs/superpowers/plans/2026-08-27-self-growing-agents.md"`
Expected: a match in `memory/history.md`'s template block (this plan file will also match itself — ignore that hit).

Run: `grep -n "revision-round count" ".claude/agents/tripPlanWorkflow/trip-planner-manager.md"`
Expected: one match, in the SOUL paragraph.

Read `.claude/agents/tripPlanWorkflow/trip-planner-manager.md` in full and confirm the new SOUL sentence doesn't contradict the existing FALLBACK bullet "If a revision cap (2 rounds) is hit... stop looping, escalate..." — both should agree the cap is 2 rounds per associate.

- [ ] **Step 5: Commit**

```bash
git add memory/history.md .claude/agents/tripPlanWorkflow/trip-planner-manager.md
git commit -m "Log revision-round counts as an outcome signal in history.md

Jarvis already tracks revision rounds per associate under its 2-round
cap; persisting that count turns the existing history log into an
evaluation signal instead of a flat record.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>"
```

---

### Task 2: Policy update — memory/heuristics.md + Jarvis reads and applies it silently

**Files:**
- Create: `memory/heuristics.md`
- Modify: `.claude/agents/tripPlanWorkflow/trip-planner-manager.md`

**Interfaces:**
- Consumes: none from Task 1.
- Produces: `memory/heuristics.md` exists with a documented bullet format (`- <rule> (based on N trips)`) that Task 3's consolidator reads as the "existing heuristics" input, and that a human writes into after reviewing the consolidator's draft.

- [ ] **Step 1: Create memory/heuristics.md**

Use the Write tool to create `memory/heuristics.md`:

```
# Planning Heuristics

Distilled patterns from past trips, promoted here by a human after
reviewing output from the `trip-planner-consolidator` agent. Jarvis
reads this file before gathering requirements for a new trip and again
when drafting the delegation brief, and folds applicable heuristics
into the brief directly.

Unlike `preferences.md` and `history.md`, these are process lessons
distilled across trips, not customer-specific facts — Jarvis does not
confirm them with the customer before applying them.

Format:

- <rule>, e.g. "Add a buffer day after any leg combining long-haul
  transit (>6h) with a hiking/activity day." (based on N trips)

No heuristics yet — this file fills in as `trip-planner-consolidator`
output gets reviewed and promoted by hand.
```

- [ ] **Step 2: Update Jarvis's SOUL section to read and apply heuristics.md**

Use the Edit tool on `.claude/agents/tripPlanWorkflow/trip-planner-manager.md`:

old_string:
```
**Memory is centralized through you.** Before interviewing the customer, check `memory/preferences.md` and `memory/history.md` for standing preferences and past trips. You are the only agent that reads these — Monday, Fiona, and Friday never see them directly; you fold anything relevant into `style_tags` or `budget` when you delegate, so all three associates work from the same filtered brief. Never apply a remembered preference silently: surface it back to the customer as a confirmation ("I have you down as preferring boutique hotels and a relaxed pace — still the case for this trip?") before treating it as settled. If `memory/` is missing or empty, proceed with a normal fresh interview — this is not an error.
```

new_string:
```
**Memory is centralized through you.** Before interviewing the customer, check `memory/preferences.md`, `memory/history.md`, and `memory/heuristics.md` for standing preferences, past trips, and distilled planning heuristics. You are the only agent that reads these — Monday, Fiona, and Friday never see them directly; you fold anything relevant into `style_tags` or `budget` when you delegate, so all three associates work from the same filtered brief. Never apply a remembered preference or history fact silently: surface it back to the customer as a confirmation ("I have you down as preferring boutique hotels and a relaxed pace — still the case for this trip?") before treating it as settled. Heuristics from `memory/heuristics.md` are the one exception — they are process lessons distilled from past trips, not customer-specific facts, so fold applicable ones directly into the brief without a confirmation step. If `memory/` is missing or empty, proceed with a normal fresh interview — this is not an error.
```

- [ ] **Step 3: Update Jarvis's CONTRACT section to match**

Use the Edit tool on `.claude/agents/tripPlanWorkflow/trip-planner-manager.md`:

old_string:
```
Before gathering requirements, read `memory/preferences.md` and
`memory/history.md` if they exist. Treat their contents as candidates to
confirm with the customer, not settled facts — fold anything confirmed
into the fields below rather than tracking it separately.
```

new_string:
```
Before gathering requirements, read `memory/preferences.md`,
`memory/history.md`, and `memory/heuristics.md` if they exist. Treat
preferences and history contents as candidates to confirm with the
customer, not settled facts — fold anything confirmed into the fields
below rather than tracking it separately. Treat heuristics differently:
fold applicable ones directly into the brief and delegation without a
customer confirmation step, since they are process lessons rather than
customer-specific facts.
```

- [ ] **Step 4: Update Jarvis's FALLBACK section to match**

Use the Edit tool on `.claude/agents/tripPlanWorkflow/trip-planner-manager.md`:

old_string:
```
- **If `memory/preferences.md` or `memory/history.md` is missing, empty, or unreadable:** proceed with a normal fresh requirements interview — don't block or mention it as a problem to the customer.
```

new_string:
```
- **If `memory/preferences.md`, `memory/history.md`, or `memory/heuristics.md` is missing, empty, or unreadable:** proceed with a normal fresh requirements interview — don't block or mention it as a problem to the customer.
```

- [ ] **Step 5: Verify the edits**

Run: `grep -n "heuristics.md" ".claude/agents/tripPlanWorkflow/trip-planner-manager.md"`
Expected: at least 4 matches (SOUL, CONTRACT, FALLBACK, and any others you added).

Read `memory/heuristics.md` back and confirm it explicitly states heuristics are applied without customer confirmation — this is the one place in `memory/` where that's true, so it must be unambiguous to avoid future edits accidentally merging it with the preferences/history confirmation rule.

- [ ] **Step 6: Commit**

```bash
git add memory/heuristics.md .claude/agents/tripPlanWorkflow/trip-planner-manager.md
git commit -m "Add memory/heuristics.md as a policy-update tier Jarvis applies silently

Heuristics are process lessons distilled from past trips (not
customer-specific facts), so Jarvis folds them into every brief
directly instead of confirming them with the customer first, unlike
preferences.md/history.md.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>"
```

---

### Task 3: Consolidation — new trip-planner-consolidator agent

**Files:**
- Create: `.claude/agents/tripPlanWorkflow/trip-planner-consolidator.md`

**Interfaces:**
- Consumes: `memory/history.md`'s `Revisions:` field (Task 1) and `memory/heuristics.md`'s bullet format (Task 2) as its two read inputs.
- Produces: no persisted output (Read-only agent) — its return value is a draft report for the invoking user to read and act on manually. No other task depends on this agent's output programmatically.

- [ ] **Step 1: Create the consolidator agent spec**

Use the Write tool to create `.claude/agents/tripPlanWorkflow/trip-planner-consolidator.md`:

```
---
name: trip-planner-consolidator
description: Reviews memory/history.md for recurring patterns across past trips and drafts candidate planning heuristics, with supporting evidence, for a human to review before they're promoted into memory/heuristics.md.
tools: Read
model: sonnet
---

# SOUL

You are the Trip Planning Consolidator. You value evidence over instinct — you never propose a heuristic from a single data point, and every candidate you draft names the trips that support it. You read `memory/history.md` (and `memory/heuristics.md` if it already exists, so you can propose amendments rather than duplicates) and look for patterns that recurred across multiple trips: a style/location combination that consistently needed extra revision rounds, a takeaway that echoes across entries, a recurring gap between plan and outcome. You do not talk to the customer, you do not touch any trip folder under `trips/`, and you never write to `memory/heuristics.md` yourself — your entire output is a draft for a human to review, edit, and promote by hand. You are invoked manually, on demand, never automatically and never by Jarvis as part of a trip.

# TONE & STYLE

- Speak plainly and lead with the strongest-evidence pattern first.
- Never use emojis.
- Present each candidate heuristic as a bulleted rule plus the evidence behind it — never a bare assertion.
- Say explicitly when a pattern is too thin to act on, rather than rounding it up to a confident rule.

# CONTRACT

You receive: no input from another agent — you read `memory/history.md` and `memory/heuristics.md` (if present) directly.

You must return, for each candidate heuristic:
- the proposed rule, phrased as guidance Jarvis could fold directly into a brief (e.g. "add a buffer day after any leg combining long-haul transit (>6h) with a hiking/activity day")
- the evidence: which history entries (by destination/date) support it, and how many
- a confidence note when the sample is thin (e.g. "based on 2 trips — weak signal, consider deferring")

If `memory/heuristics.md` already has entries, note whether each candidate is new, a refinement of an existing one, or contradicts one (and say so plainly rather than silently overwriting).

# GUARDRAILS

- Never propose a heuristic from a single trip — 2 is the practical floor, and even then, label it low-confidence.
- Never write to `memory/heuristics.md`, `memory/preferences.md`, or any file under `trips/` — you have no Write tool and no path to persist anything; your output is a draft only.
- Never invent a pattern `history.md` doesn't actually support — if the log is too thin or too uniform to say anything useful, say that plainly instead of manufacturing a rule.
- Don't restate `history.md` entries verbatim as your output — synthesize across entries; a list of unconnected facts is not a heuristic.

# FALLBACK

- **If `memory/history.md` is missing, empty, or has fewer than 2 entries:** say so plainly and stop — there isn't enough data to consolidate anything yet, and this is not an error.
- **If `memory/heuristics.md` doesn't exist yet:** treat every candidate as new; there's nothing to reconcile against.
- **If two entries suggest contradictory lessons for a similar situation:** surface both, note the contradiction, and let the human reviewer decide rather than picking one silently.
```

- [ ] **Step 2: Verify the agent file's tool grant and structure**

Run: `grep -n "^tools:" ".claude/agents/tripPlanWorkflow/trip-planner-consolidator.md"`
Expected: `tools: Read` exactly — no `Write`, no `Agent`, no `Web`.

Run: `grep -n "^# " ".claude/agents/tripPlanWorkflow/trip-planner-consolidator.md"`
Expected: five matches, in order: `SOUL`, `TONE & STYLE`, `CONTRACT`, `GUARDRAILS`, `FALLBACK` — matching the structure of the other three agent files.

- [ ] **Step 3: Commit**

```bash
git add .claude/agents/tripPlanWorkflow/trip-planner-consolidator.md
git commit -m "Add trip-planner-consolidator: read-only agent that drafts heuristics

Read-only by construction (no Write tool) so it can never persist a
policy change itself — a human must review its draft and promote
entries into memory/heuristics.md by hand.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>"
```

---

### Task 4: Documentation — CLAUDE.md

**Files:**
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: the final file/agent names and behaviors from Tasks 1–3 (this task documents what now exists; it must be done last).

- [ ] **Step 1: Add trip-planner-consolidator to the Architecture agent list**

Use the Edit tool on `CLAUDE.md`:

old_string:
```
- `financial-associate` (Friday) — produces the cost breakdown and budget
  comparison (as an Excel file) from Monday's itinerary. Dispatched in
  parallel with Fiona.
```

new_string:
```
- `financial-associate` (Friday) — produces the cost breakdown and budget
  comparison (as an Excel file) from Monday's itinerary. Dispatched in
  parallel with Fiona.
- `trip-planner-consolidator` — a Read-only agent, invoked manually (never
  by Jarvis, never automatically) after a batch of trips. Reads
  `memory/history.md` and drafts candidate planning heuristics with
  supporting evidence for a human to review; it cannot write
  `memory/heuristics.md` itself.
```

- [ ] **Step 2: Add a "Self-growing loop" subsection after Customer memory**

Use the Edit tool on `CLAUDE.md`:

old_string:
```
## Trip storage
```

new_string:
```
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
```

- [ ] **Step 3: Add memory/heuristics.md to the Customer memory section**

Use the Edit tool on `CLAUDE.md`:

old_string:
```
- `memory/history.md` — a running log of past trips, appended to by
  Jarvis after each final report is compiled.
```

new_string:
```
- `memory/history.md` — a running log of past trips, appended to by
  Jarvis after each final report is compiled, including a `Revisions:`
  outcome signal per associate.
- `memory/heuristics.md` — planning heuristics distilled from
  `history.md` by the `trip-planner-consolidator` agent and promoted by
  a human. Unlike the other two files, Jarvis applies these without
  customer confirmation, since they're process lessons rather than
  customer-specific facts.
```

- [ ] **Step 4: Verify CLAUDE.md matches the files on disk**

Run: `grep -n "trip-planner-consolidator\|heuristics.md\|Self-growing loop" "CLAUDE.md"`
Expected: matches in the Architecture agent list, the new Self-growing loop section, and the Customer memory section.

Read the full updated `CLAUDE.md` and confirm:
- the agent list order and delegation subagent_type table are unchanged except for the new consolidator bullet (it should NOT be added to the `subagent_type` delegation table, since Jarvis never dispatches it)
- the four numbered steps in "Self-growing loop" name the exact files/agents that exist on disk after Tasks 1–3

- [ ] **Step 5: Commit**

```bash
git add CLAUDE.md
git commit -m "Document the self-growing loop and trip-planner-consolidator in CLAUDE.md

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>"
```
