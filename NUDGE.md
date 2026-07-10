# NUDGE.md — When the coding agent should suggest an SA check

This file is guidance for the **surrounding coding agent** (the one writing
code with the operator), not for the `coding-situation-awareness` skill
itself. The skill is user-invoked and never auto-fires. But because
out-of-the-loop decrement (OOTL) is invisible to the operator by
definition, they may not know they should run a check. A one-line nudge
from the coding agent, at the right moment, is what closes that loop.

## When to nudge

Nudge the operator to consider running an SA check when **any** of the
following holds, and you haven't already nudged them for the same trigger
in the current session:

- **Long AI-authored run.** You (the coding agent) have made ≥ ~5
  substantive edits in a row without the operator manually editing code or
  pushing back on your direction.
- **Big accumulated diff.** The session's uncommitted diff has crossed
  roughly ~300 changed lines or ~5 changed files.
- **Data-flow-heavy work.** You just wrote or modified code involving
  DataFrames, SQL, joins, groupbys, filters, or tabular I/O, and the
  operator did not question any of the assumptions you made about the data
  (types, keys, nulls, cardinality).
- **Long silence during acceptance.** The operator has been rapidly
  accepting your diffs (rubber-stamp cadence) for a stretch.
- **Return from away.** The operator opens a repo they haven't been in for
  a while and starts making changes on top.

## How to nudge

- **One line.** No preamble, no justification paragraph.
- **Dismissible.** Never blocking. If the operator ignores it, drop it.
- **Once per trigger.** Do not renudge for the same condition in the same
  session.
- **Neutral, not scolding.** The nudge is a co-pilot check-in, not a
  reprimand.

Good examples:

> Heads up — big diff piling up. Want to run an SA check before we go on?

> This groupby changes what one row means. Might be worth an SA check.

> You've been away from this repo for a while; a return-from-away SA check
> might help before we push further.

Bad examples (do not do these):

> ⚠️ You have not reviewed the code carefully. It is important that you
> understand the changes before proceeding. Consider running the
> `coding-situation-awareness` skill to verify your understanding...

> Are you sure you know what this code does? Perhaps you should check.

## What the nudge is not

- Not a gate. The coding agent must not refuse to continue.
- Not a lecture. If the operator says "no, keep going," keep going without
  comment.
- Not a substitute for the skill. The nudge only points; the skill does
  the actual work.
