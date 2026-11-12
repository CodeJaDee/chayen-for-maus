---
name: chayen-architecture
description: Architecture knowledge for analyst and verifier bots when a spec is structural - reads ADRs in force, identifies the current style from evidence, compares options with trade-offs, drafts ADRs, defines module and service boundaries, domain models, data flows, multitenancy, resilience, and realtime design, plans phased migrations and staged rollouts, and adds fitness tests for dependency rules. Use with chayen-intent-spec or chayen-verify. Not for implementing features.
---

# Architecture

Make structural decisions explicit, evidence-based, and reviewable at gate
1, and keep accepted decisions enforced afterwards.

**Operating mode.** Under `chayen-sdlc-loop`: the output goes into `spec.md`
(Design and Options sections) and an ADR draft in the spec PR; a person
accepts it at gate 1 by merging. Do not ask; present options with a
recommended default. If that skill is not loaded: never mark a decision
accepted yourself.

## Use when the spec is structural

New service or module, new data store, change to module boundaries or
dependencies, tenancy, auth model, sync vs async integration, realtime
channels, multi-region or failure handling, migration between platforms,
or splitting a monolith.

## Workflow

1. **Read the decisions in force:** `docs/adr/`, `docs/architecture/`,
   `adr/`, decision registers. Note every accepted decision that constrains
   the change, with its ID.
2. **Identify the current style from evidence** (layered, clean or
   hexagonal, vertical slice, modular monolith, services, event-driven).
   Cite project references, folder structure, and dependency direction with
   `path:line`. Label anything not shown in code `inferred` or `unknown`.
3. **State the quality goals** from the spec (latency, availability, cost,
   time to market, team size). If the spec has none, choose the plausible
   priority, mark it an assumption, and show how the answer changes under
   the other priority.
4. **Compare options.** Always include "keep the current approach" or the
   smallest change. For each option: how it works here, cost, risks, what it
   rules out, and what would change the answer.
5. **Recommend one**, labelled proposed, with at least one rejected
   alternative and its reason.
6. **Draft the ADR** in the repository's format and location (or
   `docs/adr/NNNN-<slug>.md`): Context, Decision, Status `proposed`,
   Consequences, Alternatives. A change to an accepted ADR is a new ADR that
   supersedes it; never edit the old one.
7. **Add enforcement** where tooling can hold the decision: an architecture
   or fitness test (NetArchTest or ArchUnitNET for .NET, dependency-cruiser
   or eslint boundaries for TypeScript, import-linter for Python), written
   as an AC so the builder adds it.

## Design topics, compact

| Topic | Decide and record |
|---|---|
| Boundaries and domain | modules by business capability; dependencies point inward; aggregates own their invariants; one owner per table |
| Data flow | sequence diagram (Mermaid) of the main path and one failure path |
| Integration | sync only when the caller needs the answer now; async with idempotent consumers otherwise; contracts versioned |
| Multitenancy | isolation model (shared schema with tenant key, schema per tenant, database per tenant); tenant resolved once and enforced in data access; tests for cross-tenant denial |
| Resilience | timeouts on every remote call; retries only for idempotent calls with backoff and jitter; circuit breaker or bulkhead for flaky dependencies; degraded mode defined |
| Realtime | WebSockets, SignalR, or SSE; connection limits, backplane for scale-out, auth on connect, reconnection behaviour |
| Migration | strangler steps, each releasable; data sync strategy; cut-over and rollback criteria; decommission last |
| Rollout | flags, canary percentages, abort criteria tied to metrics, who watches |

## Checks

- Every statement about the current system cites evidence or is labelled
  `inferred` or `unknown`.
- No accepted ADR is contradicted silently; conflicts name the ADR ID.
- The recommendation lists costs and at least one rejected alternative.
- Migration and rollout plans name the steps that need gate 3 (data
  migrations, cut-overs, traffic shifts, decommissioning).
- Fitness tests, when written, run in CI; result reported with the command.

## Verifier use

When reviewing a structural PR, check that the code follows the accepted
ADR and the spec's design: dependency direction, module boundaries, data
ownership, and the fitness tests. A violation is an Important finding with
the ADR ID.
