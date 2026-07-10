# coding-situation-awareness

A skill for AI coding agents that interrupts a session to check whether the
human operator still holds a working mental model of the code the AI has
been writing.

Grounded in Endsley's three-level situation awareness (SA) model[^endsley1995]
from human-automation teaming research, applied to AI-assisted coding.

## Why

In a long AI-driven coding session the operator drifts into passive
monitoring and progressively loses the mental model needed to intervene,
review, or steer. The literature calls this **out-of-the-loop decrement
(OOTL)**. It is invisible to the operator by definition: you don't know
what you've stopped tracking.

This skill exists to detect and repair OOTL, on demand, without blocking
the session.

## What it does

When the operator invokes it, the agent:

1. Takes a **Cold Reader** stance — discards prior session context and
   re-derives its understanding of the code from `git diff` alone.
2. Computes the **Session Diff** (changes since the last human-authored
   commit) and its immediate **Neighbourhood** (importers, imported
   modules, tests).
3. Asks **3 probes** (or 6 on a "deep" pass), one at a time, covering:
   - at least one **L1** (perception),
   - at least one **L2** (comprehension),
   - at least one **L3** (projection) *or* a **Data-Flow Probe**.
4. After each answer, states the ground-truth answer with a `file:line`
   citation (the **Repair Step**).
5. Prints a terse **SA-Gap Summary** at the end. No score, no grade
   percentage.
6. Appends one entry to `.coding-sa/log.md` in the target repo.

Every probe is:

- **Closed / specific** — one sentence, answerable in one short sentence
  or pick-from-list. No open recall.
- **Gradeable** — tied to a specific `file:line` in the Session Diff or
  its Neighbourhood.
- **Pre-answered** — the agent writes the ground-truth answer at the same
  time it writes the probe. If it can't, the probe is discarded (see
  [ADR-0001](docs/adr/0001-probes-are-pre-answered-and-gradeable.md)).

When the diff touches data code (DataFrames, SQL, joins, groupbys, tabular
I/O) at least one **Data-Flow Probe** is required — because in data code
the failure mode of OOTL is not a crash, it is wrong numbers.

## What it deliberately does *not* do

- Does not block the session, ever.
- Does not auto-fire — only the operator invokes it.
- Does not lecture, score, or grade percentages.
- Does not ask open-recall ("explain the auth flow") questions — that
  format is exactly where OOTL hides behind fluent confabulation.

## Files

- [`SKILL.md`](SKILL.md) — the operational rules the coding agent follows
  when the skill is invoked.
- [`CONTEXT.md`](CONTEXT.md) — full glossary and rationale for every
  design choice.
- [`NUDGE.md`](NUDGE.md) — guidance for the *surrounding* coding agent on
  when to *suggest* an SA check to the operator (the skill itself never
  auto-fires).
- [`docs/adr/`](docs/adr/) — architectural decision records.

## Installing

This skill is intended to live in a coding agent's skills directory. For
[pi](https://github.com/earendil-works/pi-coding-agent):

```
git clone https://github.com/mcezairli/coding-situation-awareness \
  ~/.pi/agent/skills/coding-situation-awareness
```

Other agents that support skill-style extensions should be able to consume
`SKILL.md` directly.

## Invoking

Ask the coding agent for an SA check. Examples:

> Run an SA check on what we just did.

> Deep SA check before I push.

> I'm back after a week — return-from-away SA check on this repo.

## Status

First pass. Not yet extensively tested in-the-wild.

## License

MIT — see [LICENSE](LICENSE).

## References

[^endsley1995]: Endsley, M. R. (1995). Toward a theory of situation awareness
    in dynamic systems. *Human Factors*, 37(1), 32–64.
    <https://doi.org/10.1518/001872095779049543>
