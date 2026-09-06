# Repo Spec — `svc-workflow` (Workflow & Notifications)

> **Owner:** Senior Backend Engineer · **Stack:** Python 3.12 + FastAPI 0.115 + SQLAlchemy 2 + Alembic · **DB:** `workflow_db` + SMTP (MailHog local). Read [`../ARCHITECTURE.md`](../ARCHITECTURE.md) and the shared skeleton in [`svc-identity.md` §1](./svc-identity.md#1-standard-service-skeleton-shared-by-all-four-services).

- **Role:** the domain-agnostic **approval + policy engine**, plus sending the approval email. It knows nothing about hours or receipts specifically — it operates on generic *approvables* described by events.
- **Owns contracts:** `contracts/openapi/workflow.v1.yaml`; event schemas `timesheet.approved|rejected`, `expense.approved|rejected`.
- **Emits:** the domain decision events above.
- **Consumes:** `timesheet.submitted` (Time), `expense.submitted` (Expense), `employee.*` (Identity).

## 1. Data model (`workflow_db`)

| Table | Key columns |
|---|---|
| `approvals` | `id`, `subject_type (TIMESHEET\|EXPENSE)`, `subject_id`, `requester_id`, `approver_id`, `status (PENDING\|APPROVED\|REJECTED)`, `reason`, `policy_flags (jsonb)`, `created_at`, `decided_at` — **`unique(subject_type, subject_id)`** so a redelivered `*.submitted` upserts, not duplicates |
| `approval_chains` | `id`, `employee_id`, `steps (jsonb: ordered approver rules)` |
| `policies` | `id`, `type (PER_RECEIPT_CAP\|OVERTIME_THRESHOLD)`, `params (jsonb)`, `active` |
| `notifications` | `id`, `approval_id`, `to`, `template`, `status`, `sent_at` |
| `employee_read` | id, display_name, email, manager_id, status — upserted from `employee.*` |

## 2. Engines

**Policy engine.** On each `*.submitted` event, evaluate active `policies`:
- `PER_RECEIPT_CAP` (Scenario 1): any expense line `amount_home_minor > cap` → flag + require manager approval.
- `OVERTIME_THRESHOLD` (Scenario 2): `overtime_hours > threshold` → require manager approval.
Policies are data (`params` jsonb), added without code changes where possible; new *types* are code + migration.

**Routing/state machine.** Resolve approver from `approval_chains`/manager (`employee_read.manager_id`) → create `approvals(PENDING)` → send the approval email. On decision, transition state and `publish` the domain decision event (`timesheet.approved`, `expense.rejected`, …) the owning service consumes.

```mermaid
stateDiagram-v2
  [*] --> PENDING: *.submitted + policy triggers
  PENDING --> APPROVED: POST /approvals/{id}/decision approve
  PENDING --> REJECTED: decision reject
  APPROVED --> [*]: publish *.approved
  REJECTED --> [*]: publish *.rejected
```

## 3. REST API (`workflow.v1.yaml` excerpt)

| Method & path | Auth | Purpose |
|---|---|---|
| `GET /approvals` | `MANAGER\|FINANCE` | approver's queue (page params, filter by status) |
| `GET /approvals/{id}` | approver | detail incl. `policy_flags` |
| `POST /approvals/{id}/decision` | approver | `{decision: APPROVE\|REJECT, comment}` → publishes decision event |
| `GET /workflow/policies` | `HR_ADMIN\|FINANCE` | list policies |
| `POST /workflow/policies` | `HR_ADMIN` | create/activate a policy |

Error codes (prefix `WORKFLOW_`): `WORKFLOW_APPROVAL_NOT_FOUND`, `WORKFLOW_NOT_APPROVER`, `WORKFLOW_ALREADY_DECIDED`, `WORKFLOW_UNKNOWN_POLICY_TYPE`.

## 4. Events

**Consumes:** `timesheet.submitted`, `expense.submitted`, `employee.*`. Handlers upsert the `approval` by `subject_id`, so a redelivery creates no duplicate.

**Emits** (`workflow.events`): `timesheet.approved|rejected`, `expense.approved|rejected`. (Approval creation and the email are internal — not broker events.)

## 5. Notifications

- Local: SMTP → **MailHog** (`mailhog:1025`), viewable at `http://localhost:8025` — this is how a lesson *sees* an approval request land.
- One template; recipient (the approver's `email`) from `employee_read`. Each send writes a `notifications` row.

## 6. Config, local run, testing, CI

Shape per [`svc-identity.md`](./svc-identity.md) §5–§8. Differences:

```
DATABASE_URL=postgresql+psycopg://atlas:atlas@postgres:5432/workflow_db
AMQP_URL=amqp://atlas:atlas@rabbitmq:5672/
JWT_SECRET=atlas-local-development-secret-32
SMTP_HOST=mailhog
SMTP_PORT=1025
# run: uv run uvicorn app.main:app --port 8004 --reload
```

- **Engine unit tests:** policy evaluation truth tables for each policy type; routing resolves the correct approver from chain/manager.
- **Idempotency:** a duplicate `*.submitted` delivery yields exactly one `approvals` row (upsert by `subject_id`); a decision on an already-decided approval returns `WORKFLOW_ALREADY_DECIDED`.
- **Contract tests:** decision events validate against the schemas `svc-time`/`svc-expense` consume; consuming `expense.submitted` across its additive versions stays green.

## 7. Implementation build order (Phase P3)

Conventions + shared template: [`../DEVELOPMENT.md`](../DEVELOPMENT.md).

1. Instantiate the service template; add the `employee.*` consumer → `employee_read` (upsert).
2. Author `workflow.v1.yaml` + decision-event schemas → merge.
3. Models/migrations: `approvals`, `approval_chains`, `policies`, `notifications`.
4. Consumers for `timesheet.submitted` / `expense.submitted`; policy engine (`PER_RECEIPT_CAP`, `OVERTIME_THRESHOLD`) + routing/state machine.
5. Approvals API (`GET /approvals`, `POST /approvals/{id}/decision`) → `publish` domain decision events.
6. Send the approval email to MailHog.
7. Policy admin API. Tests to DoD; image.

**Definition of done:** a submitted timesheet/expense creates one approval task and an email in MailHog; a decision publishes the correct domain event the owning service consumes; duplicate event deliveries are no-ops.

## 8. Implementation notes (as-built, P3)

- **No `approval_chains` table.** Routing is direct-manager-only: the approver is
  resolved as `employee_read.manager_id` for the requester. This matches every
  P3/P5 scenario (single-step manager approval) and avoids speculative
  multi-step-chain complexity; a future phase can add `approval_chains` without
  breaking the `approvals` schema if multi-step routing is ever required.
- **`overtime_threshold_hours` default is `5`, not `40`.** `svc-time` computes
  `overtime_hours` as hours *already in excess of* the 40-hour standard week
  (`total_hours - 40`, floored at 0), so a 40-hour threshold on top of that
  would almost never fire. The policy fires when `overtime_hours > 5`, i.e.
  more than 5 hours of overtime in a week requires manager approval. There is
  no compose env override for this — it is the `Settings` default in
  `app/core/config.py`.
- **MailHog Subject-header folding workaround.** Python's `EmailMessage`
  default policy folds headers over ~78 chars; for a `Subject` like
  `Atlas approval requested: Expense <uuid>` the fold landed after a colon and
  MailHog's parser mis-split it into a bogus header. Fixed by building the
  message with `email.policy.default.clone(max_line_length=998)`
  (`app/services/mail.py`). Any future outbound email template should reuse
  this same policy (or a shared helper) to avoid the same MailHog quirk.
- **Downstream collapse, not a workflow concern:** `svc-expense` collapses its
  own `APPROVED` status straight to `REIMBURSED` on finalize (no separate
  payment step exists yet); `svc-workflow` only ever publishes `APPROVED` or
  `REJECTED` — the collapse happens entirely in the expense consumer.
