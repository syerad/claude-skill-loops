# Writing the iteration prompt

The iteration prompt is what runs every cycle with no human watching. It
lives in the loop card's Iteration section. Compile it from these patterns.
Where an existing skill already does a step, the card can simply say
"invoke the <name> skill" rather than re-specifying its logic.

## 1. Concrete checks, concrete places

Bad:  "Check for stale PRs."
Good: "List open PRs in the acme org repos. A PR is stale if it has no
review activity for 24h on weekdays."

Name the system, the query, and the threshold. The loop has no conversation
context to fall back on — everything it needs must be in the card.

## 2. State between runs (recurring loops)

Any loop that nudges people or reports changes needs memory, or it will
re-nudge and re-report every run. Give it a state file:

- Card lists `State: ~/.claude/loops/state/<name>.md`.
- Iteration starts with "read the state file" and ends with "update the
  state file with what you acted on" (e.g. PR #123 nudged 2026-06-11).
- The measurement enforces it: "every stale PR gets exactly ONE nudge."

## 3. Silence condition (recurring loops)

The measurement must say when the loop produces nothing: "If no PRs are
stale, output nothing and send no messages." A loop that always says
something trains its owner to ignore it.

## 4. Output: format and destination

Say exactly where output goes (Slack DM to the owner, a thread in a channel,
a file) and what shape it takes (three bullets max; link each item). When
the output goes to other people, write the message template into the
iteration steps so tone stays consistent run after run.

## 5. Verification separate from generation (iterative loops)

End every cycle with an explicit check of the Measurement, as written —
ideally by re-running the concrete evaluation (the test suite, the
checklist, the metric), not by judging the work "good enough". The loop
terminates when the check passes, not when the output looks plausible.

## 6. Bounded actions

The prompt states what the loop may do autonomously and what it must only
report. Nudging a teammate once: fine to automate. Closing someone's PR,
emailing externally, deleting anything: report and recommend instead. When
in doubt during compilation, ask the owner which side of the line an action
falls on.

## 7. Binary or metric (iterative loops)

A termination condition comes in two shapes. Binary: a condition that is
true or false — tests green, checklist passes, everyone submitted. Metric:
a number to push — bundle size, p95 latency, failure rate. Metric loops
need four things written into the card:

- Baseline: measure before the first change.
- Measurement command: the exact command or query that produces the number.
- Keep rule: keep a change only if the number improves; otherwise revert.
- Stop rule: a target ("under 200KB"), a plateau ("2 cycles without
  improvement"), or a budget ("10 cycles max") — never "until it's good".

Fuzzy goals ("sounds professional", "reads well") are neither — turn them
into a binary checklist and have a fresh reviewer score it (see 5).
