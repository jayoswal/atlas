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
- **Reproducibility gate:** application runtimes, direct dependencies, container
  images, and CI actions are immutable/exactly pinned before feature work;
  committed lockfiles pin transitive dependencies.
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
- [x] Install/enable Docker and Compose; verify non-root daemon access (see
  [`REPO_TRACKER.md`](./REPO_TRACKER.md#docker-status)).
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
- [x] Run `docker compose up --build -d`.
- [x] Confirm Postgres, RabbitMQ, MailHog, Adminer, gateway, web, and all four
  backend services are healthy.

**Exit criterion:** one command starts all infrastructure and five applications;
the four databases exist; each local browser tool is reachable.

**Completed 2026-09-06:** `seed.py` and `smoke.py` pass; all browser/API
endpoints return HTTP 200; Postgres contains `identity_db`, `time_db`,
`expense_db`, and `workflow_db`. Container-startup fixes are recorded in
`platform-outerloop@ef88b35`, `hrms-web@144e44c`, `svc-identity@cec7110`,
`svc-time@f924307`, `svc-expense@10f19e3`, and `svc-workflow@53bbcf2`.

## P0.1 — Reproducible dependency baseline

**Repositories:** all six code repositories and `atlas`.

This gate pins what controls application output while avoiding pins that make
security maintenance or host portability worse.

- [x] Add an exact `.python-version` to the four services, platform tools, and
  service template; keep `requires-python` as the compatible 3.12 constraint.
- [x] Add an exact `.nvmrc` to `hrms-web`; keep `engines.node` as the compatible
  Node 20 constraint.
- [x] Replace direct Python wildcard/range requirements with the exact versions
  already proven by each `uv.lock`; retain and commit `uv.lock`.
- [x] Replace npm caret/tilde direct requirements with exact versions already
  proven by `package-lock.json`; retain and commit `package-lock.json`.
- [x] Pin every Docker/Compose image and Dockerfile base image to an immutable
  digest, retaining a readable version tag beside the digest.
- [x] Pin GitHub Actions to full commit SHAs when workflows are introduced,
  with the release version in a comment.
- [x] Add `docs/DEPENDENCY_BASELINE.md`: component, exact version/digest,
  source file, update command, last verification date, and reason for any
  exception.
- [x] Add a CI check that rejects mutable image tags (`latest`, major-only tags),
  missing lockfiles, non-exact direct application dependencies, and unpinned
  Actions.
- [x] Rebuild from clean caches and run every repository quality gate plus the
  full Compose smoke test.

**Intentional exceptions:**

- Host-installed Git, Docker Engine, Compose, `uv`, and OS packages use
  documented minimum-supported versions rather than package-level pins. Exact
  tested versions are recorded in [`REPO_TRACKER.md`](./REPO_TRACKER.md), while
  Ubuntu security updates remain installable.
- Language compatibility declarations (`requires-python`, `engines.node`) stay
  ranges; `.python-version`, `.nvmrc`, containers, and lockfiles provide the
  exact reproducible selections.
- Project SemVer, API versions, and event-schema versions describe compatibility
  and are not dependency pins.
- Secret/config values and generated timestamps are never treated as versions.

**Update procedure:** dependency changes are deliberate PRs that update the
manifest, lockfile/digest, baseline document, and relevant generated/template
copies together. Dependabot/Renovate may propose updates, but never auto-merge;
the complete quality gate and Compose smoke test must pass.

**Exit criterion:** a clean Ubuntu machine resolves the same language runtimes,
dependency graph, container image digests, and CI actions with no mutable
application dependency inputs.

**Completed 2026-09-06:** backend Ruff/mypy/pytest, frontend
ESLint/Vitest/TypeScript/Vite, rendered-template checks, a no-cache image build,
all ten containers, and the full seed/smoke flow pass. Cross-repository pin
policy workflow `34035100310` passes. Evidence commits:
`platform-outerloop@777ef77`, `hrms-web@896f05c`,
`svc-identity@5bcc517`, `svc-time@6cca0f0`, `svc-expense@eec0ca2`, and
`svc-workflow@6a8d9f8`.

## P1 — Identity and authentication

**Repositories:** `platform-outerloop`, `svc-identity`, `hrms-web`.

- [x] Finalize `identity.v1.yaml` login, employee, and role contracts.
- [x] Add employee/cost-center schema and expand-only migration.
- [x] Implement Argon2 login and eight-hour HS256 token issuance.
- [x] Implement shared error envelope, correlation ID, RBAC, and ownership checks.
- [x] Seed demo users/roles and add identity contract/API tests.
- [x] Generate the web API client; add login and protected app shell.
- [x] Extend smoke test: login then authorized request.

**Exit criterion:** demo login returns a valid token and one protected request
succeeds through Traefik; lint, strict types, tests, and builds pass.

**Completed 2026-09-06:** identity migration and idempotent demo seed pass
against PostgreSQL; login, `/identity/me`, HR directory access, SPA fallback,
and API 404 behavior pass through Traefik. Backend Ruff, strict mypy, and 15
API/auth tests pass; frontend ESLint, three UI tests, generated-client drift,
TypeScript/Vite build, and production dependency audit pass. Clean no-cache
identity/web image builds and all ten running containers were verified.
Evidence commits: `platform-outerloop@8fffead`, `svc-identity@da3ed15`,
`hrms-web@84fc7c0`, `svc-time@ff0e0de`, `svc-expense@178cac9`, and
`svc-workflow@b3d813b`.

## P2 — Core time and expense domains

**Repositories:** `platform-outerloop`, `svc-time`, `svc-expense`, `hrms-web`.

- [x] Merge time/expense OpenAPI and submitted-event schemas first.
- [x] Add expand-only models/migrations for timesheets, entries, expense reports,
  line items, categories, and employee read models.
- [x] Implement create/list/update/submit APIs and validation rules.
- [x] Publish `timesheet.submitted` and `expense.submitted` after commit.
- [x] Add employee timesheet and expense screens using generated clients.
- [x] Add unit, API, contract, and UI tests.
- [x] Extend seed/smoke to submit one timesheet and expense.

**Exit criterion:** an employee submits both records through the UI/API and both
events are visible in RabbitMQ.

**Completed 2026-09-06:** Time and Expense contracts, migrations, APIs,
deterministic insert-only seeds, generated clients, and employee workspaces are
implemented. The integration smoke creates and submits a 50-hour timesheet
(10 overtime hours) and a multi-currency expense, then observes both persistent
submitted events through a temporary RabbitMQ queue. Time Ruff, strict mypy,
and 25 tests pass; Expense Ruff, strict mypy, and 18 tests pass; web ESLint,
six UI tests, generated-client drift, TypeScript/Vite build, and production
dependency audit pass. Clean no-cache application builds, repeatable seed/smoke,
all ten running containers, and both SPA routes were verified. Evidence commits:
`platform-outerloop@47d11a8`, `svc-time@0189c95`,
`svc-expense@037d9b8`, `hrms-web@11dcc11`,
`svc-identity@141b9a6`, and `svc-workflow@666833c`.

## P3 — Workflow, approvals, and notifications

**Repositories:** `platform-outerloop`, `svc-workflow`, `svc-time`,
`svc-expense`, `hrms-web`.

- [x] Merge workflow API and decision-event contracts first.
- [x] Add approval, policy, employee-read, and notification migrations.
- [x] Consume submitted events idempotently and create approval tasks.
- [x] Implement approval list/decision APIs with manager ownership checks.
- [x] Publish decisions; consume them in time/expense to finalize state.
- [x] Send decision email through MailHog.
- [x] Add manager approvals queue and employee status badges.
- [x] Extend smoke test for submit → decide → finalize → email.

**Exit criterion:** a manager decision completes the event round trip and an
email is visible in MailHog.

**Completed 2026-09-13:** Workflow contracts (`workflow.v1.yaml`, decision
event schemas), `approvals`/`policies`/`notifications`/`employee_read`
migrations, the policy engine (`PER_RECEIPT_CAP`, `OVERTIME_THRESHOLD`),
direct-manager routing, the approvals API, and MailHog notifications are
implemented (see [`repos/svc-workflow.md` §8](./repos/svc-workflow.md#8-implementation-notes-as-built-p3)
for as-built decisions, including the corrected `overtime_threshold_hours=5`
default and the MailHog Subject-folding workaround). Time and Expense consume
`timesheet.approved|rejected` / `expense.approved|rejected` idempotently and
finalize state (Expense collapses `APPROVED` to `REIMBURSED`). The web app
gained a manager Approvals queue (`/approvals`, gated on MANAGER/FINANCE) with
inline decisions and policy-flag warnings. svc-workflow Ruff, strict mypy, and
27 tests pass; svc-time Ruff, strict mypy, and 28 tests pass; svc-expense
Ruff, strict mypy, and 21 tests pass; web ESLint, TypeScript, Vite build, and
7 UI tests pass. The extended smoke test (submit timesheet/expense → route to
manager → approve timesheet [flagged `OVERTIME_THRESHOLD`] → reject expense
[flagged `PER_RECEIPT_CAP`] → verify decision events on RabbitMQ → verify
Time/Expense finalize → verify a MailHog approval-request email) passed twice
consecutively, and again after clean-cache rebuilds of `svc-workflow`,
`svc-time`, `svc-expense`, and `hrms-web`. Evidence commits:
`platform-outerloop@7ba12cc`, `svc-workflow@8dca804`,
`svc-time@277be72`, `svc-expense@d2ce3c8`, and `hrms-web@d5062a5`.

## P4 — Employee event fan-out

**Repositories:** all six.

- [x] Merge `employee.created/updated/deactivated` schemas first. (The
  AsyncAPI channels/operations/messages and the three JSON Schemas were
  already merged in an earlier phase; only the two OpenAPI contracts needed
  a new `/profiles/{employee_id}` path this phase — see below.)
- [x] Deploy/bind idempotent consumers in time, expense, and workflow. Each
  service binds one queue (`time.employee-events`, `expense.employee-events`,
  `workflow.employee-events`) to all three `employee.*` routing keys on the
  `identity.events` exchange, dispatching by `event["type"]`; consumers
  upsert their read-model **by primary key** (`db.get(Model, employee_id,
  with_for_update=True)`), which is naturally idempotent — no separate
  dedup table needed, unlike the P3 approval-decision idempotency pattern.
- [x] Publish identity events only after consumers are ready. (Conceptual
  deploy-order note only: RabbitMQ's durable queues buffer messages, and the
  current `docker compose up` starts every service together, so this is a
  documentation guarantee rather than an operational requirement locally.)
- [x] Auto-create time/expense profiles, accruals, and approval chain. Time
  and Expense each expose a new `GET /api/v1/{time,expense}/profiles/{id}`
  read endpoint (HR_ADMIN-gated) so the wizard can poll provisioning status;
  Workflow only needs its `employee_read` projection current for approval
  routing/notification, so it exposes no new endpoint this phase.
- [x] Add create-employee wizard and provisioning status. `hrms-web` gained
  `/employees` (HR_ADMIN-gated, replacing the disabled P3 nav placeholder):
  a create-employee form plus a status panel that polls the Time and Expense
  profile endpoints until both resolve.
- [x] Extend smoke test for one new hire across all read models.
  `scripts/smoke.py` now logs in as `admin@atlas.dev` (HR_ADMIN), creates one
  employee, waits for the Time and Expense profiles to appear, waits for the
  Workflow `employee_read` row to sync, re-reads the Time profile to confirm
  the upsert-by-PK consumer is idempotent, then deactivates the employee and
  confirms `employee_read.status` follows to `INACTIVE`.

**Exit criterion:** creating one employee automatically creates all three
downstream profiles exactly once, including after redelivery. **Met.** All
four backend services (Ruff, strict mypy, pytest) and the web app (ESLint,
`tsc -b`, Vitest, Vite build) pass; the extended smoke test above passed
twice consecutively, and again after a clean-cache rebuild of
`svc-identity`, `svc-time`, `svc-expense`, `svc-workflow`, and `hrms-web`.
Evidence commits: `platform-outerloop@68bf951`, `svc-identity@14648e8`,
`svc-time@2776761`, `svc-expense@77a937e`, `svc-workflow@1671a82`, and
`hrms-web@b81a195`.

## UI/UX enterprise revamp (cosmetic-only, no functional change)

**Repository:** `hrms-web` — `/home/oswa/atlas-repos/hrms-web`.

- [x] Enrich `theme.ts` (palette, typography scale, shadows, component
  overrides) and add `@mui/icons-material` (exact-pinned) for iconography.
- [x] Add shared presentational components: `PageHeader`, `StatCard`,
  `StaticCharts` (`StaticBarChart`, `Sparkline`), `ActivityFeed`.
- [x] Redesign `AppShell`: richer AppBar (logo, dummy search, notifications
  menu, avatar menu with real sign-out) and a dark iconified Drawer nav,
  grouped into "Workspace" (real items, gating unchanged) and "Organization"
  (disabled, sample-only items — for show, not interactive).
- [x] Add a `DashboardPage` with KPI stat cards, a static bar chart, and a
  static activity feed. Only real data used: `useCurrentUserQuery` (Welcome
  heading, PTO balance); all other KPI numbers are clearly-labeled sample
  data, not backed by new API calls.
- [x] Polish Time/Expense/Approvals headers with `PageHeader`; replace PTO
  balance card with `StatCard`.
- [x] Polish `LoginPage` with an enterprise split-screen branding panel
  (sign-in form/logic unchanged).
- [x] Polish `CreateEmployeePage` with `PageHeader` and a cosmetic `Stepper`
  (form/provisioning logic unchanged).

**Exit criterion:** purely visual/cosmetic change — no routes, gating logic,
API calls, or feature behavior were added, removed, or altered. **Met.**
`hrms-web` ESLint, `tsc -b`, Vitest (8/8), Vite production build, and the
repo-wide dependency pin-policy check (`platform-outerloop/scripts/check_pins.py`)
all pass. Evidence commit: `hrms-web@5163afc`.

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
