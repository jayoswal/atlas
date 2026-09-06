# Atlas HRMS — Trackable Implementation Plan

> Execution plan for turning the specifications into the six independently
> versioned repositories tracked in [`REPO_TRACKER.md`](./REPO_TRACKER.md).
> Update checkboxes and evidence links in the same pull request as completed work.

## Rules and measures

- **Runtime target:** Ubuntu (WSL for local development; Ubuntu VM for deployment).
- **Source layout:** six sibling repositories under `/home/oswa/atlas-repos`.
- **Coordination:** contract PR first; expand-only migrations; provider/producer
  before existing consumers; UI last. New-event consumers bind before producers.
- **Quality gate per repo:** format/lint, strict types, unit/contract tests, build.
- **Phase completion:** every exit criterion must be checked and demonstrated by
  `platform-outerloop/scripts/smoke.py`.
- **Git identity:** `jayoswal <jayumeshoswal2001@gmail.com>` only.

## Bootstrap — local repository and toolchain setup

**Repositories:** all six code repos plus the `atlas` documentation repo.

- [x] Create `/home/oswa/atlas-repos`.
- [x] Initialize all six local repositories on `main`.
- [x] Install user-space Python package tooling (`pip`, `uv`).
- [x] Install native Linux Node 20 and npm through nvm.
- [x] Generate dependency lockfiles for all projects.
- [x] Install/enable Docker and Compose (manual sudo step; see
  [`REPO_TRACKER.md`](./REPO_TRACKER.md#manual-host-prerequisite)).
- [x] Create six public GitHub repositories, add `origin`, and push `main`.

**Exit evidence:** `git status` works in all seven repositories; `python3`,
`python3 -m pip`, `uv`, `node`, and `npm` report expected versions; Docker
reports both engine and Compose v2 versions.

## P0 — Platform skeleton

**Repositories:** `platform-outerloop`, plus build contexts for all five app repos.

- [x] Add Postgres 16 with four databases.
- [x] Add RabbitMQ management, MailHog, Adminer, and Traefik.
- [x] Add all four service and web build definitions/routes.
- [x] Seed the OpenAPI/AsyncAPI contract registry.
- [x] Add service-template conventions and seed/smoke entrypoints.
- [x] Scaffold buildable FastAPI and React applications.
- [x] Run `docker compose config`.
- [ ] Run `docker compose up --build -d` after Docker is available.
- [ ] Confirm Postgres, RabbitMQ, MailHog, Adminer, and gateway health.

**Exit criterion:** one command starts all infrastructure and five applications;
the four databases exist; each local browser tool is reachable.

## P1 — Identity and authentication

**Repositories:** `platform-outerloop`, `svc-identity`, `hrms-web`.

- [ ] Finalize `identity.v1.yaml` login, employee, and role contracts.
- [ ] Add employee/cost-center schema and expand-only migration.
- [ ] Implement Argon2 login and eight-hour HS256 token issuance.
- [ ] Implement shared error envelope, correlation ID, RBAC, and ownership checks.
- [ ] Seed demo users/roles and add identity contract/API tests.
- [ ] Generate the web API client; add login and protected app shell.
- [ ] Extend smoke test: login then authorized request.

**Exit criterion:** demo login returns a valid token and one protected request
succeeds through Traefik; lint, strict types, tests, and builds pass.

## P2 — Core time and expense domains

**Repositories:** `platform-outerloop`, `svc-time`, `svc-expense`, `hrms-web`.

- [ ] Merge time/expense OpenAPI and submitted-event schemas first.
- [ ] Add expand-only models/migrations for timesheets, entries, expense reports,
  line items, categories, and employee read models.
- [ ] Implement create/list/update/submit APIs and validation rules.
- [ ] Publish `timesheet.submitted` and `expense.submitted` after commit.
- [ ] Add employee timesheet and expense screens using generated clients.
- [ ] Add unit, API, contract, and UI tests.
- [ ] Extend seed/smoke to submit one timesheet and expense.

**Exit criterion:** an employee submits both records through the UI/API and both
events are visible in RabbitMQ.

## P3 — Workflow, approvals, and notifications

**Repositories:** `platform-outerloop`, `svc-workflow`, `svc-time`,
`svc-expense`, `hrms-web`.

- [ ] Merge workflow API and decision-event contracts first.
- [ ] Add approval, policy, employee-read, and notification migrations.
- [ ] Consume submitted events idempotently and create approval tasks.
- [ ] Implement approval list/decision APIs with manager ownership checks.
- [ ] Publish decisions; consume them in time/expense to finalize state.
- [ ] Send decision email through MailHog.
- [ ] Add manager approvals queue and employee status badges.
- [ ] Extend smoke test for submit → decide → finalize → email.

**Exit criterion:** a manager decision completes the event round trip and an
email is visible in MailHog.

## P4 — Employee event fan-out

**Repositories:** all six.

- [ ] Merge `employee.created/updated/deactivated` schemas first.
- [ ] Deploy/bind idempotent consumers in time, expense, and workflow.
- [ ] Publish identity events only after consumers are ready.
- [ ] Auto-create time/expense profiles, accruals, and approval chain.
- [ ] Add create-employee wizard and provisioning status.
- [ ] Extend smoke test for one new hire across all read models.

**Exit criterion:** creating one employee automatically creates all three
downstream profiles exactly once, including after redelivery.

## P5 — Cross-repository scenarios

**Repositories:** scenario-specific sets in [`SCENARIOS.md`](./SCENARIOS.md).

- [ ] S1 — Expense category plus policy cap.
- [ ] S2 — Overtime threshold approval.
- [ ] S3 — PTO balance widget.
- [ ] S4 — Multi-currency expenses and FX conversion.
- [ ] S5 — New-hire provisioning verified as a learning exercise.
- [ ] Record contract delta, compatibility proof, deploy order, automated test,
  and UI screenshot for every scenario.

**Exit criterion:** each scenario's end-to-end smoke test passes independently;
all six repository quality gates pass from clean checkouts.

## Progress reporting

For each completed checkbox, record the commit/PR next to the item. At each
phase boundary, update [`REPO_TRACKER.md`](./REPO_TRACKER.md), tag independent
SemVer releases, and save the smoke-test output as CI evidence.
