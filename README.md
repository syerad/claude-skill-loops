# claude-skill-loops

A Claude Code plugin with two skills. `build-a-loop` turns a recurring chore
or a grind-until-it-passes goal into a loop that Claude runs on a schedule or
in your session. `loop-status` tells you whether your loops actually ran and
what went wrong if they didn't.

The core idea: a loop is a markdown file, called a loop card, stored in
`~/.claude/loops/`. The card states the problem, what a successful run looks
like, and the exact steps of one iteration. Every scheduler runs the same
prompt: read the card, execute one iteration. Edit the card and you have
changed the loop. Give the file to a teammate and they have their own copy.

## Quick start

Inside Claude Code:

```
/plugin marketplace add syerad/claude-skill-loops
/plugin install loops@claude-skill-loops
```

Then describe the thing that keeps eating your time:

```
/loops:build-a-loop every Friday I chase my team for status updates
```

Claude asks a few questions (how often, where the data lives, what useful
output looks like), proposes a design, runs one iteration live so you can
judge the real output before anything is scheduled, and then launches it.
Check on your loops any time:

```
/loops:loop-status
```

or just say "show my loops".

## What build-a-loop builds

**Recurring loops.** Digests, watchers, reminders, report chasers. Anything
that repeats on a clock. Every recurring loop must define a silence
condition: when there is nothing worth saying, it says nothing. A loop that
posts every run trains you to ignore it.

**Iterative loops.** One goal with a checkable done-condition: polish this
doc until the checklist passes, hunt flaky tests until the remaining
failures are deterministic, shrink the bundle below 200KB. Claude iterates,
checks the condition, and stops when it holds (or when a stop rule like
"two cycles without improvement" fires).

**Sometimes, nothing.** If the task happens once, or needs fresh human
judgment every time, the skill says so and offers to just do it now instead.
It will not build a loop nobody needs.

The skill ships a recipe gallery (`skills/build-a-loop/references/recipes.md`)
with proven shapes: PR review SLA watcher, support queue digest, competitor
changelog watcher, metrics anomaly digest, campaign link health check, and
more. Browse it any time with `/loops:recipes` (or ask "what loops could I
build?"). Recipes are starting points, not templates it forces on you.

Before any launch, the design is tested live: one full iteration runs
read-only, with outward messages shown as drafts instead of sent. Both times
this plugin was used on real problems, that test caught a bug that would
have shipped.

## How loops run

| You need | Engine |
|---|---|
| Runs by itself, nobody at the keyboard | launchd job (macOS) running `claude -p` headless |
| Runs while you work, stops when you close the session | `/loop` with an interval |
| A one-shot later today | In-session scheduled task |
| Iterate until done | Claude works it in the current session |
| Scheduling impossible (auth, platform) | You say "run one iteration of X" when it's time |

The launchd engine is the one that makes loops genuinely unattended. Each
loop gets its own LaunchAgent with a least-privilege tool allowlist derived
from its card, log files, and a failure notification if Claude itself fails
to start. Your normal Claude subscription login works headless (verified;
no API keys end up on disk). Before the first launchd loop on a machine,
the skill runs a one-time probe to prove auth and logging work under
launchd, so nothing is scheduled on assumptions.

Honest limits: unattended scheduling is implemented for macOS. The machine
can be asleep at fire time (the job runs on wake) but not powered off; a
fire time missed while powered off shows up as overdue in loop-status
instead of disappearing. Linux works with the same command in a user
crontab, described briefly in the engine docs but not yet field-tested.
Cloud-hosted scheduling (Anthropic Routines) is deliberately not used: it
has no access to local files, and loops are built on local cards and state.

## Knowing your loops are alive

`loop-status` reads every card and cross-checks it against the scheduler,
state files, and logs, then gives each loop one verdict:

- **failing** - the log shows errors for the most recent run
- **overdue** - a scheduled fire time passed and no evidence of a run
  exists. This is the catch for silent failures: machine was off, an OS
  update unloaded the job, a permission rule drifted
- **paused** - installed but unloaded
- **not scheduled** - card exists with no job; fine for on-demand loops,
  flagged if it looks orphaned
- **in-session** - lives in a session this skill cannot see into, and it
  tells you so rather than guessing
- **OK** - with the evidence cited: state date, log tail, newest output

It recommends fixes (re-run now, re-enable, refresh permissions) but never
applies one unless you ask.

## Owning a loop

Each loop belongs to one person and is controlled in plain words, written
on the card itself:

- pause: "pause the weekly-digest loop"
- resume: "resume the weekly-digest loop"
- run now: "run one iteration of weekly-digest"
- stop for good: "stop the weekly-digest loop" removes the scheduled job;
  delete the card too if you never want it back
- change behavior: edit the card; for schedule changes, also ask Claude to
  update the job
- share: send the card file; the recipient runs their own copy

Everything a loop produces lives under `~/.claude/loops/`: cards at the
top, run state in `state/`, logs in `logs/`, reports in `output/`.

## What's in the repo

- `skills/build-a-loop/SKILL.md` - the guided journey from problem to
  running loop
- `skills/build-a-loop/references/` - recipe gallery, engine mechanics
  (including the launchd plist template), prompt patterns for iteration
  steps, guardrails
- `skills/loop-status/SKILL.md` - the health overview
- `docs/superpowers/` - design specs and implementation plans for the
  plugin itself

## Updating

New versions are picked up with `/plugin marketplace update
claude-skill-loops`, or enable auto-update for this marketplace in the
`/plugin` UI. For development, run from a checkout instead:

```
git clone https://github.com/syerad/claude-skill-loops.git
claude --plugin-dir /path/to/claude-skill-loops
```

## Contributing a recipe

Built a loop your team copies? Add its card to
`skills/build-a-loop/references/recipes.md` in the existing format (Roles,
Type, Problem, Measurement, Iteration, Needs, Output) and open a PR.
