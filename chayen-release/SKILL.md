---
name: chayen-release
description: Deploy stage of the automated SDLC. After a PR merges, follows the pipeline build, promotes the same artifact to dev and staging through the pipeline, runs smoke and post-deploy checks, writes release.md with version, changes, migrations, risk, release notes, and rollback, and requests production authorization at human gate 3. Never deploys to production itself and holds no production credentials. Use with chayen-sdlc-loop and chayen-devops.
---

# Release

Take merged work to the production gate with everything the release
manager needs to say yes or no in minutes. The agent may act up to the
production gate and cannot pass it.

Follow `chayen-sdlc-loop`. Load `chayen-devops` for pipelines and
`chayen-data` when the release carries migrations.

## Start conditions

- A build PR for `<work-id>` merged into the default branch, or a release
  train schedule fires and several merged items lack `release.md`.
- The repository has a deploy pipeline. If it has none, write `release.md`
  with the manual steps, mark it `blocked`, and post the setup gap.

## Workflow

1. **Collect the release set.** Merged items since the last production
   release (from tags or the deploy history), each with its work-id, spec,
   PR, and `verify.md` verdict. An item without a `pass` verdict on its
   merged SHA is excluded and reported.
2. **Follow the build.** Wait for the pipeline run on the merge commit.
   Record the artifact identity: version, image digest, or package hash.
   Build once; every environment gets this same artifact.
3. **Dev.** Let the pipeline deploy (or trigger the dev deploy job if the
   pipeline requires a manual trigger and your token is scoped to dev).
   Run smoke checks: health endpoint, one read path, one write path in test
   data, the new feature's main AC if it can be exercised by API. Record
   results.
4. **Staging.** Promote the same artifact through the pipeline once dev is
   green. Run the smoke set plus the release's end-to-end tests if the
   project has them. Watch error rate and latency for the observation
   window the team set (default 15 minutes). Pipeline-run migrations are
   expected here; manual database commands are not allowed.
5. **Write `release.md`** (one per work item, or one per version for a
   train) and store it as the platform's release record: the release or tag
   description, or the deploy run summary, linked from the gate 3 approval.
   Do not open a PR just to store it; that adds a human step. If the
   repository wants release records in git, add them to the next build PR
   under `docs/sdlc/releases/<version>.md`:

   ```markdown
   ---
   id: <work-id or version>
   stage: release
   status: prepared | authorized | released | rolled_back | blocked
   artifact: <version and digest>
   updated: <YYYY-MM-DD>
   bot: <bot name>
   ---
   ## Summary            what ships, for whom, in three lines
   ## Items              work-id | title | PR | verify verdict
   ## Release notes      user-facing, plain language, no internals
   ## Migrations         name | expand/contract phase | lock impact | reversible?
   ## Flags              name | default | rollout plan
   ## Evidence           dev and staging checks with status and timestamps
   ## Risk               blast radius, data impact, known issues
   ## Rollback           exact pipeline action, expected time, data caveats
   ## Watch after release   metrics and thresholds, observation window
   ```

6. **Request gate 3.** Open the production deployment approval in the
   platform (GitHub Environments required reviewers, Azure DevOps
   approvals, GitLab protected environments) pointing at this artifact.
   Post one line in the room with the link to `release.md`. Then stop.
7. **After authorization** the pipeline deploys. Watch the post-release
   window with read-only access: error rate, latency, saturation, and the
   metrics listed under Watch. Update `release.md` to `released` with the
   observed values.
8. **If a threshold is breached** during the watch window: post in the room
   immediately with the evidence, recommend rollback or flag-off, and hand
   off to `chayen-maintain`. Execute nothing in production unless the
   pipeline has an automatic, pre-approved rollback; then record what it did.

## Rules

- **No production credentials, no production commands.** Production
  changes happen only through the gate 3 pipeline run.
- **Same artifact everywhere.** Never rebuild for staging or production.
- **Migrations ship expand-first.** A contract step (drop, rename, narrow)
  ships in a later release after the code that no longer needs the old
  shape is in production. Flag any contract step in Risk.
- **Release notes are facts.** Each line traces to a merged PR. No
  marketing, no internals, no promises.
- **Hotfix** follows the same path, faster: the combined gate for the fix,
  then dev and staging smoke only, then gate 3. Never skip staging.
- **Idempotent.** Check the deploy history before triggering anything; a
  restarted run must not deploy twice.

## Failure handling

| Situation | Action |
|---|---|
| Pipeline fails on build | Hand to the builder bot with the log excerpt and the commit; status `blocked` until fixed |
| Dev or staging smoke fails | Do not request gate 3. Create rework for the responsible work-id with evidence; exclude it from a train if others can ship |
| Migration takes locks longer than the window | Mark Risk high, propose a batched or online approach in `release.md`, keep gate 3 closed |
| Gate 3 not answered | Do not re-ping more than once per working day; the artifact waits |
| Production metrics degrade after release | Evidence to the room, recommend rollback, hand to `chayen-maintain` |
