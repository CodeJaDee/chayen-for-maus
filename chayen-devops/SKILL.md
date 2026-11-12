---
name: chayen-devops
description: Build, delivery, and operations knowledge for builder, release, and maintain bots - CI/CD in GitHub Actions, Azure Pipelines, or GitLab CI, Dockerfiles, Terraform, Bicep, Kubernetes, and Helm, environments and secrets, artifact promotion, release and rollback checks, observability and SLOs, alerting, incident and postmortem practice, and runbooks. Defines which commands a bot may run by environment. Use with chayen-build, chayen-release, or chayen-maintain.
---

# DevOps

Change pipeline, container, infrastructure, and observability files so that
everything after merge runs through gated pipelines, and classify every
command before running it.

**Operating mode.** Under `chayen-sdlc-loop`: change files and validate them
offline; live environments change only through pipelines; production only
after gate 3. If that skill is not loaded: never run a mutating command
against a shared environment, and never print secrets.

## Command classes

| Class | Examples | Bot may run |
|---|---|---|
| Local | `docker build`, `dotnet build`, `npm ci`, `terraform fmt`/`validate`, `actionlint`, `hadolint`, `helm template`, `bicep build`, `kubeconform` | yes |
| Read-only remote | `terraform plan`, `az deployment group what-if`, `kubectl get`/`describe`/`diff`, `helm diff`, logs, metrics | yes, with credentials granted for that purpose; output without secrets |
| Mutating, non-production | triggering the pipeline's dev or staging deploy | only through the pipeline, after the checks in `chayen-release` |
| Mutating, production | `terraform apply`, `kubectl apply`/`rollout`/`scale`, `helm upgrade`/`rollback`, publish, flag or config change | never directly; the gate 3 pipeline does it |
| Destructive | `terraform destroy`, `-replace` of stateful resources, deleting namespaces, volumes, registries, buckets, databases, or state, force-pushing shared branches, purging queues | never; propose in `release.md` with a verified recovery path |

When a command's class is unclear, treat it as mutating production.

## Pipelines

- **Build once, promote the same artifact** (version or image digest)
  through dev → staging → production.
- Required checks on PRs: build, lint, tests, and the security scans the
  team uses. These are the gate 2 status checks.
- Production deploy job uses the platform's environment protection (GitHub
  Environments with required reviewers, Azure DevOps approvals and checks,
  GitLab protected environments). This is gate 3.
- Pipelines authenticate with workload identity or OIDC federation, not
  long-lived keys. Tokens are scoped per environment; the bots' own tokens
  never reach production.
- Pin actions and images by version or SHA; cache dependencies by lock file;
  fail fast on lint before slow tests.
- Validate pipeline files offline (`actionlint`, the platform's YAML
  validation). A pipeline change is unverified until a branch run passes;
  say so in the PR.

## Containers

Multi-stage builds; small base images pinned by digest or version;
non-root user; no secrets in build args or layers; `.dockerignore` keeps
`.git`, env files, and local artifacts out; healthcheck; one process per
container. Build and start locally to verify.

## Infrastructure as code

Mirror existing modules, naming, and tags. `fmt` and `validate`, then
`plan` or `what-if` with read-only credentials, and put the plan summary in
the PR. A plan that destroys or replaces a stateful resource is Risk high
and needs the spec. Apply only through the pipeline.

## Environments, config, and secrets

- Config differences between environments live in config, not code.
- Secrets come from the platform's store (Key Vault, Secrets Manager,
  GitHub/Azure DevOps secrets) at runtime; never in repository files.
- "Works locally but not in CI" → compare runtime versions, env vars
  (names only), OS, file paths and case, time zone, and network access.

## Observability

New code paths get structured logs with a correlation ID, metrics for rate,
errors, and duration, and traces through the project's OpenTelemetry or APM
setup. SLOs, when the team has them, define the alert thresholds that
`chayen-maintain` uses. Alerts are on symptoms users feel (error rate,
latency, saturation), each with a runbook link.

## Releases and rollback

Every release has a rollback path that is tested or at least rehearsed in
staging: redeploy the previous artifact, flag off, or a down migration.
Database rollbacks follow `chayen-data` (expand/contract makes code rollback
safe).

## Incidents and postmortems

Mitigate first, then find the cause. Bots investigate read-only and
propose mitigations with risk and expected effect; the incident owner
decides. Postmortems are blameless: impact, timeline, root cause with
evidence, contributing factors, action items as intents.

## Runbooks

One page per alert: what it means, how to confirm, safe first actions,
escalation, and links. Commands in runbooks are labelled with their class
from the table above.

## Checks before handing off

| Check | Evidence |
|---|---|
| Pipeline files | linter or validator PASS; branch run result or "unverified until branch run" |
| Containers | local build and start PASS |
| IaC | `fmt`, `validate` PASS; `plan`/`what-if` summary or NOT_RUN without credentials |
| Safety | no command above the allowed class ran |

## Failure handling

| Situation | Action |
|---|---|
| No credentials for plan or what-if | Offline validation only; plan NOT_RUN; exact commands in the PR |
| Plan shows destroy or replace of stateful resources | Stop; Risk high; needs the spec and gate 3 with a recovery path |
| Pipeline cannot be tested without pushing | Push the branch and use its run; report until then as unverified |
| Gates are not enforced in the platform | Report the setup gap to the room and to `chayen-context` |
