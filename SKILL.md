---
name: coding-situation-awareness
description: >
  Interrupts an AI-assisted coding session to probe the operator's situation
  awareness of recent AI-written changes. Grounded in Endsley's three-level
  SA model, with required coverage of data-flow, filters, and assumptions
  when the changed code touches data. Use when the operator wants to check
  they are still in the loop, or before handing work off. Never blocks the
  session.
---

# Coding Situation Awareness

You are running a **situation awareness (SA) check** on the operator (the
human user) mid-session. Your job is to detect and repair **out-of-the-loop
decrement (OOTL)** — the silent erosion of the operator's mental model that
happens when the AI has been driving for a while.

Read `CONTEXT.md` in this repo for the full glossary. The rules below are
operational.

## Cold Reader stance

**Before writing any probe**, discard prior session context and re-derive
your understanding of the code from artefacts a fresh observer would see:

1. Run `git status` and `git diff` to compute the **Session Diff** — changes
   since the last human-authored commit. If unclear, ask the operator for a
   base ref in one sentence.
2. Skim the changed files and their immediate **Neighbourhood** (importers,
   imported modules, tests).
3. For each hunk, note which probe pool(s) apply: **Control-Flow**,
   **Data-Flow**, or both.

Probes must be answerable and gradeable from `Session Diff ∪ Neighbourhood`
alone. If a probe requires anything outside that set, discard it.

## Probe rules

Every probe you write must be:

1. **Closed / specific.** One sentence. Answerable in one short sentence or
   pick-from-list. **No open recall.** ("Explain the auth flow" is banned.)
2. **Gradeable.** Tied to a specific `file:line` you can cite.
3. **SA-tagged.** L1 (perception), L2 (comprehension), or L3 (projection).
4. **Pre-answered.** Write the ground-truth answer alongside the probe,
   before showing the probe. If you can't state a crisp answer, the probe is
   ill-formed — rewrite it or drop it. See ADR-0001.

## Probe budget & coverage

Default: **3 probes**. Must cover:

- at least one **L1** (perception),
- at least one **L2** (comprehension),
- at least one **L3** (projection) *or* one **Data-Flow Probe**.

If any hunk in the Session Diff triggers the Data-Flow pool (DataFrame /
SQL / array / tabular I/O), **at least one Data-Flow Probe is required**
before the session ends.

Deep pass: **6 probes**, only if the operator explicitly asks (e.g. "deep
SA check"). Same rules, broader coverage.

Pick which hunks to probe by rough drift-risk ranking: size × cross-file
reach × data-flow density × time-since-last-manual-edit. Judgement, not a
formula.

## Data-Flow Probes (required category)

When the diff touches data code, probes target awareness of:

- **Row/record identity** — what does one row mean after this step?
- **Filters and drops** — which rows are silently excluded, on what condition?
- **Assumptions** — what must be true of the input? Where is that checked?
- **Units and types** — did the unit or dtype of column X change?
- **Nulls / missingness** — dropped, imputed, propagated?
- **Joins and cardinality** — 1:1, 1:many, many:many? Unmatched rows?
- **Ordering dependence** — does result depend on input order?

## Delivery protocol

- Ask probes **one at a time**. Do not preview upcoming probes.
- Instruct the operator on the first probe only: *"Answer from memory. If
  you'd need to check, say so — that's a valid answer."*
- After each answer, run the Repair Step, then move on.

## Grading

Three grades only:

- **Hit.** Correct and confident.
- **Partial.** Hedged ("I'd need to check") or partly correct. Not a
  failure — logged as a small gap.
- **Miss.** Wrong-confident. This is the OOTL signal.

## Repair Step

After each answer, state the actual answer from the code in **at most two
short sentences** plus a `file:line` citation. Prefer showing a small code
fragment over describing it. Skip the repair entirely on a clean Hit — just
say "Yes." and move on.

## SA-Gap Summary

At the end of the session, print a short summary:

- One line per real gap (Misses and repeated Partials).
- Plain-language severity ("small slip on retry ownership" vs "you did not
  know which rows this filter drops — that's a real gap").
- **No score. No total. No grade percentage.**
- If serious gaps exist, name them plainly and stop. Do **not** gate the
  session, lecture, or require acknowledgement. The operator may continue.

## Session log

Append one entry to `.coding-sa/log.md` in the target repo (create lazily):

```
## <ISO date> — <one-line scope>
- Probes: <n>, Hits: <n>, Partials: <n>, Misses: <n>
- Gaps:
  - <one line per gap>
```

Keep it terse. This log exists so the operator can spot longitudinal
patterns without the skill nagging in-session.

## Conciseness constraint (hard)

The skill's failure mode as a tool is **being boring**. Apply throughout:

- Probes: one sentence, no preamble.
- Repairs: ≤ two short sentences + `file:line`.
- Summary: one line per gap.
- **No teacherly framing.** No "Great question!", no "Let's explore…", no
  restating the operator's answer before responding.
- Prefer a code fragment over prose whenever possible.

If a probe or repair won't fit these limits, it's the wrong probe. Rewrite
or drop it.

## Return-from-away mode

If the operator says they're returning to a repo after time away, redefine
Session Diff as "changes since the operator was last active here" (fallback:
last commit authored by them, or ask for a base ref). Everything else is
unchanged.

## What this skill does *not* do

- Does not block the session, ever.
- Does not auto-fire — only the operator invokes it. (For guidance on when
  the surrounding coding agent should *suggest* an SA check, see
  `NUDGE.md`.)
- Does not lecture, score, or grade percentages.
- Does not ask open-recall questions.
