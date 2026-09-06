# Atlas HRMS — Data Contracts (field-level)

> The authoritative, unambiguous field specification an engineer codes against: canonical enums, per-entity schemas (type · nullability · constraints), full event payloads, representative request/response bodies, and the consolidated error-code registry. The machine-readable OpenAPI/JSON-Schema in `platform-outerloop/contracts/` is generated to match this; where they ever differ, the registry file is source of truth and this doc is updated to agree.

- **Conventions:** all ids are UUID v4 strings. Timestamps are RFC-3339 UTC (`...Z`). Money is an **integer in minor units** (`_minor`) paired with an ISO-4217 `currency` (`char(3)`). Dates are `YYYY-MM-DD`. List endpoints use `?limit&offset` and return `{ items, total }` ([`ARCHITECTURE.md §3`](./ARCHITECTURE.md#3-synchronous-communication-rest)).
- **Companion:** [`ARCHITECTURE.md`](./ARCHITECTURE.md) (standards), per-repo specs in `repos/`.

---

## 1. Canonical enums (single definition, referenced everywhere)

| Enum | Values |
|---|---|
| `Role` | `EMPLOYEE`, `MANAGER`, `FINANCE`, `HR_ADMIN` |
| `EmployeeStatus` | `ACTIVE`, `INACTIVE` |
| `TimesheetStatus` | `DRAFT`, `PENDING_APPROVAL`, `APPROVED`, `REJECTED` |
| `PtoLedgerEntryType` | `ACCRUAL`, `TAKEN`, `ADJUSTMENT` |
| `PtoRequestStatus` | `PENDING`, `APPROVED`, `REJECTED`, `CANCELLED` |
| `ExpenseStatus` | `DRAFT`, `PENDING_APPROVAL`, `APPROVED`, `REJECTED`, `REIMBURSED` |
| `ApprovalSubjectType` | `TIMESHEET`, `EXPENSE` |
| `ApprovalStatus` | `PENDING`, `APPROVED`, `REJECTED` |
| `ApprovalDecision` | `APPROVE`, `REJECT` |
| `PolicyType` | `PER_RECEIPT_CAP`, `OVERTIME_THRESHOLD` |
| `NotificationChannel` | `EMAIL`, `IN_APP` |
| `NotificationStatus` | `QUEUED`, `SENT`, `FAILED` |
| `Currency` | ISO-4217 alpha-3 (`USD`, `EUR`, `INR`, `GBP`, …) |

Enums are stored as Postgres native `text` + `CHECK` constraint (not PG `ENUM` type — avoids migration pain) and mirrored as Python `StrEnum` / TS string-literal unions.

---

## 2. Entity schemas

### 2.1 `svc-identity`

**employee**
| Field | Type | Null | Constraint / notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `email` | text | no | unique, citext-lower |
| `full_name` | text | no | |
| `grade` | text | no | e.g. `IC1..IC6`, `M1..M4`; drives entitlement |
| `cost_center` | text | no | free-form code |
| `manager_id` | uuid | yes | FK→employee.id; null for top of tree |
| `home_currency` | char(3) | no | default `USD` |
| `pto_entitlement_days` | int | no | default per `grade`; default 0 until set |
| `status` | text | no | `EmployeeStatus`, default `ACTIVE` |
| `created_at`/`updated_at` | timestamptz | no | |

Indexes: `unique(email)`, `index(manager_id)`, `index(cost_center)`.

**role**(`id`,`code Role`,`description`) · **employee_role**(`employee_id`,`role_id`, PK both) · **credential**(`employee_id` PK, `password_hash` argon2, `updated_at`).

*(No outbox / refresh-token / service-account tables — auth is a single HS256 token and events publish directly. See [`ARCHITECTURE.md §0/§4/§5`](./ARCHITECTURE.md#0-design-charter--what-this-project-optimizes-for).)*

### 2.2 `svc-time`

**timesheet**(`id`,`employee_id`,`period_start date`,`period_end date`,`status TimesheetStatus`,`total_hours numeric(6,2)`,`overtime_hours numeric(6,2) default 0`,`created_at`,`updated_at`) — unique `(employee_id, period_start)`; CHECK no overlap enforced in domain.
**time_entry**(`id`,`timesheet_id` FK,`work_date date`,`hours numeric(4,2)`,`project_code text`,`note text?`).
**pto_request**(`id`,`employee_id`,`start_date`,`end_date`,`days numeric(4,1)`,`status PtoRequestStatus`,`created_at`).
**pto_ledger**(`id`,`employee_id`,`entry_type PtoLedgerEntryType`,`days numeric(5,1)`,`effective_date`,`source_ref text?`).
**employee_read**(`id` PK,`display_name`,`cost_center`,`manager_id?`,`status`,`pto_entitlement_days int default 0`) — upserted from `employee.*` events (natural key = `id`, so redelivery is safe; no dedupe table).

### 2.3 `svc-expense`

**expense_report**(`id`,`employee_id`,`title`,`status ExpenseStatus`,`home_currency char(3)`,`total_home_minor bigint default 0`,`created_at`,`updated_at`).
**expense_line**(`id`,`report_id` FK,`category_id uuid` FK,`amount_minor bigint`,`currency char(3)`,`amount_home_minor bigint`,`fx_rate numeric(18,8)`,`receipt_url text?`,`spent_on date`,`policy_flags jsonb default '[]'`).
**category**(`id`,`code text unique`,`name`,`active bool default true`).
**fx_rate**(`rate_date date`,`base_currency char(3)`,`quote_currency char(3)`,`rate numeric(18,8)`, PK `(rate_date,base_currency,quote_currency)`).
**expense_profile**(`employee_id` PK,`default_currency char(3)`,`category_access jsonb`).
**employee_read**(as time, plus `home_currency char(3)`).

### 2.4 `svc-workflow`

**approval**(`id`,`subject_type ApprovalSubjectType`,`subject_id uuid`,`requester_id uuid`,`approver_id uuid`,`status ApprovalStatus`,`reason text?`,`policy_flags jsonb default '[]'`,`comment text?`,`created_at`,`decided_at?`).
**approval_chain**(`id`,`employee_id`,`steps jsonb`) — `steps` = ordered `[{ "type":"MANAGER" }, { "type":"ROLE", "role":"FINANCE" }]`.
**policy**(`id`,`type PolicyType`,`params jsonb`,`active bool`,`created_at`). `params`: `PER_RECEIPT_CAP → {"cap_minor":15000,"currency":"EUR","category_code":"CLIENT_ENT"?}`; `OVERTIME_THRESHOLD → {"threshold_hours":10}`.
**notification**(`id`,`approval_id? uuid`,`channel NotificationChannel`,`to text`,`template text`,`status NotificationStatus`,`sent_at?`).
**employee_read**(`id`,`display_name`,`email`,`manager_id?`,`status`).

---

## 3. Event payloads (`data` field of the CloudEvents envelope)

Envelope is defined in [`ARCHITECTURE.md §5`](./ARCHITECTURE.md#5-asynchronous-communication-events). Only `data` shown; every payload also travels with `id` (=dedupe key), `type`, `correlationid`, `time`.

```jsonc
// employee.created  (schema 1-1-0)   producer: svc-identity
{ "employee_id":"uuid", "email":"a@x.com", "full_name":"Ada L.",
  "cost_center":"CC-100", "manager_id":"uuid|null",
  "home_currency":"USD", "pto_entitlement_days":25, "status":"ACTIVE" }

// employee.updated  (1-1-0)          data = employee_id + only changed fields
{ "employee_id":"uuid", "changed":{ "cost_center":"CC-200", "manager_id":"uuid" } }

// employee.deactivated (1-0-0)
{ "employee_id":"uuid" }

// timesheet.submitted (1-1-0)        producer: svc-time
{ "timesheet_id":"uuid", "employee_id":"uuid",
  "period_start":"2026-09-01", "period_end":"2026-09-07",
  "total_hours":48.0, "overtime_hours":8.0 }

// timesheet.approved | timesheet.rejected (1-0-0)  producer: svc-workflow
{ "timesheet_id":"uuid", "approval_id":"uuid", "approver_id":"uuid", "comment":"string|null" }

// expense.submitted (1-2-0)          producer: svc-expense
{ "report_id":"uuid", "employee_id":"uuid", "home_currency":"USD",
  "total_home_minor": 42350,
  "lines":[ { "line_id":"uuid", "category_code":"CLIENT_ENT",
              "amount_minor":18000, "currency":"EUR",
              "amount_home_minor":19500, "spent_on":"2026-09-03" } ] }

// expense.approved | expense.rejected (1-0-0)  producer: svc-workflow
{ "report_id":"uuid", "approval_id":"uuid", "approver_id":"uuid", "comment":"string|null" }
```

That is the full event set — the four boundary-crossing events. Approval creation, the approval email, and marking a report `REIMBURSED` all happen *inside* a service and are not broker events.

**Schema evolution rule:** fields are added optional within a major; `total_hours`/`overtime_hours`, `currency`/`amount_home_minor`, `home_currency`/`pto_entitlement_days` are the additive deltas introduced by Scenarios 2/4/3. A consumer on an older minor must ignore unknown fields.

---

## 4. Representative request/response bodies

### POST `/api/v1/auth/login`
```jsonc
// req
{ "email":"ada@atlas.dev", "password":"atlas" }
// 200  (one HS256 token, ~8h; re-login is the refresh story)
{ "token":"<jwt>", "token_type":"Bearer" }
```

### POST `/api/v1/identity/employees`  (roles: `HR_ADMIN`)
```jsonc
// req
{ "email":"grace@atlas.dev", "full_name":"Grace H.", "grade":"IC4",
  "cost_center":"CC-100", "manager_id":"7a1c...", "home_currency":"USD",
  "roles":["EMPLOYEE"] }
// 201
{ "id":"9f2b...", "email":"grace@atlas.dev", "full_name":"Grace H.",
  "grade":"IC4", "cost_center":"CC-100", "manager_id":"7a1c...",
  "home_currency":"USD", "pto_entitlement_days":22, "status":"ACTIVE",
  "provisioning":"PROVISIONING" }
```

### POST `/api/v1/expense/reports/{id}/submit`  (owner)
```jsonc
// req  (body optional; server reads the report's lines)
{}
// 202  (async approval; report moves to PENDING_APPROVAL)
{ "report_id":"c1...", "status":"PENDING_APPROVAL", "total_home_minor":42350,
  "home_currency":"USD", "policy_flags":[ { "line_id":"l1...", "code":"PER_RECEIPT_CAP", "cap_minor":15000, "currency":"EUR" } ] }
```

### GET `/api/v1/approvals?status=PENDING&limit=50`  (roles: `MANAGER|FINANCE`)
```jsonc
{ "items":[ { "id":"ap1...", "subject_type":"EXPENSE", "subject_id":"c1...",
    "requester":{ "id":"9f2b...", "display_name":"Grace H." },
    "status":"PENDING", "reason":"POLICY_CAP",
    "policy_flags":[ { "code":"PER_RECEIPT_CAP" } ], "created_at":"2026-09-06T10:00:00Z" } ],
  "total":1 }
```

### POST `/api/v1/approvals/{id}/decision`  (approver)
```jsonc
// req
{ "decision":"APPROVE", "comment":"Within travel policy." }
// 200
{ "id":"ap1...", "status":"APPROVED", "decided_at":"2026-09-06T10:05:00Z" }
```

### Error (shape from ARCHITECTURE §3.1)
```jsonc
// 422
{ "error":{ "code":"EXPENSE_UNKNOWN_CURRENCY",
  "message":"No FX rate available for XAF on 2026-09-03.",
  "correlation_id":"b3c1...", "details":[ { "field":"lines[0].currency", "issue":"unsupported" } ] } }
```

---

## 5. Consolidated error-code registry

`code` is stable, UPPER_SNAKE, service-prefixed. UIs branch on `code`, never `message` ([`ARCHITECTURE.md §3.1`](./ARCHITECTURE.md#31-standard-error-envelope)).

| Service | Codes → HTTP |
|---|---|
| identity | `IDENTITY_INVALID_CREDENTIALS`→401, `IDENTITY_TOKEN_EXPIRED`→401, `IDENTITY_EMAIL_TAKEN`→409, `IDENTITY_EMPLOYEE_NOT_FOUND`→404, `IDENTITY_FORBIDDEN`→403 |
| time | `TIME_OVERLAPPING_ENTRY`→409, `TIME_NOT_DRAFT`→422, `TIME_TIMESHEET_NOT_FOUND`→404, `TIME_INSUFFICIENT_PTO`→422 |
| expense | `EXPENSE_NOT_DRAFT`→422, `EXPENSE_REPORT_NOT_FOUND`→404, `EXPENSE_UNKNOWN_CURRENCY`→422, `EXPENSE_OVER_CAP`→422*(flag, not a hard reject unless configured)*, `EXPENSE_CATEGORY_INACTIVE`→422 |
| workflow | `WORKFLOW_APPROVAL_NOT_FOUND`→404, `WORKFLOW_NOT_APPROVER`→403, `WORKFLOW_ALREADY_DECIDED`→409, `WORKFLOW_UNKNOWN_POLICY_TYPE`→400 |
| shared | `VALIDATION_ERROR`→400, `UNAUTHENTICATED`→401, `FORBIDDEN`→403, `NOT_FOUND`→404 |
