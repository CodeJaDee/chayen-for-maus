---
name: chayen-intent-spec
description: Plan and Design stages of the automated SDLC. Turns an idea, ticket, bug report, incident, or scan finding into intent.md, then into a reviewable spec.md with testable acceptance criteria, design, risks, assumptions, and open decisions, and opens the spec PR for human gate 1. Also reworks a spec from PR review comments. Use with chayen-sdlc-loop. Not for writing production code.
---

# Intent and spec (Plan + Design)

Capture what is wanted once, as `intent.md`, then produce a `spec.md` that a
builder bot can implement without asking anything, and that a person can
accept or reject in one review.

Follow `chayen-sdlc-loop` for artifacts, gates, autonomy, and handoff.

## Inputs

- **Entry points:** a message in the room, a ticket or issue, an incident or
  alert summary, a scan finding, or an `intent.md` written by
  `chayen-maintain`.
- **Read before writing:** `AGENTS.md` or `CLAUDE.md`, existing
  `docs/sdlc/*/spec.md` for the same area, ADRs (`docs/adr/`,
  `docs/architecture/`), the code the change touches, and its tests.

## Workflow

1. **Pick the work-id** and check for existing work: an open spec PR or a
   `docs/sdlc/<work-id>/` folder. Resume it if one exists.
2. **Write `intent.md`** (no gate). Keep it to one screen:

   ```markdown
   ---
   id: <work-id>
   stage: intent
   status: proposed
   source: [<ticket, message, or alert link>]
   updated: <YYYY-MM-DD>
   bot: <bot name>
   ---
   # <Outcome in one line>
   ## Problem        who is affected, what happens now, evidence
   ## Outcome        what is true when done, measurable
   ## Users and systems
   ## Constraints    compatibility, deadlines, platforms, compliance as stated
   ## Out of scope
   ## Open questions
   ```

   For incidents, add `## Anomaly` and `## Evidence` (metric, threshold,
   window, log or trace links). Never paste personal data.
3. **Investigate the code** for the area: current behaviour with
   `path:line`, the closest analogous feature, data model, contracts, tests,
   and anything an accepted ADR fixes in place. Extract business rules
   implemented in code with their citations.
4. **Classify the size and risk.**

   | Class | Signals | Spec depth |
   |---|---|---|
   | Fix | defect, reproducible, local blast radius | short: criteria + reproduction; may use the combined gate |
   | Feature | new behaviour in existing architecture | full spec |
   | Structural | new service or module, data model change, auth, money, public contract, migration between platforms | full spec + design options + ADR draft (load `chayen-architecture`) |

5. **Write `spec.md`.** Use this structure, dropping empty sections:

   ```markdown
   ---
   id: <work-id>
   stage: spec
   status: proposed
   class: fix | feature | structural
   source: [intent.md, ...]
   updated: <YYYY-MM-DD>
   bot: <bot name>
   ---
   # <title>
   ## Summary              three lines a reviewer reads first
   ## Requirements         numbered R1..Rn, each one behaviour
   ## Acceptance criteria  AC1..ACn, Given/When/Then, each names its check
   ## Edge cases           boundaries, invalid input, concurrency, permissions, empty and error states
   ## Non-functional       only measurable targets (p95 latency, error rate, a11y level)
   ## Design               components touched, contracts, data changes, sequence (Mermaid)
   ## Options considered   structural only: options, trade-offs, chosen, rejected and why
   ## Risks                migrations, auth, dependencies, breaking changes, rollout
   ## Rollout              flag, migration order (expand → migrate → contract), rollback
   ## Assumptions          each: statement, impact if wrong, how to reverse
   ## Open decisions       each: question, options, recommended default, owner
   ## Out of scope
   ## Traceability         R → AC → planned test names
   ```

6. **Self-review before opening the PR** (checks below). Fix failures.
7. **Open the spec PR** from `sdlc/<work-id>/spec` containing `intent.md`
   and `spec.md`. PR body: the Summary, the Open decisions as a checklist
   with the recommended default for each, Risks, and "Merging this PR
   accepts the spec. Unchecked open decisions take the recommended
   default." Request review from the product owner or CODEOWNERS of
   `docs/sdlc/`. This is gate 1.
8. **Hand off** per `chayen-sdlc-loop`: post the PR link and stop. Building
   starts only after the merge.

## Writing rules

- **Testable.** "Fast", "user-friendly", "secure", and "works correctly" are
  not criteria. Each AC is observable and names how it is checked: unit,
  integration, API, end-to-end, or a measured value.
- **Never invent business rules.** A rule is a fact only when a source
  states it or code at a cited line implements it. Otherwise it is an
  assumption with a reversible default, or an open decision.
- **Decide what you can.** Technical choices inside the existing
  architecture and conventions (naming, file layout, validation style,
  status codes that follow the project's pattern) are decided in the spec,
  not left open.
- **Small slices.** If the intent cannot ship in one PR of reasonable size,
  split it into ordered work items, each with its own spec, and say which
  ships first. Each slice must be releasable on its own or behind a flag.
- **Contradictions are listed**, with both sources, as an open decision.
  Never resolve them silently.
- **Design follows accepted decisions.** An ADR in force is followed; a
  spec that needs to change one includes a superseding ADR draft as an open
  decision.

## Rework on review

When the spec PR has review comments or "changes requested":

1. Read every comment. Answer each one in the PR thread with what changed,
   or why not.
2. Update `spec.md`, keeping the same work-id and branch. Record resolved
   open decisions under `## Decisions` with who decided and the PR comment
   link.
3. Re-request review. If the PR is closed unmerged, stop; do not reopen it.

## Self-review checklist

- Every requirement has at least one AC, and every AC traces to a
  requirement.
- Every AC names its check type and could fail.
- Edge cases cover empty, invalid, unauthorized, concurrent, and failure
  paths where they apply.
- Every claim about current behaviour cites `path:line`.
- Assumptions have impact and reversal; open decisions have options, a
  recommended default, and an owner (or "owner unknown").
- Structural changes include options with at least one rejected
  alternative and its reason.
- No secrets, no personal data, no code changes outside `docs/sdlc/`.

## Failure handling

| Situation | Action |
|---|---|
| Intent is too vague to state any outcome | Write `intent.md` with what is known, put the missing outcome as the first open decision with two or three concrete options, and still open the spec PR. The reviewer chooses. |
| The request is a solution ("add a dropdown") | Record it, infer the underlying goal as an assumption, and spec the goal with the requested solution as the default option. |
| Code and documents disagree | Spec the code's current behaviour as found, and the change as an open decision. |
| The work duplicates an open item | Link the existing work-id in the room and stop. |
| Legal, regulatory, or pricing meaning is needed | Record as stated by the source, mark "needs qualified review" as an open decision. Never interpret law. |
