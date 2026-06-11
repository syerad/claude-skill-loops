# claude-skill-loops

A Claude Code plugin (`loops`) that helps you turn recurring chores and hard
problems into loops. Run `/loops:build-a-loop`, describe what keeps eating
your time, and walk away with a tested, running loop you own — no prompt
engineering required.

Every loop it builds has three parts: a **Problem** (what it solves), a
**Measurement** (how we know a run worked — and when it stays silent), and
an **Iteration** (what happens each cycle). The result is saved as a loop
card in `~/.claude/loops/` — edit the card to change the loop, share it so
a teammate can run their own copy.

## Install

From inside Claude Code:

```
/plugin marketplace add syerad/claude-skill-loops
/plugin install loops@claude-skill-loops
```

To pick up new versions later: `/plugin marketplace update claude-skill-loops`
(or enable auto-update for this marketplace in the `/plugin` UI).

For development, run straight from a checkout instead:

```
git clone https://github.com/syerad/claude-skill-loops.git
claude --plugin-dir /path/to/claude-skill-loops
```

Then start with:

```
/loops:build-a-loop I keep doing <annoying thing> every <week/day/...>
```

Check on your loops any time with `/loops:loop-status` — or just say
"show my loops".

## What's inside

- `skills/build-a-loop/SKILL.md` — the guided journey
- `skills/build-a-loop/references/` — recipe gallery, engine mechanics,
  prompt patterns, guardrails

## Contributing a recipe

Built a loop your team copies? Add its card to
`skills/build-a-loop/references/recipes.md` in the existing format (Roles,
Type, Problem, Measurement, Iteration, Needs, Output) and open a PR.
