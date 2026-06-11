# Unattended Engine (launchd) + loop-status Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make recurring loops run with no Claude session open (per-loop macOS LaunchAgents running `claude -p` headless with subscription OAuth), and add a `loop-status` skill that reports every loop's schedule, last run, and health.

**Architecture:** Each unattended loop owns five artifacts derivable from its name (card, plist, log+err, state, output). `build-a-loop` installs and verifies the plist at launch; `loop-status` reconciles cards ↔ `launchctl` ↔ state ↔ logs and classifies health deterministically. Spec: `docs/superpowers/specs/2026-06-11-unattended-engine-and-status-design.md`.

**Tech Stack:** Markdown skill files (the "code" is LLM instructions), launchd plists, bash, `claude -p` headless (v2.1.173, `--permission-mode dontAsk`, per-loop `--allowedTools`).

**Conventions used throughout:** `$HOME` is `/Users/rapha` on the dogfood machine. launchd does **not** expand `~` in plist values — plist paths must be absolute. In XML, `&&` must be written `&amp;&amp;`. Allowlist rule syntax follows what this machine's settings already use: `Bash(<prefix>:*)`, `Read(//abs/path/**)`.

---

### Task 1: launchd plumbing probe (GATE — resolves the spec's open question)

Proves subscription OAuth from Keychain works when `claude -p` runs under launchd. **If this fails, STOP: revise the spec's auth section (apiKeyHelper fallback) before any other task.**

**Files:**
- Create: `~/.claude/loops/logs/` (directory)
- Create: `/tmp/com.claude-loops.probe.plist` (temporary)
- Create: `~/.claude/loops/state/.probe.md`
- No repo changes in this task; nothing to commit.

- [ ] **Step 1: Terminal headless pre-check**

Run:
```bash
mkdir -p ~/.claude/loops/logs && claude -p "reply with exactly: OK" --max-turns 1
```
Expected: prints `OK`. This proves headless + OAuth in a terminal context. If this already fails, fix before touching launchd (likely auth or PATH).

- [ ] **Step 2: Write the probe plist**

Write `/tmp/com.claude-loops.probe.plist`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.claude-loops.probe</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>-c</string>
    <string>claude -p "reply with exactly: OK" --max-turns 1</string>
  </array>
  <key>RunAtLoad</key><false/>
  <key>StandardOutPath</key><string>/Users/rapha/.claude/loops/logs/probe.log</string>
  <key>StandardErrorPath</key><string>/Users/rapha/.claude/loops/logs/probe.err</string>
  <key>EnvironmentVariables</key>
  <dict>
    <key>PATH</key><string>/Users/rapha/.local/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
  </dict>
  <key>WorkingDirectory</key><string>/Users/rapha</string>
</dict>
</plist>
```

- [ ] **Step 3: Bootstrap and kickstart the probe**

```bash
launchctl bootstrap gui/$(id -u) /tmp/com.claude-loops.probe.plist
launchctl kickstart gui/$(id -u)/com.claude-loops.probe
sleep 45 && cat ~/.claude/loops/logs/probe.log ~/.claude/loops/logs/probe.err
```
Expected: `probe.log` contains `OK`; `probe.err` empty. If `probe.err` shows an auth/login error → the Keychain question is answered NO → STOP per the gate above and report.

- [ ] **Step 4: Record the result and clean up**

Write `~/.claude/loops/state/.probe.md`:
```markdown
# launchd plumbing probe
result: pass
date: <today YYYY-MM-DD>
claude: <output of `claude --version`>
```
Then:
```bash
launchctl bootout gui/$(id -u)/com.claude-loops.probe
rm /tmp/com.claude-loops.probe.plist ~/.claude/loops/logs/probe.log ~/.claude/loops/logs/probe.err
```
Expected: `launchctl list | grep claude-loops` shows nothing.

---

### Task 2: Rewrite `engines.md`

**Files:**
- Modify: `skills/build-a-loop/references/engines.md` (full replacement)

- [ ] **Step 1: Replace the entire file content with:**

```markdown
# Engines: how a loop actually runs

Pick the engine with this table. The person should never need to understand
this — you apply it and explain the result in plain words.

| Situation | Engine |
|---|---|
| Recurring; must run on its own, nobody present | launchd LaunchAgent (macOS) running `claude -p` headless |
| Recurring; only useful while the person is actively working | `/loop` with an interval, in-session |
| Same-day one-shot ("at 4pm today, remind me") | In-session scheduled task (CronCreate) |
| One goal with a checkable done-condition | Self-paced iteration in the current session |
| Unattended needed but launchd impossible (auth, platform) | Owner-triggered manual loop |

## launchd LaunchAgent (macOS) — the unattended engine

This is real OS-level scheduling: it fires with no Claude session open.
The laptop must be awake or asleep (launchd runs a missed
StartCalendarInterval once on wake); fire times missed while powered off
are skipped — the loop-status skill catches those as "overdue".

Every unattended loop named `<name>` owns five artifacts:

| Artifact | Path |
|---|---|
| Loop card | `~/.claude/loops/<name>.md` |
| launchd job | `~/Library/LaunchAgents/com.claude-loops.<name>.plist` |
| Logs | `~/.claude/loops/logs/<name>.log` + `<name>.err` |
| State | `~/.claude/loops/state/<name>.md` |
| Output | `~/.claude/loops/output/` |

Plist template (replace NAME, USER, the schedule, TOOL_LIST; launchd does
NOT expand `~` — use absolute paths; escape `&` as `&amp;` in XML):

    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
      "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
    <plist version="1.0">
    <dict>
      <key>Label</key><string>com.claude-loops.NAME</string>
      <key>ProgramArguments</key>
      <array>
        <string>/bin/bash</string>
        <string>-c</string>
        <string>claude -p "Read ~/.claude/loops/NAME.md and execute one
    iteration per its instructions." --permission-mode dontAsk
    --allowedTools "TOOL_LIST" --max-turns 50 || { echo "FAILED $(date
    '+%Y-%m-%d %H:%M')" >> /Users/USER/.claude/loops/logs/NAME.err;
    osascript -e 'display notification "Loop NAME failed — say: show my
    loops" with title "Claude loops"'; }</string>
      </array>
      <key>StartCalendarInterval</key>
      <dict>
        <key>Weekday</key><integer>5</integer>
        <key>Hour</key><integer>9</integer>
        <key>Minute</key><integer>3</integer>
      </dict>
      <key>StandardOutPath</key>
      <string>/Users/USER/.claude/loops/logs/NAME.log</string>
      <key>StandardErrorPath</key>
      <string>/Users/USER/.claude/loops/logs/NAME.err</string>
      <key>EnvironmentVariables</key>
      <dict>
        <key>PATH</key>
        <string>/Users/USER/.local/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
      </dict>
      <key>WorkingDirectory</key><string>/Users/USER</string>
    </dict>
    </plist>

(The ProgramArguments command is ONE plist string; the line breaks above
are for readability — keep it on one line in the real file.)

Key mechanics:

- **No `--bare`.** Bare mode skips Keychain/OAuth; subscription login
  requires a normal headless run. PATH in the plist must include the
  directory containing the `claude` binary (`which claude`).
- **`dontAsk` + per-loop `--allowedTools`.** Least privilege. Enumerate
  every command and tool the card's Iteration steps invoke, as concrete
  rules (`Bash(<command prefix>:*)`, `Read(//abs/path/**)`,
  `Write(//abs/path/**)`, tool names like `PushNotification`). An
  unlisted tool makes the run error loudly (non-zero exit → `.err`) —
  never hang. If the owner later edits the card to need new tools, the
  allowlist must be refreshed: tell them to ask Claude to update it.
- **Failure wrapper.** The `|| { echo FAILED …; osascript …; }` tail
  covers the class where `claude` itself cannot run (auth broke, binary
  moved): the owner gets a macOS notification and `.err` gets a marker
  line loop-status can read. Failures *inside* the iteration are the
  card's job (its own "notify owner on failure" step).
- **Plumbing probe, once per machine.** Before trusting the first
  launchd loop: install a temporary job (label `com.claude-loops.probe`,
  same env) running `claude -p "reply with exactly: OK" --max-turns 1`,
  `launchctl kickstart` it, confirm OK lands in its log, record
  pass/fail + date + claude version in
  `~/.claude/loops/state/.probe.md`, remove the job. Skip when that file
  already shows a pass for the current claude version.
- **Sparse cadences.** launchd cannot say "every other week" — schedule
  the nearest expressible cadence (weekly) and gate in the card with the
  state file ("if last full run < 13 days ago, exit silently").
- Install / pause / resume / run-now (write Controls as words the owner
  says to Claude, not these commands):
  - install: `launchctl bootstrap gui/$(id -u) <plist>`
  - pause: `launchctl bootout gui/$(id -u)/com.claude-loops.<name>`
  - resume: bootstrap again
  - run now: `launchctl kickstart gui/$(id -u)/com.claude-loops.<name>`

Linux: same design, different scheduler — a user crontab line
(`crontab -e`) running the identical `claude -p … || …` command, logs
redirected to the same paths; `notify-send` instead of osascript.

## /loop with an interval (in-session)

- The person runs: `/loop <interval> Read ~/.claude/loops/<name>.md and
  execute one iteration per its instructions.` (e.g. `/loop 30m …`).
- Lives only while the Claude Code session is open; it is not background
  automation. Good for "babysit this while I work today."
- Stop: interrupt the loop or close the session. Put that in Controls.

## In-session scheduled task (CronCreate) — same-day one-shots only

Honest limits, all verified: jobs are session-scoped (they die with the
session — the durable flag is not honored), recurring jobs auto-expire
after 7 days, and a job fires only while a session is open and idle.
That makes this engine right for exactly one thing: a one-shot later
today, in a session that will stay open ("at 4pm, check the deploy").
Never use it as the engine for an unattended recurring loop.

## Self-paced iteration (in-session)

- For iterative loops, don't schedule anything — keep working the problem
  in the current session: iterate, check the Measurement, repeat until it
  passes, then report.
- Each iteration must end by honestly evaluating the Measurement. Do not
  claim done because the output "looks good" — check the written
  condition.
- If the loop will outlive one sitting, save progress notes to the card's
  State file so a future session can resume.
- Card or no card: per the diagnosis tiebreaker in SKILL.md, save a card
  only if the same kind of problem will come back.

## Owner-triggered manual loop

If launchd is impossible (headless auth unfixable, unsupported platform)
and the cadence is too sparse for an in-session `/loop`, save the card
anyway; the owner says "run one iteration of <name>" when it's time. A
manual loop beats one that silently fails.

## Alternatives to know about (don't default to them)

- **Routines** (cloud, research preview): Anthropic-hosted schedules;
  the laptop can be off. No local file access — loops built on cards,
  state files, or local binaries cannot use it. Right for purely-web
  loops only.
- **Desktop Scheduled Tasks**: native scheduling for Claude Desktop app
  users, local file access, survives reboots. Right when the person
  already lives in the Desktop app; this plugin targets terminal users.
```

- [ ] **Step 2: Verify the file**

Run: `grep -c "launchd" skills/build-a-loop/references/engines.md`
Expected: ≥ 8. Also `grep -n "Cron scheduled task" skills/build-a-loop/references/engines.md` → no matches.

- [ ] **Step 3: Commit**

```bash
git add skills/build-a-loop/references/engines.md
git commit -m "feat: launchd is the unattended engine; honest in-session cron limits"
```

---

### Task 3: Update `guardrails.md`

**Files:**
- Modify: `skills/build-a-loop/references/guardrails.md`

- [ ] **Step 1: Replace the headless-auth section**

Old:
```markdown
## Headless-auth check before any cron launch

Exercise each needed tool the way the cron job will run it before creating
the job. If a tool needs interactive auth, redesign, fall back to an
in-session `/loop`, or make the loop owner-triggered (see engines.md).
Never launch a loop that will silently fail at 9am.
```

New:
```markdown
## Headless check before any unattended launch

Two proofs before trusting any schedule: (1) the machine's plumbing probe
has passed for the current claude version (see engines.md — Keychain
auth, binary path, logs under real launchd conditions), and (2) every
tool the iteration needs has been exercised headless, the way the job
will run it, and is covered by the plist's allowlist. If either fails,
redesign, fall back to an in-session `/loop`, or make the loop
owner-triggered (see engines.md). Never launch a loop that will silently
fail at 9am.
```

- [ ] **Step 2: Append a new guardrail section at the end of the file**

```markdown
## Every unattended loop must be visible to loop-status

An unattended loop must follow the five-artifact convention (card, plist
label `com.claude-loops.<name>`, logs, state, output — see engines.md) so
the loop-status skill can see it. A loop the overview cannot see is a
loop that can fail silently. Tell every owner the health check: say
"show my loops".
```

- [ ] **Step 3: Commit**

```bash
git add skills/build-a-loop/references/guardrails.md
git commit -m "feat: guardrails for launchd probe and loop-status visibility"
```

---

### Task 4: Update `build-a-loop/SKILL.md` (steps 6, 8, card template)

**Files:**
- Modify: `skills/build-a-loop/SKILL.md`

- [ ] **Step 1: Update step 6 (pre-launch check)**

Old:
```markdown
Read `references/engines.md` (confirm the engine picked in step 4 and
follow its mechanics) and `references/guardrails.md` (apply every
guardrail). For unattended cron loops: exercise each tool the iteration
needs the way the cron job will (headless — interactive-auth MCP servers
often fail there; see engines.md for testing send-capable tools without
messaging third parties). If anything fails, redesign — different data
source, an
in-session loop, or an owner-triggered manual loop (see engines.md) — and
re-confirm the changed design with the person. Never launch a loop that
will silently fail.
```

New:
```markdown
Read `references/engines.md` (confirm the engine picked in step 4 and
follow its mechanics) and `references/guardrails.md` (apply every
guardrail). For unattended (launchd) loops: run the once-per-machine
plumbing probe if not yet recorded, derive the `--allowedTools` list by
enumerating every command and tool the card's Iteration steps invoke,
and exercise each tool headless the way the job will run it
(interactive-auth MCP servers often fail there; see engines.md for
testing send-capable tools without messaging third parties). If anything
fails, redesign — different data source, an in-session loop, or an
owner-triggered manual loop (see engines.md) — and re-confirm the changed
design with the person. Never launch a loop that will silently fail.
```

- [ ] **Step 2: Update the card template's Engine line**

Old:
```markdown
Engine:      <cron: <schedule, in plain words> | /loop <interval> |
              self-paced | owner-triggered>
```

New:
```markdown
Engine:      <launchd: <schedule, in plain words> · label
              com.claude-loops.<name> | /loop <interval> | self-paced |
              owner-triggered>
```

- [ ] **Step 3: Update step 8 (launch and hand off), first bullet**

Old:
```markdown
- Recurring, must run unattended → create the cron job. Its prompt is
  exactly: `Read ~/.claude/loops/<name>.md and execute one iteration per its
  instructions.`
```

New:
```markdown
- Recurring, must run unattended → with the person's consent (it writes
  system state), install the launchd job per engines.md: write the plist
  (its `claude -p` prompt is exactly `Read ~/.claude/loops/<name>.md and
  execute one iteration per its instructions.`), bootstrap it, and
  confirm the plumbing probe has passed. Tell them logs live in
  `~/.claude/loops/logs/` and the health check is saying "show my loops".
```

- [ ] **Step 4: Verify and commit**

Run: `grep -n "cron job" skills/build-a-loop/SKILL.md`
Expected: no matches (the word "cron" may remain only if part of an honest in-session reference; step 8's other bullets don't mention cron).

```bash
git add skills/build-a-loop/SKILL.md
git commit -m "feat: build-a-loop launches unattended loops on launchd"
```

---

### Task 5: Create the `loop-status` skill

**Files:**
- Create: `skills/loop-status/SKILL.md`

- [ ] **Step 1: Write the file:**

```markdown
---
name: loop-status
description: >
  Show every loop built with build-a-loop and whether it is healthy.
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
4. **not scheduled** — card exists, no plist: fine if Engine says
   owner-triggered or self-paced (say so); flag as orphaned otherwise.
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

If there are no loops at all, say so and point at `/loops:build-a-loop`.
```

- [ ] **Step 2: Verify plugin loads both skills**

Run: `claude -p "List the names of the skills available from the loops plugin, one per line, nothing else." --plugin-dir . --max-turns 1`
Expected output contains: `build-a-loop` and `loop-status`.

- [ ] **Step 3: Commit**

```bash
git add skills/loop-status/SKILL.md
git commit -m "feat: add loop-status skill (health overview of all loops)"
```

---

### Task 6: Cross-file consistency check

**Files:** none modified (fix-ups only if findings).

- [ ] **Step 1: Stale-reference sweep**

Run: `grep -rn -i "croncreate\|cron scheduled task\|chained one-shot" skills/`
Expected: matches only in engines.md's honest in-session-cron section. Any other match: fix it to point at the launchd engine, then amend/commit.

- [ ] **Step 2: Convention sweep**

Run: `grep -rn "com.claude-loops" skills/ | wc -l`
Expected: ≥ 6 (engines.md template + commands, guardrails, SKILL.md card template, loop-status). All five artifact paths in engines.md's table must appear identically in loop-status SKILL.md §1.

- [ ] **Step 3: Scenario re-test (spec validation item 4)**

Dispatch one subagent: "You are a non-technical marketing manager. Read `skills/build-a-loop/SKILL.md` and all four files in `skills/build-a-loop/references/` (repo root: this repo). Walk the journey for: 'every weekday at 8:30 I want a digest of broken campaign links, even when my laptop terminal is closed.' Report: which engine the docs lead you to, the exact launch mechanics the docs would have Claude perform, and any contradiction, dead reference, or step where the docs leave Claude without instructions." Expected: launchd engine; plist + probe + bootstrap mechanics; zero contradictions. Fix any finding, then re-run the failing part of the walkthrough.

- [ ] **Step 4: Commit any fixes**

```bash
git add -A skills/ && git commit -m "fix: cross-file consistency after launchd rewrite" || echo "nothing to fix"
```

---

### Task 7: Migrate `weekly-update-digest` to launchd (dogfood)

**Files:**
- Modify: `~/.claude/loops/weekly-update-digest.md`
- Create: `~/Library/LaunchAgents/com.claude-loops.weekly-update-digest.plist`
- Delete: in-session cron job `2f1ed60e` (via CronDelete)
- No repo changes; nothing to commit.

- [ ] **Step 1: Headless allowlist dry-run**

Run exactly (one line; this is the real allowlist with a read-only test prompt):
```bash
claude -p "Run: cd ~/.claude/skills/weekly-update-analyzer && ./fetch-updates -status -week 2026-06-11 — then report only the 'Filled out' line. Do not write any files or send notifications." --permission-mode dontAsk --allowedTools "Read(//Users/rapha/.claude/loops/**),Read(//Users/rapha/.claude/skills/weekly-update-analyzer/**),Write(//Users/rapha/.claude/loops/**),Edit(//Users/rapha/.claude/loops/**),Bash(date:*),Bash(cd ~/.claude/skills/weekly-update-analyzer && ./fetch-updates:*),Bash(cd /Users/rapha/.claude/skills/weekly-update-analyzer && ./fetch-updates:*),PushNotification" --max-turns 10
```
Expected: a "Filled out: N / 36" line, no permission errors. A permission error means an allowlist rule is wrong — fix the rule, re-run, and carry the fix into Step 3.

- [ ] **Step 2: Update the loop card**

In `~/.claude/loops/weekly-update-digest.md`:

a. Replace Iteration step 9 (the CronCreate chaining step, including its KNOWN LIMIT paragraph) with:
```
  9. Done — launchd fires this loop every Friday 9:03 AM; the step-1
     state gate turns that into every other Friday. No re-scheduling
     needed.
```
b. In the failure-handling paragraph, delete the clause "still do step 9;" and append: "If claude itself cannot start, the launchd wrapper notifies the owner and marks the .err log."
c. Replace the Engine block with:
```
Engine:      launchd: every Friday 9:03 AM, label
             com.claude-loops.weekly-update-digest; the 13-day state
             gate makes it every other Friday, anchored to the first
             full run (Fri 2026-06-12). Runs with no session open;
             machine must be awake or asleep (fires on wake), not off.
             Health check: say "show my loops".
```
d. In Needs, replace "cron (CronCreate, durable one-shots)" with "launchd LaunchAgent".
e. In Controls, replace the pause/stop sentence with: "pause/stop: tell Claude 'pause the weekly-update-digest loop' (unloads the launchd job; the card stays). Resume: 'resume the weekly-update-digest loop'. Run manually any time: 'run one iteration of weekly-update-digest'."
f. Verify Iteration step 7's notification text carries no session caveat — the old step 9's KNOWN LIMIT paragraph instructed appending one ("fires only if this session stays open"), and deleting step 9 in (a) must remove that instruction entirely.

- [ ] **Step 3: Write and install the plist**

Write `~/Library/LaunchAgents/com.claude-loops.weekly-update-digest.plist` from the engines.md template with: NAME=`weekly-update-digest`, USER=`rapha`, Weekday 5 / Hour 9 / Minute 3, TOOL_LIST = the (possibly fixed) allowlist from Step 1. Remember: ProgramArguments command on ONE line; `&&` as `&amp;&amp;` in the XML.

```bash
plutil -lint ~/Library/LaunchAgents/com.claude-loops.weekly-update-digest.plist
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.claude-loops.weekly-update-digest.plist
launchctl print gui/$(id -u)/com.claude-loops.weekly-update-digest | head -20
```
Expected: lint OK; print shows the job with the calendar interval. Do **NOT** kickstart — a run today would write state and gate out tomorrow's anchor run.

- [ ] **Step 4: Delete the session-only job**

Use CronDelete on job `2f1ed60e`. Expected: CronList no longer shows it.

---

### Task 8: Live-test `loop-status`

**Files:** temporary test artifacts only; all removed in Step 4. No commit.

- [ ] **Step 1: Baseline run**

Execute the loop-status skill's instructions in-session ("show my loops"). Expected report: `weekly-update-digest` → OK-pending (scheduled, never fired yet — state file absent is correct pre-first-run; verdict cites the plist + empty state); `doc-polish-loop` → not scheduled, and that's fine if its card says self-paced/owner-triggered (read it; report per classification rule 4); no orphans.

- [ ] **Step 2: Simulate "failing"**

```bash
echo "FAILED 2026-06-11 09:03" >> ~/.claude/loops/logs/weekly-update-digest.err
```
Re-run the skill. Expected: weekly-update-digest classified **failing**, citing the marker. Then: `rm ~/.claude/loops/logs/weekly-update-digest.err`

- [ ] **Step 3: Simulate "paused" and "overdue"**

Paused: `launchctl bootout gui/$(id -u)/com.claude-loops.weekly-update-digest`, re-run skill → expected **paused** (plist exists, label not loaded). Then re-bootstrap and confirm the verdict returns to OK-pending.

Overdue: create card `~/.claude/loops/test-dummy.md` (Engine: `launchd: daily 9:03, label com.claude-loops.test-dummy`, State: none) and plist `~/Library/LaunchAgents/com.claude-loops.test-dummy.plist` from the template with Hour 9 / Minute 3, **no Weekday key** (daily), bootstrapped — but with ProgramArguments `/usr/bin/true` so it never actually runs claude. If current time is past 11:03 (9:03 + 2h grace), re-run skill → expected **overdue** for test-dummy (scheduled fire passed, no state, no log). If before 11:03, expected OK-pending — note which branch was exercised.

- [ ] **Step 4: Clean up test artifacts**

```bash
launchctl bootout gui/$(id -u)/com.claude-loops.test-dummy
rm ~/Library/LaunchAgents/com.claude-loops.test-dummy.plist ~/.claude/loops/test-dummy.md
```
Expected: re-run skill → test-dummy gone, no orphan flags.

---

### Task 9: Push and report

**Files:** none.

- [ ] **Step 1: Push the branch**

```bash
git push origin main
```
Expected: tasks 2-6 commits (plus the spec/plan docs) land on github.com/syerad/claude-skill-loops.

- [ ] **Step 2: Report**

Summarize to the owner: probe result (the Keychain answer), what migrated, that tomorrow 9:03 AM Friday is the first real unattended run, and that the definitive validation is checking "show my loops" after 11:00 AM tomorrow (grace period) — expected verdict OK with a fresh report in `~/.claude/loops/output/`.
```
