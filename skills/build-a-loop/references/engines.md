# Engines: how a loop actually runs

Pick the engine with this table. The person should never need to understand
this — you apply it and explain the result in plain words.

| Situation | Engine |
|---|---|
| Recurring; must run with nobody at the keyboard | Cron scheduled task |
| Recurring; only useful while the person is actively working | `/loop` with an interval, in-session |
| One goal with a checkable done-condition | Self-paced iteration in the current session |

## Cron scheduled task

- Create with the cron/scheduled-task capability (e.g. the CronCreate tool).
- Job prompt is always: `Read ~/.claude/loops/<name>.md and execute one
  iteration per its instructions.` The card stays the single source of truth.
- Runs headless. **Before launching**, exercise every tool the iteration
  needs the way the job will run it. Interactive-auth MCP servers (Google
  Drive is the classic case) frequently work in a live session but fail
  headless. If a needed tool fails headless, redesign the loop or fall back
  to an in-session `/loop`.
- Test send-capable tools (Slack, email) headless with the least intrusive
  call that proves access — an identity/permission check, or a test message
  to the owner only. Never test-message third parties.
- If headless access can't be fixed and the cadence is too sparse for an
  in-session `/loop` (e.g. weekly), save the card anyway and make the loop
  owner-triggered: the owner says "run one iteration of <name>" when it's
  time. A manual loop beats one that silently fails.
- Pause/stop: list jobs with CronList, delete with CronDelete. In the card's
  Controls line, write this as words the owner can say to Claude (e.g.
  "tell Claude: delete the <name> cron job"), not tool names.

## /loop with an interval (in-session)

- The person runs: `/loop <interval> Read ~/.claude/loops/<name>.md and
  execute one iteration per its instructions.` (e.g. `/loop 30m …`).
- Lives only while the Claude Code session is open; it is not background
  automation. Good for "babysit this while I work today."
- Stop: interrupt the loop or close the session. Put that in Controls.

## Self-paced iteration (in-session)

- For iterative loops, don't schedule anything — keep working the problem in
  the current session: iterate, check the Measurement, repeat until it
  passes, then report.
- Each iteration must end by honestly evaluating the Measurement. Do not
  claim done because the output "looks good" — check the written condition.
- If the loop will outlive one sitting, save progress notes to the card's
  State file so a future session can resume.
- Card or no card: per the diagnosis tiebreaker in SKILL.md, save a card
  only if the same kind of problem will come back; a one-off goal gets
  worked to done with no card.
