# Loop recipes

Proven shapes, grounded in common workplace tools. Use them to ground
proposals (step 4) and as inspiration for blank-page users (step 1). Adapt
freely — these are starting points, not templates to force.

Format per recipe: Roles · Type · Problem / Measurement / Iteration sketch /
Needs / Output.

---

## Weekly-update completion chaser
Roles: EM · Type: recurring (scheduled, Mondays)
- Problem: Chasing direct reports for weekly updates eats Monday mornings.
- Measurement: Every missing update gets exactly one nudge; silent if
  everyone submitted; owner pinged only with the still-missing list.
- Iteration: Check the weekly-update doc/folder for the current week → list
  who hasn't submitted (vs. state file) → send each one short Slack nudge →
  update state → DM owner only if someone remains missing.
- Needs: Google Drive, Slack. Output: Slack DMs.

## PR review SLA watcher
Roles: EM, IC eng · Type: recurring (scheduled, weekday mornings)
- Problem: PRs sit unreviewed past the team's 24h norm.
- Measurement: Each PR >24h without review gets exactly one nudge to its
  reviewers; silent when none qualify.
- Iteration: List open PRs in the team's repos → filter by review-idle time
  (vs. state file) → one Slack nudge per PR → update state.
- Needs: GitHub, Slack. Output: Slack channel or DMs.

## Stale-PR pruner
Roles: IC eng · Type: recurring (scheduled, weekly)
- Problem: Dead PRs pile up and bury the active ones.
- Measurement: Every PR idle >30 days gets listed with a recommended action
  (close/rebase/ping); the loop recommends, never closes; silent when no
  PR qualifies.
- Iteration: List open PRs idle >30 days → classify each → post one summary
  with recommendations to the owner.
- Needs: GitHub. Output: Slack DM or issue comment.

## Flaky-test hunter
Roles: IC eng · Type: iterative (self-paced)
- Problem: CI fails intermittently; nobody has traced which tests flake.
- Measurement: Done when the top flaky tests are identified with evidence
  (failure rate over recent runs) and each has a ticket or fix proposed.
- Iteration: Pull recent CI runs → rank tests by intermittent failures →
  reproduce/inspect the worst → write up cause or ticket → re-rank; stop
  when remaining failures are deterministic.
- Needs: CI access (GitHub Actions). Output: tickets/report.

## Performance budget loop
Roles: IC eng · Type: iterative (self-paced)
- Problem: A page or bundle has drifted past its performance budget.
- Measurement: Metric loop — baseline first; keep a change only if the
  number improves; stop at the target (e.g. bundle <200KB, Lighthouse
  perf ≥90) or after 2 cycles without improvement.
- Iteration: Measure (build stats / Lighthouse run) → find the biggest
  contributor → apply one change → re-measure → keep or revert.
- Needs: the repo, the build/measure command. Output: PR with before/after
  numbers.

## Morning support-queue digest
Roles: Support · Type: recurring (scheduled, daily)
- Problem: Mornings start with manually triaging the overnight queue.
- Measurement: Digest contains only tickets needing action today (escalation
  risk, VIP, >24h waiting); silent if the queue is clean.
- Iteration: Pull overnight tickets → classify urgency → three-bullet digest
  with links, most urgent first → post to support channel.
- Needs: Zendesk. Output: Slack.

## Escalation watcher
Roles: Support, EM · Type: recurring (in-session /loop, 30m, during shift)
- Problem: High-priority tickets breach SLA while everyone is heads-down.
- Measurement: Any ticket within 1h of SLA breach gets flagged once; silent
  otherwise.
- Iteration: Check open high-priority tickets → compare to SLA clock (vs.
  state) → flag new at-risk tickets in the channel.
- Needs: Zendesk. Output: Slack.

## Competitor changelog watcher
Roles: PM · Type: recurring (scheduled, weekly)
- Problem: Competitor releases get noticed weeks late.
- Measurement: Only genuinely new items since last run; silent if nothing
  shipped.
- Iteration: Fetch competitor changelog/release pages (list URLs in card) →
  diff against state file → summarize new items with a why-it-matters line →
  update state.
- Needs: web access. Output: Slack or doc.

## Metrics anomaly digest
Roles: PM, EM · Type: recurring (scheduled, daily)
- Problem: Dashboard checking is manual and skipped on busy days.
- Measurement: Reports only metrics outside their normal band (define the
  band per metric in the card); silent when all normal.
- Iteration: Pull the named Looker dashboards/metrics → compare to bands →
  report breaches with links.
- Needs: Looker. Output: Slack.

## Content QA loop
Roles: Marketing · Type: iterative (self-paced)
- Problem: A draft (post, page, campaign) needs to hit the bar before review.
- Measurement: Done when the draft passes the checklist written into the
  card (voice, claims sourced, links valid, length, CTA present).
- Iteration: Review draft against checklist → fix the worst failure →
  re-check; stop when all items pass, then deliver the diff summary.
- Needs: the draft file/doc. Output: revised draft + checklist results.

## Campaign link health check
Roles: Marketing · Type: recurring (scheduled, daily during campaign)
- Problem: A live campaign's links can break or underperform unnoticed.
- Measurement: Flags only links that error or fall below the click threshold
  in the card; silent when healthy.
- Iteration: Check each campaign link (list in card) for resolution and
  click counts → compare to thresholds → flag breaches.
- Needs: link/campaign analytics, web. Output: Slack.

## Channel triage digest
Roles: anyone · Type: recurring (scheduled, daily)
- Problem: A busy Slack channel buries the messages that actually need you.
- Measurement: Digest lists only items needing the owner's reply or decision;
  silent if nothing does.
- Iteration: Read the channel since last run (state file) → filter to
  mentions/questions/decisions for the owner → three-bullet digest with
  permalinks → update state.
- Needs: Slack. Output: Slack DM.

## Doc polish loop
Roles: anyone · Type: iterative (self-paced)
- Problem: An important doc (proposal, spec, review packet) needs to be
  genuinely good, not first-draft good.
- Measurement: Done when the doc passes the card's checklist (audience-fit,
  one idea per section, no unsupported claims, skimmable headings).
- Iteration: Critique against checklist → fix the weakest section →
  re-critique; stop when the checklist passes.
- Needs: the doc. Output: revised doc + what changed.
