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
are skipped — the status skill catches those as "overdue".

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
are for readability — keep it on one line in the real file.
StartCalendarInterval takes one dict or an array of dicts: "every
weekday at 8:30" is five dicts, Weekday 1-5, each with Hour 8 / Minute
30; omit Weekday entirely for every day.)

Key mechanics:

- **No `--bare`.** Bare mode skips Keychain/OAuth; subscription login
  requires a normal headless run. PATH in the plist must include the
  directory containing the `claude` binary (`which claude`) AND the
  directory of every binary the card's commands shell out to — check
  each with `which`. launchd's environment is not your shell: a loop
  has failed in the field because `gcloud` lived under /opt/homebrew
  and only the terminal PATH knew it. The step-6 dry-run only proves
  permissions; to prove PATH, run the dry-run with `env PATH=<exact
  plist PATH>` prefixed.
- **Notify via the OS, not the harness.** The PushNotification tool
  does not deliver from headless runs (verified). Any "notify the
  owner" step in a card destined for launchd must use
  `osascript -e 'display notification …'` (macOS) or `notify-send`
  (Linux), with `Bash(osascript:*)` in the allowlist.
- **Headless runs cannot use the Write/Edit tools.** Verified (Claude
  Code 2.1.173, via launchd and nested): in `claude -p` the Write and
  Edit tools are denied regardless of permission mode (`dontAsk`,
  `acceptEdits`) and regardless of allow rules, whether passed as
  flags or settings. Spawned processes are NOT tool-gated (a binary
  the card runs writes its own files freely), so loops write through
  a small allowlisted helper instead. Create it once per machine at
  `~/.claude/loops/bin/save`, `chmod +x` it, and allow it with both
  `Bash(~/.claude/loops/bin/save:*)` and the absolute-path form:

      #!/bin/sh
      # Write stdin to a file under ~/.claude/loops — the only write
      # path granted to headless loop runs.
      set -eu
      base="$HOME/.claude/loops"
      append=false
      if [ "${1:-}" = "-a" ]; then append=true; shift; fi
      rel="${1:?usage: save [-a] <path-relative-to-loops-dir>}"
      case "$rel" in
        /*|*..*) echo "save: path must be relative to $base, no '..'" >&2; exit 1;;
      esac
      dest="$base/$rel"
      mkdir -p "$(dirname "$dest")"
      if [ "$append" = true ]; then cat >> "$dest"; else cat > "$dest"; fi
      echo "saved: $dest"

  Cards write via heredoc (the command must START with the helper
  path so the allow rule matches):
  `~/.claude/loops/bin/save output/report.md <<'EOF' … EOF`,
  append with `-a`. The step-6 dry-run must exercise one write
  through the helper — a read-only dry-run validates only half the
  allowlist.
- **`dontAsk` + per-loop `--allowedTools`.** Least privilege. Enumerate
  every command and tool the card's Iteration steps invoke, as concrete
  rules (`Bash(<command prefix>:*)`, `Read(//abs/path/**)`). Compound
  commands are split for permission checks: `cd X && ./run Y` needs the
  parts allowed separately (`Bash(cd:*)`, `Bash(./run:*)`) — one rule
  containing `&&` never matches. Always validate the list with a
  headless dry-run before installing. An unlisted tool is denied, never
  prompts or hangs — but the run may still exit 0 with the model
  narrating the denial, and on exit 0 the plist's `||` failure wrapper
  does not fire: a denied iteration writes no state, and the status skill's
  overdue check is what catches it. If
  the owner later edits the card to need new tools, the allowlist must
  be refreshed: tell them to ask Claude to update it.
- **Failure wrapper.** The `|| { echo FAILED …; osascript …; }` tail
  covers the class where `claude` itself cannot run (auth broke, binary
  moved): the owner gets a macOS notification and `.err` gets a marker
  line the status skill can read. Failures *inside* the iteration are the
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
