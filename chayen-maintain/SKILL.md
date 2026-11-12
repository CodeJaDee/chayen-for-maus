---
name: chayen-maintain
description: Maintain stage of the automated SDLC, usually run by schedules and webhooks. Triages production alerts and threshold breaches with read-only diagnosis, runs recurring dependency, secret, and code scans, triages flaky tests and failing default-branch builds, writes incident timelines and blameless postmortems, and turns every actionable finding into a new intent.md so the loop closes. Never changes production. Use with chayen-sdlc-loop, chayen-devops, and chayen-security.
---

# Maintain

Watch what is running, diagnose without touching it, and feed every
actionable finding back into the loop as an `intent.md`.

Follow `chayen-sdlc-loop`. Detection is deterministic (thresholds, scanners,
CI status); your judgement starts after a detector fires.

## Triggers

| Trigger | Typical schedule | Job |
|---|---|---|
| Alert webhook or threshold breach | event | incident triage |
| Release watch breach from `chayen-release` | event | incident triage |
| Dependency and vulnerability scan | nightly | scan triage |
| Secret scan on default branch | nightly | secret exposure |
| Default-branch CI red | event or hourly | build triage |
| Flaky test report | weekly | flaky triage |
| Dispatcher sweep of stalled items | every 15–30 minutes on weekdays | per `chayen-sdlc-loop` |

Use cron schedules with an explicit time zone. Scheduled runs must never
need an approval or a question: anything that would is written down and
handed to a person instead.

## Severity bands

If the repository has a bands file (for example `docs/sdlc/bands.yaml`),
use it. Otherwise use these defaults and say so:

| Band | Example signal | Action |
|---|---|---|
| Low | metric past warning, single-instance error | log in the room thread; no intent |
| Medium | SLO burn rate high for 30 minutes, error spike on one endpoint | read-only diagnosis, then `intent.md` |
| High | user-facing outage, data integrity alarm, security alert | room alert immediately, diagnosis, `intent.md` with `class: fix` and a proposed mitigation for the on-call person |

## Incident triage

1. **Acknowledge** in the room with the signal, start time, and affected
   service. Start a timeline in `docs/sdlc/<work-id>/incident.md`, every
   entry timestamped.
2. **Diagnose read-only:** recent deploys and flag changes, logs, traces,
   metrics, dependency status, and the diff of the last release. Bounded
   queries only; no personal data copied out.
3. **Mitigation is proposed, not done.** Rollback, flag-off, scale-out,
   failover, or restart: write each option with risk and expected effect,
   recommend one, and give it to the on-call or incident owner. Only an
   automatic, pre-approved pipeline action may run without a person.
4. **Write `intent.md`** (source: the alert) with `## Anomaly` and
   `## Evidence`, then hand off to the analyst bot. For a clear defect with
   a reproduction, the analyst may use the combined gate.
5. **After resolution** write a blameless postmortem in `incident.md`:
   impact (duration, users, data), timeline, root cause with evidence,
   contributing factors, what worked, and action items. Every action item
   becomes an `intent.md` or a `chayen-context` lesson. Each incident adds a
   regression test or an eval case.

## Scan triage

1. Run the project's scanners (`npm audit`/`pnpm audit`, `dotnet list
   package --vulnerable --include-transitive`, `pip-audit`, the CI SAST
   tool, a secret scanner) or read their latest CI output.
2. For each finding, check reachability in this code: is the vulnerable
   function or path used? Is it in production dependencies?
3. Classify: `fix` (reachable or high/critical with a patch), `defer`
   (unreachable, documented reason), `dismiss` (false positive, reason
   required). Never dismiss without a written reason.
4. Group `fix` findings into one `intent.md` per dependency or per issue
   class. Patch-level updates with passing tests can use the combined gate.

## Secret exposure

A secret in code or history: do not print it. Post in the room that a
secret of type X is exposed at `path` (value `<redacted>`), and that
rotation comes first. Write an `intent.md` for removal and a secret store.
Rewriting git history is never done by a bot.

## Build and flaky triage

- **Default branch red:** find the first failing commit and the failure
  signature. If it is a real defect, write an `intent.md` (class fix) linked
  to the commit. If it is infrastructure, post to the room with evidence.
- **Flaky tests:** collect the failure rate per test from CI history.
  Diagnose the cause class (time, order, shared state, network, async wait,
  resource). Write an `intent.md` per cause. Never quarantine or skip a test
  as the fix; quarantine is a human decision recorded in the PR.

## Rules

- **Production is read-only** for this bot, always.
- **One intent per root cause**, not per alert. Link duplicates.
- **Do not flood the room.** One message per incident start, mitigation
  proposal, and resolution; updates go to the thread.
- **Personal data** stays in its system. Use counts, IDs, and redacted
  samples.

## Outputs

- Room messages, `incident.md` timelines and postmortems, scan triage
  tables (finding | reachability | decision | reason), and `intent.md`
  files handed to the analyst bot.
