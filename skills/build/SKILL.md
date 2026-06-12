---
name: build
description: Use when someone wants to automate a recurring task, set up a monitor/digest/reminder, or solve a hard problem by iterating until it's done — triggers include "build a loop", "automate this", "I keep doing X every week", "can Claude check Y every morning", "keep improving Z until it passes". Works for non-coding tasks (inbox digests, report chasing, content QA) as well as engineering tasks.
---

# Build a Loop

Help the person turn a problem into a working loop. **The product is the
loop prompt itself** — a precise, tested iteration prompt packaged as a loop
card. Launching it is just delivery. Most users are not prompt engineers and
may not be technical at all: keep language plain, do the mechanics for them.

## Loop anatomy — every loop, no exceptions

1. **Problem** — what this loop exists to solve, in one sentence.
2. **Measurement** — how we know an iteration worked.
   - Iterative loop: the termination condition (checkable, written down) —
     pass/fail, or a metric with a stop rule (see prompt-patterns.md).
   - Recurring loop: what a useful run produces AND when the loop must stay
     silent. No silence condition → notification spam → abandoned loop.
3. **Iteration** — what happens each cycle, plus cadence (clock-driven) or
   pacing (goal-driven).

A loop without a measurement does not launch.

## Journey

Follow the steps in order. Ask ONE question at a time. Don't show off
mechanics (cron syntax, tool names) unless the person is technical.

### 1. Problem intake
If they stated a problem, restate it in one sentence and confirm. If they
arrived empty-handed, ask: "What's a task you keep doing over and over — or a
goal you'd like Claude to keep working at until it's done?" (The second half
matters: people rarely realize iterate-until-done work belongs here too.)
Only if they have nothing: ask their role,
then read `references/recipes.md` and show the 2-3 most relevant recipes
(role-tagged ones first, then "anyone" ones) as inspiration.

### 2. Interview — up to 4 questions, one at a time
- How often does this come up — or when would you want it handled?
- Where does the information live? (Slack, Jira, GitHub, Looker, Zendesk,
  Google Drive, a website…)
- What would a genuinely useful output look like, and who should see it?

Stop interviewing as soon as you can fill in Problem/Measurement/Iteration.

### 3. Diagnose
- Repeats on a calendar or clock → **recurring loop**.
- One goal with a checkable done-condition → **iterative loop**.
- Happens once, or needs fresh human judgment every time → **not a loop**.
  Say so honestly: offer to do the task right now, or to help build a
  reusable skill instead. Do not build a loop nobody needs.

Tiebreaker — one-off goal with a checkable done-condition: still not a
loop. Work it right now, iterating against that condition until it passes,
but save no card and schedule nothing. Save an iterative loop card only
when the same kind of problem will come back (the card is reusable: polish
any doc, hunt any flaky test). If unsure whether it will come back, ask.

### 4. Propose
Read `references/recipes.md`. Propose 1-2 concrete designs — grounded in a
recipe when one fits, from scratch when not. Each proposal states: Problem,
Measurement, Iteration, engine (picked via the decision table in
`references/engines.md`), where output goes, what access it needs.
Let the person pick.

### 5. Compile and test the loop prompt
Read `references/prompt-patterns.md`. Write the iteration prompt. Then run
**one iteration live, right now**, and show the person the real output.
Run it read-only: gather and check everything for real, but show drafts of
any outward messages instead of sending them — the test must not nudge or
notify anyone, and must not write the loop's state file (the first real
run must not believe the test's actions happened). For iterative loops
whose work is editing an artifact, editing it IS the test — read-only
means no outward messages and no state writes. Refine the prompt until
they say the output is genuinely useful. Never skip this step; this is where the prompt gets debugged.

### 6. Pre-launch check
Read `references/engines.md` (confirm the engine picked in step 4 and
follow its mechanics) and `references/guardrails.md` (apply every
guardrail). For unattended (launchd) loops: run the once-per-machine
plumbing probe if not yet recorded, derive the `--allowedTools` list by
enumerating every command and tool the card's Iteration steps invoke,
and exercise each tool headless the way the job will run it
(interactive-auth MCP servers often fail there; exercise send-capable
tools by sending only to the owner or a test target, never to third
parties). If anything
fails, redesign — different data source, an in-session loop, or an
owner-triggered manual loop (see engines.md) — and re-confirm the changed
design with the person. Never launch a loop that will silently fail.

### 7. Save the loop card
Save the card to `~/.claude/loops/<kebab-case-name>.md` (create the
directory if needed) with exactly this structure:

```markdown
# <Loop name>

Problem:     <one sentence>
Measurement: <successful iteration looks like X; stay silent when Y;
              iterative loops: terminate when Z>
Iteration:   <numbered steps each cycle: check what, where, do what,
              output where>
Engine:      <launchd: <schedule, in plain words> · label
              com.claude-loops.<name> | /loop <interval> | self-paced |
              owner-triggered>
Needs:       <tools/integrations/auth required>
State:       <path to state file if the loop tracks what it already did,
              e.g. ~/.claude/loops/state/<name>.md — or "none">
Controls:    owner: <name> · created: <date>
             pause/stop: <what the owner says to Claude to pause or stop it>
```

Loop-specific data — rosters, message templates, URL lists, checklists,
exception lists — lives inside the Iteration field: the loop must run from
the card alone.

### 8. Launch and hand off
The card now exists, so the loop can point at it:

- Recurring, must run unattended → with the person's consent (it writes
  system state), install the launchd job per engines.md: write the plist
  (its `claude -p` prompt is exactly `Read ~/.claude/loops/<name>.md and
  execute one iteration per its instructions.`), bootstrap it, and
  confirm the plumbing probe has passed. Tell them logs live in
  `~/.claude/loops/logs/` and the health check is saying "show my loops".
- Recurring, only matters while they're working → tell them to run
  `/loop <interval>` with the same card-reading prompt, and that it stops
  when the session closes.
- Iterative → start now in this session, self-paced; keep iterating until
  the Measurement passes, then report.
- Owner-triggered (headless auth unfixable, cadence too sparse for an
  in-session loop) → nothing to schedule; tell them to say "run one
  iteration of <name>" when it's time.

Then tell the person, in plain words: the card IS the loop — edit the card
to change the loop's behavior (to change how often it runs, also ask
Claude to update the schedule); here is how to pause or stop it; share the
card file with a teammate so they can run their own copy.
