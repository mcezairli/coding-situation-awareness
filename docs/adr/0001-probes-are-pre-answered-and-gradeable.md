# ADR 0001 — Probes are pre-answered and gradeable from a bounded file set

Date: 2026-07-10
Status: Accepted

## Context

The skill exists to detect **out-of-the-loop decrement (OOTL)**: the silent
erosion of the operator's mental model during long AI-driven coding
sessions. To do that, the skill asks the operator questions ("probes")
about the code the AI has recently written.

There is a real design fork in what a probe looks like:

- **Open-recall probes.** "Explain what changed in the auth flow." Easy to
  write, feel educational, and feel more like real code review.
- **Closed, specific probes.** "Which function now owns retry logic for
  `fetchUser`?" Feel narrower and mechanical.

Open recall is the default temptation because it looks richer. Situation
awareness research rejects it for measurement (Endsley, 1995a), because
open recall lets the operator produce fluent, coherent-sounding answers
that mask actual gaps in their model. SAGAT (the Situation Awareness Global
Assessment Technique; Endsley, 1995b) responds by using closed, targeted
probes against a known ground truth — which is the shape we adopt here.
That fluent-but-empty answer is the exact failure mode we are trying to
catch.

## Decision

Every probe the skill emits must satisfy all of the following:

1. **Closed / specific.** Answerable in one short sentence or pick-from-list.
2. **Gradeable against a location in `Session Diff ∪ Neighbourhood`.** The
   answer must be checkable against a specific `file:line`.
3. **Pre-answered.** The agent writes the ground-truth answer at the same
   time it writes the probe, before showing it to the operator. If the
   agent cannot state a crisp ground-truth answer, the probe is ill-formed
   and must be discarded — not asked and figured out on the fly.

Open-recall probes are banned.

## Alternatives considered

- **Open recall.** Rejected because it defeats the primary purpose (SA
  measurement) — the exact operators most drifted out of the loop are the
  ones best able to bluff a fluent explanation. Rejecting it is the whole
  reason SAGAT exists.
- **Closed probes without pre-answers.** Rejected because it lets the agent
  ask questions it hasn't checked, which degrades to vibes: the operator
  answers, the agent grades on the fly and may itself be wrong, and the
  Repair Step becomes unreliable. The pre-answer gate forces probe quality
  up-front.
- **Allow the operator to look at the code before answering.** Rejected
  because that measures "can you find the answer" — which is code review —
  not "do you have the answer in your working model," which is what OOTL is
  about. Instead we made `"I'd need to check"` an explicitly valid answer
  (graded Partial), so the operator can be honest without bluffing.

## Consequences

**Makes easier:**
- The Repair Step is reliable — there is always a citable ground truth.
- The Session Log is meaningful — Hits, Partials, and Misses map to a real
  construct (SA gap), not to grader mood.
- Rejecting probes is cheap: if the agent can't pre-answer, it drops the
  probe and picks another hunk. This raises the floor on probe quality.

**Makes harder:**
- Writing good probes takes more work than writing open questions. That is
  intentional — the skill trades authoring effort for measurement validity.
- The probe budget is small partly because each probe is expensive to
  author well.

**Reversal cost:**
- Loosening this rule later would silently degrade every future probe
  session with no visible failure — the skill would keep running and
  producing output, but the output would stop meaning what it claims to
  mean. That is why this is written down: future contributors who look at
  the skill and think *"why not open questions? they'd be richer"* need to
  see the reason before changing it.

## References

- Endsley, M. R. (1995a). Toward a theory of situation awareness in dynamic
  systems. *Human Factors*, 37(1), 32–64.
  <https://doi.org/10.1518/001872095779049543>
- Endsley, M. R. (1995b). Measurement of situation awareness in dynamic
  systems. *Human Factors*, 37(1), 65–84.
  <https://doi.org/10.1518/001872095779049499>
