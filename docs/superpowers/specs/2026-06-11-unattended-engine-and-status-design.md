# Unattended engine (launchd) + loop status overview — design

Date: 2026-06-11
Status: approved
Owner: Raphael Syed

## Problem

Two gaps found by real use of the plugin:

1. `engines.md` claims "Recurring; must run with nobody at the keyboard →
   Cron scheduled task", but Claude Code's in-session scheduler cannot
   deliver that: jobs are session-scoped (die with the session, `durable`
   not honored), recurring jobs auto-expire after 7 days, and jobs fire
   only while a session is open and idle. Loops that should "kick off on
   their own" silently don't.
2. Once several loops exist there is no overview: nothing lists every
   loop, whether it is scheduled, when it last ran, and whether it worked.

## Goals

- Recurring unattended loops genuinely run with no session open, on the
  loop owner's machine, authenticated with their normal Claude
  subscription login (no API keys on disk).
- A loop that fails or is missed becomes visible, never silent.
- One command answers "what loops do I have and are they healthy?"

Target user: terminal users of Claude Code with Anthropic platform /
claude.ai subscription auth, on macOS. (Linux gets a short equivalence
note; Windows, cloud Routines, and Desktop Scheduled Tasks are out of
scope — Routines/Desktop are documented as alternatives only.)

## Design

### 1. Conventions — one name, five artifacts

For a loop named `<name>` (kebab-case):

| Artifact | Path |
|---|---|
| Loop card (source of truth) | `~/.claude/loops/<name>.md` |
| launchd job | `~/Library/LaunchAgents/com.claude-loops.<name>.plist` |
| Logs | `~/.claude/loops/logs/<name>.log` and `<name>.err` |
| State | `~/.claude/loops/state/<name>.md` |
| Outputs | `~/.claude/loops/output/` (loop-specific filenames) |

The plist runs (single ProgramArguments invocation, shown logically):

```
bash -lc 'claude -p "Read ~/.claude/loops/<name>.md and execute one
  iteration per its instructions." \
  --permission-mode dontAsk \
  --allowedTools "<derived from the card Needs line>" \
  --max-turns <N, default 30> \
  || { echo "FAILED $(date '+%Y-%m-%d %H:%M')" >> ~/.claude/loops/logs/<name>.err;
  osascript -e "display notification \"Loop <name> failed — say:
  show my loops\" with title \"Claude loops\""; }'
```

Key choices:
- **No `--bare`**: bare mode skips OAuth/Keychain reads; subscription
  auth requires a normal (non-bare) headless run.
- **`dontAsk` + per-loop allowlist**: least privilege; an unauthorized
  tool errors loudly (non-zero exit → `.err` → status flags it) instead
  of hanging.
- **`osascript` wrapper**: covers the failure class where `claude`
  itself cannot run (auth, binary path) and the card's own
  "notify owner on failure" step therefore never executes.
- **`StartCalendarInterval`** for the schedule; `PATH` env in the plist
  includes the directory containing `claude` (e.g. `~/.local/bin`).
  launchd runs a missed calendar interval once on wake from sleep;
  powered-off fire times are skipped (caught by the status skill).
- Cadences launchd cannot express (every other week) keep the existing
  state-file gate pattern (e.g. "exit silently if last full run < 13
  days ago"), with the job firing weekly.

### 2. engines.md rewrite — the engine ladder

| Situation | Engine |
|---|---|
| Recurring; must run on its own, nobody present | **launchd LaunchAgent** (macOS) running `claude -p` headless; Linux: user crontab equivalent, one paragraph |
| Recurring; only matters while actively working | `/loop <interval>` in-session |
| Same-day one-shot ("at 4pm today") | In-session cron (CronCreate) — stated plainly: session-scoped, recurring jobs expire after 7 days, fires only while a session is open and idle |
| Scheduling impossible (auth or platform) | Owner-triggered ("run one iteration of <name>") |

Also documented: Routines (cloud; laptop can be off; **no local file
access**, so card/state/local-binary loops cannot use it) and Desktop
Scheduled Tasks (needs the Desktop app) as alternatives with their
constraints; guidance that editing a card's `Needs:` line requires asking
Claude to refresh the plist allowlist.

### 3. build-a-loop changes (steps 6 and 8)

- Step 6 (pre-launch): unchanged tool-by-tool headless exercise, plus:
  Claude (as the builder, at launch time) derives the `--allowedTools`
  list by enumerating every command and tool the card's Iteration steps
  invoke — it is written out concretely in the plist, never inferred at
  run time. A tool the iteration uses but the list misses will error
  under `dontAsk`, so the step-6 headless exercise doubles as the
  allowlist completeness check.
- Step 8 (launch), for unattended recurring loops, with explicit consent
  (it writes system state):
  1. Write the plist; `launchctl bootstrap gui/$UID <plist>`.
  2. **Plumbing probe**: install a temporary job (same env, label
     `com.claude-loops.probe`) whose command is
     `claude -p "reply OK" --max-turns 1`, `launchctl kickstart` it,
     verify "OK" lands in its log, then remove it. This proves
     Keychain auth, binary path, and log writing under real launchd
     conditions. Result (pass/fail, date, claude version) is recorded in
     `~/.claude/loops/state/.probe.md`; the probe is skipped when that
     file shows a pass for the current claude binary, so it runs once
     per machine, not per loop.
  3. Tell the owner in plain words: what was installed, where logs live,
     how to pause/stop (phrased as words to say to Claude; mechanically
     `launchctl bootout gui/$UID/com.claude-loops.<name>`).
- Card `Engine:` line format for these loops:
  `launchd: <schedule in plain words> · label com.claude-loops.<name>`.
- Controls line: pause/stop/resume phrasing as today, plus "ask Claude to
  show my loops" as the health check.

### 4. New skill: `loop-status`

Location: `skills/loop-status/SKILL.md` in the plugin (invokes as
`/loops:loop-status`; trigger phrases "show my loops", "loop status",
"are my loops healthy", "did <loop> run").

Behavior — read-only gather, then one line per loop:

`name · engine+schedule · last run · last result · next due · health`

Sources: cards (parse Engine/State/Controls), `launchctl list` filtered
to `com.claude-loops.*`, plists (schedule), state files (last run), log
tails (errors, exit), output dir (latest artifact).

Health classification (deterministic, in this order):
1. **failing** — `.err` shows errors for the most recent fire, or the
   osascript failure path triggered.
2. **overdue** — next-due time (from plist schedule, respecting any
   state-file gate) has passed by more than a grace period (default 2h)
   with no state/log evidence of a run. This is the silent-failure
   catch.
3. **paused** — plist exists but the job is not loaded.
4. **not scheduled** — card exists with no plist/job: owner-triggered
   (fine, say so) or orphaned (flag).
5. **in-session** — card's engine is `/loop` or in-session cron; report
   honestly that liveness can't be verified from outside the session.
6. **OK** — none of the above; show last-run evidence.

Also flags plists labeled `com.claude-loops.*` with no matching card
(orphans). The skill may RECOMMEND fixes (kickstart now, re-bootstrap,
refresh allowlist) but acts only when the user says so.

### 5. guardrails.md additions

- "Headless-auth check before any cron launch" generalizes to: before
  trusting any unattended schedule, the plumbing probe must have passed
  on this machine, and every tool must be exercised headless.
- New guardrail: every unattended loop must be visible to `loop-status`
  (five-artifact convention) — a loop the overview can't see is a loop
  that can fail silently.

## Validation plan (spine of the implementation plan)

1. Probe `claude -p "reply OK"` headless from a plain shell, then from a
   real launchd job on Raphael's machine. **Open question this resolves:
   does Keychain OAuth work under launchd?** If not: fallback is
   `apiKeyHelper` reading the macOS Keychain, and the spec's auth section
   gets revised before anything ships.
2. Migrate `weekly-update-digest` from the session-only one-shot to a
   launchd job (delete session job; biweekly state gate keeps the
   anchor). Its next scheduled morning run is the real-world validation.
3. Run `loop-status` against the real layout; then simulate one failure
   (e.g. rename `fetch-updates`) and confirm it reports **failing**, and
   simulate a missed fire and confirm **overdue**.
4. Re-run the plugin's scenario tests against the rewritten engines.md
   (existing practice for this repo).

## Out of scope

Linux mechanics beyond a one-paragraph note; Windows; Routines or
Desktop Scheduled Tasks integration; retrofitting iterative loops (no
schedule to manage); automatic key management.
