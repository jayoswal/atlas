# Repo Spec — `svc-expense` (Expense)

> **Owner:** Senior Backend Engineer · **Stack:** Python 3.12 + FastAPI 0.115 + SQLAlchemy 2 + Alembic · **DB:** `expense_db`. Read [`../ARCHITECTURE.md`](../ARCHITECTURE.md) and the shared skeleton in [`svc-identity.md` §1](./svc-identity.md#1-standard-service-skeleton-shared-by-all-four-services).

- **Role:** authority for *what* people spend and *how* it is reimbursed; owns money, currency, and FX.
- **Owns contracts:** `contracts/openapi/expense.v1.yaml`; event schemas `expense.submitted`.
- **Emits:** `expense.*` on `expense.events`.
- **Consumes (P3/P4):** `employee.*` (Identity),
  `expense.approved|rejected` (Workflow).

## 1. Data model (`expense_db`)

| Table | Key columns |
|---|---|
| `expense_reports` | `id`, `employee_id`, `title`, `status (DRAFT\|PENDING_APPROVAL\|APPROVED\|REJECTED\|REIMBURSED)`, `home_currency`, `total_home_minor`, `created_at` |
| `expense_lines` | `id`, `report_id`, `category_id`, `amount_minor`, `currency`, `amount_home_minor`, `fx_rate`, `receipt_url`, `spent_on` |
| `categories` | `id`, `code`, `name`, `active` |
| `fx_rates` | `rate_date`, `base_currency`, `quote_currency`, `rate` (pk: date+pair) |
| `employee_profiles` | `id`, `email`, `full_name`, `manager_id`, `home_currency`, `status`, `roles`, `updated_at` |

`categories` (Scenario 1), `currency`/`amount_home_minor`/`fx_rate`/`fx_rates` (Scenario 4) are expand-only additions.

## 2. Domain rules

- **Money:** integer minor units + ISO-4217 `currency`. Never floats.
- **FX:** at submit, each line converts `amount_minor@currency → amount_home_minor@home_currency` using `fx_rates` for `spent_on` (fallback: 1:1 when currency == home_currency; else most recent rate ≤ date). `home_currency` comes from `employee_read`.
- **Receipts:** a `receipt_url` string on the line (paste a link). No file upload/object store — that's a prod note in [`../ARCHITECTURE.md §6`](../ARCHITECTURE.md#6-data-management).
- Only `DRAFT` reports are editable; submit locks and publishes.

## 3. REST API (`expense.v1.yaml` excerpt)

| Method & path | Auth | Purpose |
|---|---|---|
| `GET /expense/categories` | any auth | active categories (Scenario 1) |
| `GET /expense/reports` | self or `FINANCE` | list (page params) |
| `POST /expense/reports` | `EMPLOYEE` | create draft |
| `GET/PUT /expense/reports/{id}` | owner; Finance may read | read/update draft |
| `POST /expense/reports/{id}/lines` | owner | add line (category, amount, currency, receipt_url) |
| `POST /expense/reports/{id}/submit` | owner | convert FX, publish `expense.submitted` |
| `GET /expense/profiles/{employee_id}` | self or `HR_ADMIN` | planned P4 provisioning check |

Error codes (prefix `EXPENSE_`): `EXPENSE_OVER_CAP` (informational flag set by Workflow, echoed), `EXPENSE_UNKNOWN_CURRENCY`, `EXPENSE_NOT_DRAFT`, `EXPENSE_REPORT_NOT_FOUND`, `EXPENSE_CATEGORY_INACTIVE`.

## 4. Events

**Emits** (`expense.events`):
- `expense.submitted` (report_id, employee_id, lines[{category_code, amount_minor, currency, amount_home_minor}], total_home_minor) → Workflow.

**Consumes:**
- `employee.created` → create `expense_profile` + `employee_read` (Scenario 5; upsert by id).
- `employee.updated` → upsert read model (incl. `home_currency`).
- `expense.approved` → status `APPROVED` → mark `REIMBURSED` (a real system would run a payout; here it's a state change).
- `expense.rejected` → status `REJECTED`.

## 5. Config, local run, testing, CI

Shape per [`svc-identity.md`](./svc-identity.md) §5–§8. Differences:

```
DATABASE_URL=postgresql+psycopg://atlas:atlas@postgres:5432/expense_db
AMQP_URL=amqp://atlas:atlas@rabbitmq:5672/
JWT_SECRET=atlas-local-development-secret-32
# FX rates are seeded locally; no external provider
# run: uv run uvicorn app.main:app --port 8003 --reload
```

- **FX tests:** conversion is deterministic against a seeded `fx_rates` table; same-currency lines yield `fx_rate=1` and `amount_home_minor==amount_minor`.
- **Contract test:** emitted `expense.submitted` payload validates against the
  `1-2-0` JSON Schema.
- **Migration test:** Scenario 4 backfill sets `currency=home_currency`, `amount_home_minor=amount_minor` for pre-existing rows before the column becomes required.

## 6. P2 implementation status

Conventions + shared template: [`../DEVELOPMENT.md`](../DEVELOPMENT.md).

Implemented in `svc-expense@037d9b8`: contract-backed report CRUD, line entry,
categories, signed-64-bit minor-unit money, deterministic dated FX conversion,
serialized draft mutations, insert-only local fixtures, and persistent
`expense.submitted` publication.

**P2 definition of done:** an employee submits an expense with an optional
receipt link; money is stored as minor units plus currency; FX is deterministic
against seeded rates; the submitted event carries the exact home-currency total.

## 7. P3/P4 implementation status

The P3 `expense.approved`/`expense.rejected` decision consumer
(`svc-expense@d2ce3c8`) finalizes report status idempotently, collapsing
`APPROVED` straight to `REIMBURSED` (no separate payment step exists). P4
(`svc-expense@77a937e`) added the `employee.*` consumer: one
`expense.employee-events` queue bound to `employee.created`/`updated`/
`deactivated` on the `identity.events` exchange, upserting `EmployeeProfile`
by primary key, plus `GET /api/v1/expense/profiles/{employee_id}`
(HR_ADMIN-gated, 404 `EXPENSE_PROFILE_NOT_FOUND` until provisioned) so the
`hrms-web` create-employee wizard can poll provisioning status. Ruff, strict
mypy (23 files), and pytest (29/29) pass.
