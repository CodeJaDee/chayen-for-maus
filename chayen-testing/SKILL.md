---
name: chayen-testing
description: Automated testing knowledge for builder, verifier, and maintain bots with xUnit, NUnit, MSTest, Jest, Vitest, Jasmine/Karma, Playwright, Cypress, and pytest. Covers mapping acceptance criteria to tests, unit, integration, end-to-end, API and contract tests, regression tests for bugs, test data, determinism, flaky and failing test triage, coverage gaps by risk, and simple performance budget checks. Use with chayen-build, chayen-verify, or chayen-maintain. Not for manual QA plans or testing systems the team does not own.
---

# Testing

Write and judge tests that prove acceptance criteria, fail when behaviour
breaks, and give the same result every run.

**Operating mode.** Under `chayen-sdlc-loop`: do not ask; derive cases from
the spec's acceptance criteria and edge cases, and report results with its
evidence vocabulary. If that skill is not loaded: never report a test as
passed unless it ran; never weaken a test to get green.

## Detect the suite

Use the existing framework, assertion and mocking libraries, layout, and
naming. Never add a second framework.

| Stack | Look for | Integration helpers |
|---|---|---|
| .NET | `xunit`, `NUnit`, `MSTest` in test `.csproj` | `WebApplicationFactory`, Testcontainers, the project's fakes |
| JS/TS | `jest`, `vitest`, `karma`, `@playwright/test`, `cypress` in `package.json` | MSW, Testing Library, Playwright route mocking |
| Python | `pytest`, `unittest`, Django `TestCase` | fixtures, `TestClient`, `pytest-django`, Testcontainers |

Find the run command in scripts and CI, not from defaults.

## From acceptance criteria to tests

1. For each AC, choose the lowest level that proves it:
   - pure logic → unit;
   - HTTP behaviour, auth, persistence, serialization → integration through
     the real pipeline with a real database engine in a container;
   - a user journey across UI and API → end-to-end, only for the main path
     and critical failures.
2. Name the test so it reads as the criterion (`Returns403_WhenUserDoesNotOwnOrder`,
   `shows empty state when no orders match filters`).
3. List the cases before writing: happy path, boundaries, invalid input,
   error paths, authorization allowed and denied, concurrency where the spec
   mentions it, and each UI state the spec lists.
4. Record AC → test name in `plan.md`; the verifier checks it.

## Rules

- **Behaviour through public surfaces.** Do not test private methods or
  implementation details; one reason to fail per test; arrange, act, assert.
- **Deterministic.** Control time (inject a clock, fake timers), randomness
  (seed), network (fakes, MSW, recorded fixtures), and order (no shared
  mutable state between tests). No `sleep`; wait for a condition.
- **Expected values come from the spec,** never from current output. Never
  bulk-update snapshots or golden files to make a run green.
- **Never skip, delete, retry, or loosen** a test to get green. A changed
  expectation needs the spec AC that changed it, in its own commit.
- **Regression tests** for bugs fail on the code before the fix and pass
  after. Record which you observed.
- **Test data:** builders or factories already in the project; synthetic
  data only; no production copies; no secrets.
- **No live systems:** tests never call production services or real
  third-party APIs.

## API and contract tests

Status code, response shape, error shape, and headers for each endpoint the
spec touches. If the project uses consumer-driven contracts (Pact) or
schema checks against an OpenAPI file, update and run them; a breaking
contract change fails them on purpose.

## Advanced techniques, when they pay

| Technique | Use when |
|---|---|
| Property-based (FsCheck, fast-check, Hypothesis) | parsers, serializers, money and date arithmetic, round trips |
| Mutation (Stryker, mutmut) | a critical module where coverage looks high but tests may not assert enough |
| Fuzzing | input parsers and file handlers in code the team owns |

Bound the run time; report the command and the score.

## Performance budgets

When the spec has a measurable target (for example p95 under 300 ms for an
endpoint), add a repeatable check with the project's tool (k6, NBomber,
BenchmarkDotNet, Locust, Lighthouse CI) against a local or disposable
environment. Report the numbers and the conditions. Never load-test shared
or production systems.

## Failing and flaky tests

1. Classify each failure: new (caused by this change), pre-existing (fails
   on the default branch), flaky (passes and fails on the same code),
   environment (missing service, tool, or credential).
2. For a new failure, decide test defect or product defect from evidence.
   A product defect is fixed in code, not in the test.
3. For flaky tests, find the cause class: time, order, shared state, async
   wait, network, resource limits. Show repeated-run counts with the
   command (for example 20 runs, 0 failures after the fix).

## Coverage gaps

Rank untested code by risk, not by percentage: authorization, money, data
writes, error handling, and code changed often. Propose tests for the top
items as a new intent; do not chase a coverage number.

## Result reporting

Use PASS, FAIL, SKIPPED, BLOCKED, NOT_RUN, UNKNOWN with the exact command
and the runner's counts. A test that did not run in this session is never
PASS. Zero tests matched a filter is UNKNOWN, not PASS.

## Failure handling

| Situation | Action |
|---|---|
| A new test exposes a product bug outside the spec | Keep the test out of this PR; write an `intent.md` with the failing test as evidence |
| Dependency unavailable (DB, browser) | Testcontainers or the project's fakes; otherwise write the test and report BLOCKED with the command |
| No test project exists | Do not scaffold one inside a feature PR; propose it as a new intent |
| Code is hard to test | Add the smallest seam the spec allows; larger refactors become a new intent |
