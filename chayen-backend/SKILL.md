---
name: chayen-backend
description: Server-side knowledge for builder and verifier bots - ASP.NET Core (C#, EF Core, Dapper), Node servers (Express, Fastify, NestJS, Next.js route handlers and server actions, Nuxt server routes), and Python (FastAPI, Django, Flask). Covers API contracts and versioning, endpoints, validation, authentication and authorization, background jobs, messaging, external integrations, webhooks, file processing, and idempotency. Use with chayen-build or chayen-verify. Not for UI, infrastructure-only work, or other backend stacks.
---

# Backend

Implement or check server-side behaviour that matches the project's stack,
API style, data access, and error conventions.

**Operating mode.** Under `chayen-sdlc-loop`: do not ask; decide from the
spec and the repository, record assumptions, and declare risky changes in
`plan.md` and the PR. If that skill is not loaded: never invent a contract,
permission, or business rule; never touch shared or production systems;
never print secrets.

## Detect first

Read the owning unit's manifests before writing code. In a monorepo, a
sibling's stack does not decide this unit's stack.

| Signal | Stack | Check commands usually in |
|---|---|---|
| `*.csproj` with `Microsoft.NET.Sdk.Web` | ASP.NET Core | `dotnet build`, `dotnet test`, `dotnet format --verify-no-changes` |
| `package.json` with `express`, `fastify`, `@nestjs/*` | Node server | the `scripts` block; lock file decides npm, pnpm, or yarn |
| `next` with `app/**/route.ts` or `'use server'` | Next.js server | `next build`, project test script |
| `nuxt` with `server/api/` | Nuxt server routes | `nuxi build`, project test script |
| `pyproject.toml` or `requirements*.txt` with `fastapi`, `django`, `flask` | Python web | `pytest`, `ruff`, `mypy` if configured |

Then find the closest analogous endpoint, job, consumer, or client and
mirror it: routing style (controllers or minimal APIs, routers, view sets),
DTO naming, validation library, error shape, logging, DI, and test style.

## API contract rules

- Request and response models are explicit types, never persistence
  entities or ORM models.
- Status codes follow the project's pattern; if none: 200/201/204 success,
  400 validation, 401 unauthenticated, 403 forbidden, 404 not found, 409
  conflict, 422 only if already used, 5xx never for client errors.
- One error shape for the whole API (ASP.NET Core: `ProblemDetails`;
  FastAPI: the project's exception handler model). Never leak stack
  traces, SQL, or exception text.
- Unbounded lists are paginated. Filters and sort fields are allow-listed.
- Cancellation flows to I/O (`CancellationToken` in .NET, `AbortSignal` in
  Node, async cancellation in Python).
- Additive changes by default. Removing or renaming a field, changing a
  type, or tightening validation is breaking: it needs the spec, a version,
  or a deprecation window.

## Authorization

- Every new endpoint, action, job trigger, and webhook has an explicit
  authorization decision, using the project's mechanism (policies and
  `[Authorize]`, route guards, dependencies, decorators, middleware).
- Check resource ownership, not only role: user A cannot read B's order by
  changing the ID.
- Tests cover the allowed path and the denied path at request level
  (`WebApplicationFactory`, `supertest`, `TestClient`, Django test client).
- Never weaken an existing policy to make a test pass.

## Data access

- Parameterized queries only. No string-built SQL with input, including
  `ORDER BY` columns (allow-list them).
- A transaction where several writes must succeed together; keep it short
  and never hold one across an external call.
- Avoid N+1: project the fields you need, include or batch related data.
- Schema changes go through the migration tool (`chayen-data`).

## Patterns by concern

| Concern | Rules |
|---|---|
| Background jobs | use the project's runner (Hangfire, Quartz, hosted services, BullMQ, Celery, RQ); idempotent handlers; retries with backoff; no work in request threads that outlives the request |
| Messaging | at-least-once assumed: consumers are idempotent (dedupe key or natural idempotency); poison messages go to a dead-letter path; publish after commit (outbox if the project has one) |
| External HTTP | typed client from the project's factory (`IHttpClientFactory`, a shared fetch wrapper, `httpx.AsyncClient`); timeouts on every call; retries only for idempotent requests; circuit breaking if the project has it |
| Webhooks in | verify the signature over the raw body before parsing; reject old timestamps; dedupe by event ID; respond fast, process async |
| Webhooks out | sign payloads; retry with backoff; record delivery attempts |
| File processing | size and type limits; never trust the client file name or content type; stream large files; scan if the project has a scanner; store outside the web root |
| Idempotency | client-supplied idempotency key for non-idempotent creates and payments; unique constraint backs it; replay returns the first result |
| Money | decimal types, never float; currency stored with amount; rounding rule from the spec |
| Time | store UTC; convert at the edge; inject a clock for tests |

## Bug path

1. Reproduce with a failing request-level or unit test.
2. Find the root cause from the stack trace and code; cite `path:line`.
3. Fix the cause, not the symptom. Keep the test.

## Checks before handing off

| Check | Evidence |
|---|---|
| Build, lint, typecheck, format | project commands, PASS |
| Tests | new and affected tests PASS; regression test failed before the fix |
| Authorization | allowed and denied request tests for every new endpoint |
| Contract | API summary in the PR: method, route, request, responses, errors |
| Migrations | generated SQL reviewed; down path described (`chayen-data`) |
| External calls | none to real third parties in tests; fakes or recorded fixtures |

## Failure handling

| Situation | Action |
|---|---|
| Cannot reproduce | Record what was tried; do not ship a speculative fix as confirmed; mark `blocked` with the data needed |
| Test needs a database or broker | Testcontainers or the project's fakes; otherwise BLOCKED with the command |
| Fix needs a breaking contract change | Not in the spec → `blocked` with options (version, additive, deprecate) |
| Needed library absent | Use the platform's built-in first; a new one follows the dependency rule in `chayen-build` |
| Mixed styles in the repo | Follow the module you are editing; note it in the PR |
