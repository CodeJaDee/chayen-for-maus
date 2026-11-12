---
name: chayen-build
description: Build stage of the automated SDLC. Takes an accepted spec.md, writes plan.md, implements the smallest correct change in .NET, TypeScript or JavaScript, or Python with tests, runs the project's real build, lint, and tests, opens or updates the build PR, and fixes CI failures and verifier rework without asking. Covers features, bug fixes, refactors, upgrades, config and flags, and codegen. Use with chayen-sdlc-loop and the matching domain skill.
---

# Build

Turn an accepted spec into a green, reviewable PR that does exactly what the
spec says, follows the repository's conventions, and proves each acceptance
criterion with fresh evidence.

Follow `chayen-sdlc-loop` for gates, autonomy, handoff, and limits. Load the
domain skill for the code you touch: `chayen-backend`, `chayen-frontend`,
`chayen-data`, `chayen-testing`, `chayen-devops`.

## Start conditions

- `docs/sdlc/<work-id>/spec.md` is on the default branch (accepted), or the
  item uses the combined gate for a low-risk fix.
- Or: rework on an existing build PR (CI red, `verify.md` verdict
  `rework`, or code owner review comments).

If neither holds, do not build. Hand the item back per the dispatcher table.

## Workflow

1. **Orient.** Read `spec.md`, `AGENTS.md` or `CLAUDE.md`, the closest
   analogous code, and the real commands from CI files and scripts
   (`package.json`, `*.csproj`/`Directory.Build.props`, `pyproject.toml`,
   `Makefile`, `.github/workflows/`, `azure-pipelines.yml`). In a monorepo,
   use the owning unit's manifests and commands. Read only what the change
   needs.
2. **Baseline.** On a fresh branch `sdlc/<work-id>/build` from the default
   branch, run build and the relevant tests. Record pre-existing failures so
   they are never attributed to this change.
3. **Write `plan.md`** and commit it first:

   ```markdown
   ---
   id: <work-id>
   stage: plan
   status: in_progress
   source: [spec.md]
   updated: <YYYY-MM-DD>
   bot: <bot name>
   ---
   ## Approach      one paragraph
   ## Steps         ordered; each: files, what changes, which AC it serves
   ## Tests         AC → test name → level (unit, integration, e2e)
   ## Risk          migrations, auth, new deps, contract changes, config — each with mitigation
   ## Baseline      commands and pre-existing failures
   ## Divergences   filled when implementation departs from the plan
   ```

   Self-accept the plan when every AC has a test and every step maps to a
   requirement. If the spec cannot be implemented as written (contradiction,
   missing contract, an accepted ADR blocks it), mark the item `blocked`
   with the exact spec change needed. Do not improvise the product.
4. **Tests first where it fits.** For a defect, commit a failing test that
   reproduces it before the fix. For a feature, write the AC tests early.
5. **Implement in small steps.** Keep the build green after each step.
   Commit with messages that name the work-id. If the plan changes, update
   `## Divergences` in the same commit.
6. **Verify locally** with the project's commands: build, typecheck, lint,
   format check, unit and integration tests, and any check CI runs that can
   run locally. Compare with the baseline.
7. **Self-check the diff** (checklist below).
8. **Open or update the PR** (template below), mark `plan.md` `status:
   done`, and hand off to the verifier per `chayen-sdlc-loop`. Do not request
   human review yet; the verifier does that after a `pass`.
9. **Watch CI.** If CI fails, read the logs, fix the cause, and push. At
   most 2 attempts per failure signature, then `blocked`.

## Change rules

- **Smallest correct diff.** No unrelated refactors, renames, or
  reformatting. Unrelated problems become new `intent.md` files.
- **Mirror the nearest analogous code** for structure, naming, error
  handling, logging, DI, and tests.
- **Keep change types pure.** A bug fix does not refactor; a refactor does
  not change behaviour. If both are needed, make separate commits with the
  refactor first and its tests unchanged.
- **Tests are the specification.** Never edit, skip, loosen, or delete a
  test to make a fix pass. If an existing test encodes behaviour the spec
  changes, update it in a separate commit and name the AC that requires it.
- **Dependencies.** Prefer what the project already uses. A new dependency
  is allowed when it is maintained, has a compatible licence (MIT, Apache-2.0,
  BSD, ISC, or what the repository already accepts), has no known high or
  critical vulnerability, and is declared under `Risk`. Replacing a library
  or framework, or a major upgrade, needs it in the spec.
- **Contracts.** Additive by default. A breaking change to an API, event,
  schema, or package others consume needs it in the spec, plus versioning or
  deprecation.
- **Data.** Migrations use the project's migration tool and follow expand →
  migrate → contract (see `chayen-data`). Never apply to a shared database
  manually; pipelines apply them.
- **Config and flags.** New behaviour that is risky or partial ships behind
  the project's flag mechanism, default off, with the flag named in the PR.
- **Instruction files.** Do not change `AGENTS.md`, `CLAUDE.md`,
  `REVIEW.md`, or skills. If you hit the same mistake twice, note it in the
  PR under `Lessons` for `chayen-context`.

## Change-type notes

| Type | Must do |
|---|---|
| Bug fix | failing test first; root cause in one sentence with `path:line`; fix cause not symptom |
| Feature | AC tests; all UI or API states the spec lists; flag if partial |
| Refactor | behaviour-preserving; tests unchanged and green before and after |
| Upgrade | read the release notes and breaking changes for every major crossed; one major at a time; record the list in `plan.md` |
| Dead code | prove no references (search, reflection, DI registration, config, routes, public API); deprecate before removing anything public |
| Concurrency | name the race; reproduce with a test where possible; fix with the platform's primitive, not sleeps |
| Codegen | regenerate with the project's generator and pinned version; never hand-edit generated files |

## Self-check before the PR

- `git diff --stat` touches only files the plan names, plus tests and the
  artifacts.
- Every AC has a test that ran PASS after the last relevant edit, or an
  explicit reason it cannot be automated.
- No secrets, keys, connection strings, or personal data in code, tests,
  fixtures, logs, or the PR.
- No commented-out code, no debug logging, no TODO without a work-id.
- Generated files regenerated, not edited.
- New dependencies, migrations, auth changes, and contract changes are all
  listed under `Risk`.

## PR template

```markdown
<work-id>: <title>

Spec: docs/sdlc/<work-id>/spec.md   Plan: docs/sdlc/<work-id>/plan.md

## What changed       3–6 bullets, user-visible first
## Acceptance         AC | test | status (PASS/FAIL/…) | verdict
## Verification       command | result | counts
## Risk               migrations, auth, deps, contracts, config, flags — or "none"
## Rollback           revert, flag off, down migration
## Not verified       anything that needs a browser, device, or environment
## Lessons            repeated mistakes for chayen-context (optional)
```

## Rework

- **From the verifier** (`verify.md` verdict `rework`): fix each finding
  marked Important. For a finding you disagree with, reply on the PR with
  evidence; do not ignore it. Push, update the PR's Acceptance table, and
  hand back to the verifier. At most 3 cycles.
- **From the code owner** (gate 2 review comments): treat them like
  verifier findings, then hand to the verifier again so `verify.md` matches
  the final head commit.

## Failure handling

| Situation | Action |
|---|---|
| Baseline already failing | Record it; keep new failures distinguishable; continue if unrelated |
| Tests need a service that is unavailable | Use Testcontainers or the project's fakes; otherwise BLOCKED with the exact command |
| The spec is wrong or incomplete in a way that changes behaviour | `blocked`, with the proposed spec change as a PR comment on the spec; do not guess |
| Scope grows beyond the spec | Stop at the boundary; create a new `intent.md` for the rest |
| A needed credential or environment is missing | BLOCKED, name what is needed; never borrow other credentials |
