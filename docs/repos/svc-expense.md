# Repo Spec — `svc-expense` (Expense)

> **Owner:** Senior Backend Engineer · **Stack:** Python 3.12 + FastAPI 0.115 + SQLAlchemy 2 + Alembic · **DB:** `expense_db`. Read [`../ARCHITECTURE.md`](../ARCHITECTURE.md) and the shared skeleton in [`svc-identity.md` §1](./svc-identity.md#1-standard-service-skeleton-shared-by-all-four-services).

- **Role:** authority for *what* people spend and *how* it is reimbursed; owns money, currency, and FX.
- **Owns contracts:** `contracts/openapi/expense.v1.yaml`; event schemas `expense.submitted`.
- **Emits:** `expense.*` on `expense.events`.
- **Consumes:** `employee.*` (Identity), `expense.approved|rejected` (Workflow).

## 1. Data model (`expense_db`)

| Table | Key columns |
|---|---|
| `expense_reports` | `id`, `employee_id`, `title`, `status (DRAFT\|PENDING_APPROVAL\|APPROVED\|REJECTED\|REIMBURSED)`, `home_currency`, `total_home_minor`, `created_at` |
| `expense_lines` | `id`, `report_id`, `category_id`, `amount_minor`, `currency`, `amount_home_minor`, `fx_rate`, `receipt_url`, `spent_on` |
| `categories` | `id`, `code`, `name`, `active` |
| `fx_rates` | `rate_date`, `base_currency`, `quote_currency`, `rate` (pk: date+pair) |
| `expense_profiles` | `employee_id`, `default_currency`, `category_access (jsonb)` |
| `employee_read` | id, display_name, cost_center, manager_id, status, home_currency — upserted from `employee.*` |

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
| `POST /expense/reports/{id}/lines` | owner | add line (category, amount, currency, receipt_url) |
| `POST /expense/reports/{id}/submit` | owner | convert FX, publish `expense.submitted` |
| `GET /expense/profiles/{employee_id}` | self or `HR_ADMIN` | provisioning-ready check (Scenario 5) |

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
- **Contract tests:** `expense.submitted` payload validates across `1-0-0 → 1-1-0 → 1-2-0` (additive) so Workflow's older consumers still parse.
- **Migration test:** Scenario 4 backfill sets `currency=home_currency`, `amount_home_minor=amount_minor` for pre-existing rows before the column becomes required.

## 6. Implementation build order (Phase P2)

Conventions + shared template: [`../DEVELOPMENT.md`](../DEVELOPMENT.md).

1. Instantiate the service template; add the `employee.*` consumer → `employee_read` + `expense_profile` (upsert).
2. Author `expense.v1.yaml` + `expense.submitted` schema → merge.
3. Models/migrations: `expense_reports`, `expense_lines`, `expense_profiles`. (Add `categories`, `fx_rates`, currency columns in the S1/S4 phases, expand-only.)
4. Report/line CRUD with a `receipt_url` field.
5. Submit → FX conversion → `publish("expense.submitted", …)`.
6. Consume `expense.approved/rejected` → set status (`APPROVED`→`REIMBURSED` / `REJECTED`).
7. Tests to DoD; image.

**Definition of done:** employee submits an expense with a receipt link; money stored as minor units + currency; FX deterministic against seeded rates; approval transitions driven by Workflow events.
