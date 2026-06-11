# Guardrails

Apply every one of these before launch. They exist because the failure mode
of loop-building is not "no loop" — it is a loop that spams, costs money,
or silently dies.

## When NOT to loop

- **One-off task** → just do it now, in this session (iterating against
  its done-condition if it has one — but save no card).
- **Needs fresh human judgment every time** (e.g. "decide which candidate to
  hire") → a loop can gather and prepare, but say plainly that the decision
  part stays human.
- **Really a reusable on-demand action** ("format my release notes") → that
  wants to be a skill, not a loop. Offer to point them at skill-building.

Saying "you don't need a loop" builds more trust than shipping one.

## Silence condition is mandatory

Every recurring loop's Measurement must state when the loop says nothing.
Refuse to launch without it.

## Cheapest cadence that solves the problem

Default to the longest interval that still works: daily-granularity problems
get daily loops, not 5-minute loops. Anything more frequent than every 30
minutes needs a stated reason. Each run costs tokens and attention.

## Headless-auth check before any cron launch

Exercise each needed tool the way the cron job will run it before creating
the job. If a tool needs interactive auth, redesign, fall back to an
in-session `/loop`, or make the loop owner-triggered (see engines.md).
Never launch a loop that will silently fail at 9am.

## One owner per loop

A loop belongs to the person who built it; the card names them. Teammates
who want it get a copy of the card, not a shared job — shared jobs rot
because nobody owns the failure.
