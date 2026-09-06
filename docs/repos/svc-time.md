# Repo Spec — `svc-time` (Time & Attendance)

> **Owner:** Senior Backend Engineer · **Stack:** Python 3.12 + FastAPI 0.115 + SQLAlchemy 2 + Alembic · **DB:** `time_db` (Postgres 16). Read [`../ARCHITECTURE.md`](../ARCHITECTURE.md) and the shared service skeleton in [`svc-identity.md` §1](./svc-identity.md#1-standard-service-skeleton-shared-by-all-four-services) — not restated here.

- **Role:** authority for *when* people work and take leave; owns overtime and PTO accrual math.
- **Owns contracts:** `contracts/openapi/time.v1.yaml`; event schemas `timesheet.submitted`.
- **Emits:** `timesheet.*` on `time.events`.
- **Consumes (P3/P4):** `employee.*` (Identity),
  `timesheet.approved|rejected` (Workflow).
- **Read model:** `employee_read` (id, display_name, cost_center, manager_id, status, pto_entitlement_days).

## 1. Data model (`time_db`)

| Table | Key columns |
|---|---|
| `timesheets` | `id`, `employee_id`, `period_start`, `period_end`, `status (DRAFT\|PENDING_APPROVAL\|APPROVED\|REJECTED)`, `overtime_hours (numeric)`, `created_at`, `updated_at` |
| `time_entries` | `id`, `timesheet_id`, `work_date`, `hours`, `project_code`, `note` |
| `employee_read` | `id`, `manager_id`, `status`, `pto_entitlement_days`, `roles`, timestamps |

`pto_requests` and `pto_ledger` are later expand-only additions. Until P4 event
fan-out, the deterministic local seed supplies the employee projection; signed
JWT claims remain authoritative for P2 authorization.

## 2. Domain rules

- **Overtime:** per weekly timesheet, `overtime_hours = max(0, sum(hours) − standard_week)` where `standard_week` is config (default 40). Computed on submit; included in `timesheet.submitted`.
- **PTO balance:** `balance = pto_entitlement_days + Σaccruals − Σtaken − Σpending`. Entitlement comes from the `employee_read` (fed by `employee.*`); Time never calls Identity synchronously for it.
- **Overlap guard:** entries/timesheets for the same employee may not overlap in date range → `TIME_OVERLAPPING_ENTRY`.

## 3. REST API (`time.v1.yaml` excerpt)

| Method & path | Auth | Purpose |
|---|---|---|
| `GET /time/timesheets` | self or `MANAGER` | list (page params) |
| `POST /time/timesheets` | `EMPLOYEE` | create draft |
| `GET/PUT /time/timesheets/{id}` | owner | read/update draft |
| `POST /time/timesheets/{id}/submit` | owner | validate, compute overtime, emit `timesheet.submitted` |
| `GET /time/pto/balance` | self or `MANAGER` | composed PTO balance (Scenario 3) |
| `POST /time/pto/requests` | `EMPLOYEE` | planned PTO workflow |

Error codes (prefix `TIME_`): `TIME_OVERLAPPING_ENTRY`, `TIME_NOT_DRAFT`, `TIME_TIMESHEET_NOT_FOUND`, `TIME_INSUFFICIENT_PTO`.

## 4. Events

**Emits** (`time.events`): `timesheet.submitted` (id, employee_id, period, total_hours, overtime_hours) → consumed by Workflow.

**Consumes:**
- `employee.created` → insert `employee_read` + seed initial PTO accrual (idempotent on `event_id`).
- `employee.updated` → upsert `employee_read`.
- `employee.deactivated` → mark read model inactive; block new submissions.
- `timesheet.approved|rejected` (from Workflow) → set timesheet `status`.

## 5. Config, local run, testing, CI

Identical shape to [`svc-identity.md`](./svc-identity.md) §5–§8. Differences:

```
DATABASE_URL=postgresql+psycopg://atlas:atlas@postgres:5432/time_db
AMQP_URL=amqp://atlas:atlas@rabbitmq:5672/
STANDARD_WEEK_HOURS=40
# run: uv run uvicorn app.main:app --port 8002 --reload
```

- **Contract test:** emitted `timesheet.submitted` validates against its JSON Schema; a replayed `employee.created` is a no-op (upsert by `employee_id`).
- **Scenario 3 test:** balance endpoint returns entitlement-based number even when `employee_read.pto_entitlement_days` is null (defaults 0) — proves tolerance of not-yet-arrived events.

## 6. P2 implementation status

Conventions + shared template: [`../DEVELOPMENT.md`](../DEVELOPMENT.md).

Implemented in `svc-time@0189c95`: contract-backed timesheet CRUD, seven-entry
and overlap guards, serialized draft mutations, configurable overtime,
PTO-balance composition, deterministic insert-only projection seed, and
persistent `timesheet.submitted` publication.

**P2 definition of done:** an employee submits a timesheet; overtime is
computed and emitted; PTO balance remains resilient to a missing projection.

## 7. P3/P4 implementation status

The P3 `timesheet.approved`/`timesheet.rejected` decision consumer
(`svc-time@277be72`) finalizes timesheet status idempotently. P4
(`svc-time@2776761`) added the `employee.*` consumer: one
`time.employee-events` queue bound to `employee.created`/`updated`/
`deactivated` on the `identity.events` exchange, upserting the `employee_read`
projection by primary key, plus `GET /api/v1/time/profiles/{employee_id}`
(HR_ADMIN-gated, 404 `TIME_PROFILE_NOT_FOUND` until provisioned) so the
`hrms-web` create-employee wizard can poll provisioning status. Ruff, strict
mypy (20 files), and pytest (36/36) pass.
