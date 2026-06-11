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

## Headless check before any unattended launch

Two proofs before trusting any schedule: (1) the machine's plumbing probe
has passed for the current claude version (see engines.md — Keychain
auth, binary path, logs under real launchd conditions), and (2) every
tool the iteration needs has been exercised headless, the way the job
will run it, and is covered by the plist's allowlist. If either fails,
redesign, fall back to an in-session `/loop`, or make the loop
owner-triggered (see engines.md). Never launch a loop that will silently
fail at 9am.

## One owner per loop

A loop belongs to the person who built it; the card names them. Teammates
who want it get a copy of the card, not a shared job — shared jobs rot
because nobody owns the failure.

## Every unattended loop must be visible to loop-status

An unattended loop must follow the five-artifact convention (card, plist
label `com.claude-loops.<name>`, logs, state, output — see engines.md) so
the loop-status skill can see it. A loop the overview cannot see is a
loop that can fail silently. Tell every owner the health check: say
"show my loops".
