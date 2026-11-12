---
name: chayen-data
description: Relational data knowledge for builder, verifier, and release bots on SQL Server and PostgreSQL through EF Core, Dapper, Prisma, Drizzle, SQLAlchemy and Alembic, Django, Flyway, DbUp, or raw SQL. Covers schema modeling, expand-migrate-contract migrations, data migrations and backfills, query and index tuning, transactions and concurrency, integrity and data quality, retention, and environment safety rules. Use with chayen-build, chayen-verify, or chayen-release. Not for NoSQL, warehouses, MySQL, or database provisioning.
---

# Data

Change and check relational data so that every migration is reversible or
staged, verified on a disposable database, and applied to shared
environments only by the pipeline.

**Operating mode.** Under `chayen-sdlc-loop`: do not ask; follow the spec and
the existing schema, verify on disposable databases, and declare every
migration under Risk. If that skill is not loaded: never run writes against
any shared or production database; never invent retention periods or
business invariants.

## Know the engine and the tool

Find the engine and version (connection config, docker-compose, IaC,
migration history), and the migration tool. Never write engine-specific SQL
for an unknown engine. SQL Server and PostgreSQL differ in identity columns,
`TOP` vs `LIMIT`, `MERGE` semantics, case sensitivity, transactional DDL
(PostgreSQL yes, SQL Server mostly), and online index options.

Take table, column, and type names from migrations, models, or the catalog;
never guess one.

## Environment classes

| Class | Signals | Bot may |
|---|---|---|
| Disposable | Testcontainers, a container started for this work, throwaway DB | read, write, migrate, drop |
| Shared dev / test | team server, shared connection string | read; migrations only through the pipeline |
| Staging | `stg`, `staging`, `uat`, `preprod` | read with granted access; treat data as production |
| Production | production config or any doubt | bounded read-only queries with granted access; nothing else |

When unsure, treat it as production.

## Migration rules

- Generate with the project's tool (`dotnet ef migrations add`, `prisma
  migrate dev --create-only`, `alembic revision --autogenerate`, `python
  manage.py makemigrations`, a new Flyway/DbUp script) and **read the
  generated SQL**. Fix what the generator got wrong (renames generated as
  drop + add, missing defaults, wrong types).
- **Expand → migrate → contract.** Each release works with the previous
  application version:
  1. Expand: add nullable columns or new tables, new indexes, new
     constraints `NOT VALID`/`WITH NOCHECK` where supported.
  2. Migrate: backfill in batches; dual-write if needed.
  3. Contract (a later release): drop old columns, add `NOT NULL`, validate
     constraints.
- Never combine a destructive contract step with the code change that stops
  using the old shape.
- Large tables: create indexes online (`CREATE INDEX CONCURRENTLY` on
  PostgreSQL, `ONLINE = ON` on editions that support it); avoid table
  rewrites; estimate lock duration and state it in Risk.
- Every migration has a down path, or the PR says why it is irreversible
  and what the restore plan is.
- No automatic migration at application startup in shared environments
  unless the project already does it deliberately.

## Data migrations and backfills

Idempotent, batched by key range, resumable, with a progress log. Before:
detection query (how many rows need it). After: reconciliation query (how
many rows are still wrong — target zero). Both queries go in the PR and in
`release.md`.

## Queries and performance

- Read the plan (`EXPLAIN` without `ANALYZE` for writes on PostgreSQL; the
  estimated plan on SQL Server) before and after. A performance claim cites
  measurements under the same conditions, or says "expected, not measured".
- Index for the query's predicates and sort; check write cost and existing
  overlapping indexes.
- Avoid N+1 from ORMs; project only needed columns; paginate with keyset for
  deep pages.

## Transactions and integrity

- The database enforces invariants: primary and foreign keys, unique
  constraints, check constraints, `NOT NULL`. Application checks alone are
  not enough.
- Keep transactions short; no external calls inside them.
- Concurrency: optimistic concurrency (row version / `xmin`) for user edits;
  unique constraints for idempotency; explicit isolation only where the
  spec needs it, with the reason.
- Existing rows that violate a planned constraint: count them and propose a
  cleanup as part of the spec; never delete or "fix" data silently.

## Retention and personal data

Retention periods, purge rules, and legal holds come from the spec, never
invented. Purges are batched, logged, and reversible until the retention
window passes. Query results in PRs and rooms show counts and keys, not
personal values.

## Verification

| Check | Evidence |
|---|---|
| Migration | apply → rollback → re-apply on a disposable database, PASS |
| Data tests | integration tests against the real engine version (Testcontainers) PASS |
| Backfill | detection and reconciliation queries with results on test data |
| Performance | before/after plans or timings, or "not measured" |
| Safety | no command ran against a shared or production database outside the pipeline |

## Failure handling

| Situation | Action |
|---|---|
| Engine or version unknown | Detect from config; if still unknown, `blocked` |
| No disposable database possible | Mark migration checks NOT_RUN with exact commands; do not substitute a shared database |
| A change would drop, narrow, or rewrite data | Use expand/contract; if the spec requires destruction, put it in `release.md` for gate 3 with impact counts and a verified restore point |
| Migration lock time too long for the window | Risk high; propose an online or batched alternative |
