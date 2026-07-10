# Context: coding-situation-awareness

A skill that interrupts an AI-assisted coding session to probe whether the user
still holds a working mental model of the code the AI has been writing.
Grounded in situation awareness (SA) research from human-automation teaming.

## Glossary

### Situation Awareness (SA)
Endsley's three-level model, adapted to AI-assisted coding:
- **Level 1 — Perception.** What elements/changes exist in the current state of
  the code. ("What did the agent just change?")
- **Level 2 — Comprehension.** What those elements mean together — how they
  fit the system, what invariants they touch, what role each plays.
- **Level 3 — Projection.** What will happen next — downstream effects, what
  the next agent turn is likely to do, what breaks if X is called.

### Out-of-the-loop decrement (OOTL)
The failure mode this skill exists to catch. Over a long AI-driven session
the user drifts into passive monitoring and progressively loses the mental
model needed to intervene, review, or steer. Invisible to the user by
definition: you don't know what you've stopped tracking.

### The Operator
The human user. Analogous to a pilot or process-control operator supervising
an automated system.

### The Automation
The AI coding agent. Analogous to an autopilot or supervisory controller.

### Probe
A single targeted question the skill asks the operator. Each probe is tagged
with the SA level it tests (L1 / L2 / L3).

### Cold Reader stance
Before writing any probe, the agent must discard prior session context and
re-derive its understanding of the code from artefacts a fresh observer
would see: current file contents, `git diff`, `git log`, and any plan
documents in the repo. Probes must be answerable (and gradeable) from those
artefacts alone. This mirrors the situation the operator would face if the
automation vanished — which is what an OOTL check is testing for.

### Session Diff
The perimeter of code a probe session covers. Defined as the diff between
the current working tree and the last human-authored commit (fallback: ask
the operator to name a base ref). This is the *trunk* of what probes may
ask about — the code the operator has been letting the automation write.

### Neighbourhood
The files immediately adjacent to files in the Session Diff: importers,
imported modules, and tests that exercise them. L2 (comprehension) and L3
(projection) probes may reach into the Neighbourhood, because comprehension
and projection are meaningless without surrounding context. Probes must not
require information outside `Session Diff ∪ Neighbourhood`, so every probe
remains gradeable from a bounded set of files.

### Probe Shape
Every probe is:
1. **Closed / specific.** Answerable in one short sentence or a pick-from-list.
   Open recall ("explain the auth flow") is banned — it is the exact format
   that lets OOTL hide behind fluent confabulation.
2. **Gradeable.** Tied to a specific location in `Session Diff ∪ Neighbourhood`.
3. **SA-tagged.** Labelled L1 (perception), L2 (comprehension), or L3
   (projection).
4. **Pre-answered.** The agent must write the ground-truth answer at the same
   time it writes the probe. If it can't state a crisp answer, the probe is
   discarded. This gate prevents the skill from degrading into vibes.

### Data-Flow Probes
A required probe category when the Session Diff touches code that reads,
transforms, filters, joins, aggregates, or otherwise moves data. In
data/analysis code the "system state" is not the call graph — it is the
**shape, meaning, and provenance of the data at each step**. OOTL in this
setting is silent: no crash, just wrong numbers. Data-Flow Probes target:

- **Row/record identity.** "After this groupby, what does one row represent?"
- **Filters and drops.** "Which rows are silently excluded by this step, and
  on what condition?"
- **Assumptions.** "What must be true of the input for this transform to be
  correct? Where is that checked, if anywhere?"
- **Units and types.** "What is the unit of column X after this step? Did it
  change?"
- **Nulls and missingness.** "How are missing values handled here — dropped,
  imputed, propagated?"
- **Joins and cardinality.** "Is this join one-to-one, one-to-many, or
  many-to-many? What happens to unmatched rows?"
- **Ordering dependence.** "Does the result depend on input order? Is order
  preserved?"

These probes are L2 by default (comprehension of what the pipeline *means*)
but can be L3 when they ask about downstream consequences ("if a null slips
through here, where does it first cause a wrong answer?").

### Control-Flow Probes
The other probe pool. Targets logic ownership, call graph, invariants, error
handling, and public API shape. Examples: "Which function now owns retry
logic for X?", "If `cancel()` is called twice, what happens on the second
call?", "Which of these files no longer imports `logger`?"

### Probe Pool Applicability
For each hunk in the Session Diff the skill decides which pool(s) apply:

- Touches a DataFrame / array / SQL / query builder / tabular I/O →
  **Data-Flow pool** applies.
- Touches control flow, state, side effects, or public API surface →
  **Control-Flow pool** applies.
- Most non-trivial hunks trigger both.

**Coverage rule.** If any hunk in the Session Diff triggers the Data-Flow
pool, the probe session must include at least one Data-Flow Probe before it
ends. This enforces the skill's emphasis on awareness of data shape,
filters, and assumptions without hard-coding a mode.

### Repair Step
After each probe, if the operator's answer is wrong, hedged, or absent, the
skill briefly states the actual answer from the code, citing `file:line`.
This turns the probe into a live re-boarding of the operator's mental model
— SA is *restored*, not just measured. Endsley's own work stresses that
SA-restoring interventions matter more than SA measurement.

### SA-Gap Summary
The skill's output is not a score. It is (a) a repair transcript and (b) a
short SA-Gap Summary at the end that names, in one line each, the areas
where the operator's model diverged from the code. Severity is signalled by
plain language, not numbers ("small slip on retry ownership" vs "you did
not know which rows this filter drops — that's a real gap").

### No Hard Gating
The skill never blocks the session. The operator may continue at any time.
When the SA-Gap Summary reveals a serious gap, the skill *points at it*
plainly and stops — it does not lecture, plead, or require acknowledgement.
Respecting the operator's autonomy is what makes the skill something they
will actually invoke.

### Conciseness Constraint
The skill's failure mode as a *tool* is being boring. Rules:
- Probes: one sentence, no preamble.
- Repair answers: at most two short sentences plus a `file:line` citation.
- SA-Gap Summary: one line per gap.
- No teacherly framing ("Great question!", "Let's explore..."). No restating
  of the operator's answer before responding.
- Prefer showing a code fragment over describing it in prose.
If a probe or repair can't fit these limits, it is the wrong probe —
rewrite it or drop it.

### Session Log
Appended to `.coding-sa/log.md` in the target repo (created lazily). One
entry per probe session, containing: date, Session Diff scope, probes
asked, pass/fail per probe, and the SA-Gap Summary. Kept terse. The log
exists so the operator can spot longitudinal patterns ("I keep failing L2
data-flow probes on joins") without the skill having to nag in-session.

### Probe Budget
Default: **3 probes per session**, covering:
- at least one L1 (perception),
- at least one L2 (comprehension),
- at least one L3 (projection) *or* one Data-Flow Probe when the Coverage
  Rule is triggered.

Deep pass: **6 probes**, invoked only when the operator explicitly asks
(e.g. "deep SA check"). Broader coverage across hunks; still bound by the
Conciseness Constraint on each probe.

### Drift-Risk Ranking
Heuristic used to pick which hunks to probe when there are more candidates
than the budget allows. Rank hunks by, roughly:
- size of hunk,
- cross-file reach (imports/importers touched),
- data-flow density (DataFrame / SQL / array ops present),
- time since the operator last edited that region *manually* (longer = more
  likely to be out-of-the-loop).

Probe the top-ranked hunks first. The ranking is a heuristic, not a
formula — the agent uses judgement, but must be able to justify why a
probed hunk beat an unprobed one if asked.

### Delivery Protocol
Probes are delivered **one at a time**. The operator answers from memory,
then the skill runs the Repair Step, then the next probe. This mirrors
SAGAT methodology: the answer reflects the operator's *current* mental
model, not what they can re-derive by re-reading the code on the spot.

### Answer Grades
Three grades only:
- **Hit.** Correct and confident.
- **Partial.** Either hedged ("I'd need to check") or partially correct.
  `"I'd need to check"` is an explicitly valid answer and is graded here —
  not as a failure. Making not-knowing safe is what keeps the operator
  honest; otherwise they bluff and the skill is worthless.
- **Miss.** Wrong-confident. This is the grade that signals real OOTL.

Only Misses and repeated Partials feed into the SA-Gap Summary as "real
gaps."

### Invocation Model
The skill is **user-invoked**. The operator asks for an SA check when they
want one. The skill never auto-fires and never blocks.

Supplementing this, a short guidance file in this repo (`NUDGE.md`)
describes when the *coding agent* should suggest an SA check to the
operator: long session, large accumulated diff, many AI-authored turns
without human pushback, data-flow-heavy hunks. Suggestions are one line,
dismissible, and issued at most once per triggering condition. The nudge
is guidance for the surrounding coding agent, not behaviour of the skill
itself — but it lives here because it is meaningless without the skill it
points to.

### Return-From-Away Mode
A variant invocation: the operator opens a repo they haven't touched in a
while and asks for an SA check on it. Same skill, but Session Diff is
redefined as "changes since the operator was last active here" (fallback:
ask them for a base ref, or use last commit authored by them). Everything
else — Cold Reader stance, Probe Shape, Budget, Delivery Protocol — is
unchanged.
