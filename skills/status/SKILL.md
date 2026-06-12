---
name: status
description: >
  Show every loop built with the build skill and whether it is healthy.
  Reads loop cards, launchd jobs, state files, and logs, then reports
  schedule, last run, last result, next due, and a health verdict per
  loop. Trigger on "show my loops", "loop status", "are my loops
  healthy", "did <loop> run", "why didn't <loop> run", or any request
  for an overview of scheduled loops. Read-only: it recommends fixes
  but never applies one unless the person asks.
---

# Loop Status

Report the health of every loop. Gather first, classify second, report
third. Never guess: every verdict cites the artifact it came from.

## 1. Gather

Run these (substitute the real home directory):

- Cards: `ls ~/.claude/loops/*.md` — read each; parse the Engine, State,
  and Controls lines, and any state-file gate mentioned in Iteration
  (e.g. "exit silently if last full run < 13 days ago").
- Jobs: `launchctl list | grep com.claude-loops` — running/loaded labels.
- Plists: `ls ~/Library/LaunchAgents/com.claude-loops.*.plist` — read
  each for its StartCalendarInterval schedule.
- State: read `~/.claude/loops/state/<name>.md` for each card.
- Logs: `tail -20 ~/.claude/loops/logs/<name>.log` and `<name>.err` —
  look for `FAILED <timestamp>` markers, permission errors, auth errors.
- Output: most recent artifact per loop in `~/.claude/loops/output/`.

## 2. Classify (first match wins)

1. **failing** — `.err` has a FAILED marker or errors for the most
   recent fire time.
2. **overdue** — the most recent scheduled fire time that the state-file
   gate would NOT have silenced is more than 2 hours past, and neither
   state nor log shows a run. (This is the silent-failure catch: machine
   off, job unloaded by an OS update, allowlist drift.)
3. **paused** — plist exists but its label is not in `launchctl list`.
4. **not scheduled** — card exists, no plist, and Engine is not
   in-session: fine if Engine says owner-triggered or self-paced (say
   so); flag as orphaned otherwise.
5. **in-session** — Engine is `/loop` or an in-session scheduled task:
   report honestly that liveness cannot be verified from outside that
   session.
6. **OK** — none of the above; cite last-run evidence (state date, log
   tail, newest output file).

Also flag reverse orphans: any `com.claude-loops.*` plist with no
matching card.

## 3. Report

One line per loop, then flags, then recommendations:

    weekly-update-digest · launchd, Fridays 9:03 (biweekly via state
    gate) · last run 2026-06-12 (report written, notification sent) ·
    next due 2026-06-26 · OK

Recommendations only — do not act without the person asking. When they
ask, the fixes are (see engines.md): run now = `launchctl kickstart
gui/$(id -u)/com.claude-loops.<name>`; resume = `launchctl bootstrap
gui/$(id -u) <plist>`; pause = `launchctl bootout …`; allowlist drift =
regenerate the plist's --allowedTools from the card and re-bootstrap.

If there are no loops at all, say so and point at `/loops:build`.
