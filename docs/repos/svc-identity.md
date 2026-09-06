# Repo Spec — `svc-identity` (Identity & Org)

> **Owner:** Senior Backend Engineer · **Stack:** Python 3.12 + FastAPI 0.115 + SQLAlchemy 2 + Alembic · **DB:** `identity_db`. Read [`../ARCHITECTURE.md`](../ARCHITECTURE.md) — start with **§0 Design charter** (thin services, no outbox/JWKS/tracing) — for auth, error envelope, events, and versioning; not restated here.

- **Role:** the authority for *who* someone is and *what they may do*. Issues the login JWT.
- **Owns contracts:** `contracts/openapi/identity.v1.yaml`; event schemas `employee.created|updated|deactivated`.
- **Emits:** `employee.*` on exchange `identity.events`.
- **Consumes:** none.
- **Downstream consumers:** `svc-time`, `svc-expense`, `svc-workflow` (read models), `hrms-web`.

## 1. Standard service skeleton (shared by all four services)

Deliberately small — a thin CRUD + emit/consume app. Every backend has the *same* shape:

```
svc-identity/
  app/
    main.py               # FastAPI app + routers + correlation-id middleware
    api/                  # routers: auth.py, employees.py, health.py
    core/                 # config.py (pydantic-settings), auth.py (HS256 verify + RBAC dep)
    db.py                 # SQLAlchemy session + base
    models.py             # SQLAlchemy models
    schemas.py            # pydantic request/response (match the OpenAPI)
    events.py             # publish(exchange, type, data) + consume(bindings)
    consumers.py          # BINDINGS: (exchange, queue, keys, handler) list for this service
  migrations/             # Alembic
  tests/                  # pytest: unit + api (httpx) + contract
  Dockerfile  pyproject.toml  .env.example
```

Every service ships: correlation-id middleware, the HS256 verify + RBAC dependency, `/healthz`, structured JSON logging, and the `events.py` helper. **`auth.py` and `events.py` are copied verbatim into each service** — in a polyrepo the only shared artifact is the *contract*, not a runtime library; both files (and the FastAPI `lifespan` that starts the consumer in-process — **one container, one process**) are given ready-to-copy in [`../RUNBOOK.md §4–§5`](../RUNBOOK.md#4-appcoreauthpy--the-shared-hs256-dependency-copied-into-each-service). No outbox, relay worker, dedupe table, or tracing SDK — the deliberate omissions from [`../ARCHITECTURE.md §0`](../ARCHITECTURE.md#0-design-charter--what-this-project-optimizes-for).

## 2. Data model (`identity_db`)

| Table | Key columns |
|---|---|
| `employees` | `id (uuid pk)`, `email (unique)`, `full_name`, `grade`, `cost_center`, `manager_id (fk→employees)`, `home_currency (char3)`, `pto_entitlement_days (int)`, `status (ACTIVE\|INACTIVE)`, `created_at`, `updated_at` |
| `roles` | `id`, `code (EMPLOYEE\|MANAGER\|FINANCE\|HR_ADMIN)`, `description` |
| `employee_roles` | `employee_id`, `role_id` (composite pk) |
| `credentials` | `employee_id (pk)`, `password_hash (argon2)`, `updated_at` |

No refresh-token or outbox tables. `home_currency` and `pto_entitlement_days` are added by Scenarios 4 and 3 (expand-only migrations).

## 3. REST API (`identity.v1.yaml` excerpt)

| Method & path | Auth | Purpose |
|---|---|---|
| `POST /auth/login` | public | email+password → `{ token }` (HS256, ~8h) |
| `GET /identity/employees` | `MANAGER\|HR_ADMIN` | list (page params) |
| `POST /identity/employees` | `HR_ADMIN` | create employee → publishes `employee.created` |
| `GET /identity/employees/{id}` | self or `MANAGER\|HR_ADMIN` | fetch one |
| `PATCH /identity/employees/{id}` | `HR_ADMIN` | update → publishes `employee.updated` |
| `POST /identity/employees/{id}/deactivate` | `HR_ADMIN` | → publishes `employee.deactivated` |

No JWKS / verify / token endpoints — every service verifies the HS256 token itself with the shared `JWT_SECRET`. Error codes (prefix `IDENTITY_`): `IDENTITY_INVALID_CREDENTIALS`, `IDENTITY_EMAIL_TAKEN`, `IDENTITY_EMPLOYEE_NOT_FOUND`, `IDENTITY_FORBIDDEN`.

## 4. Events emitted (`identity.events` exchange)

| `type` | Routing key | Payload highlights | Consumers |
|---|---|---|---|
| `employee.created` | `employee.created` | id, full_name, cost_center, manager_id, home_currency, pto_entitlement_days, status | time, expense, workflow |
| `employee.updated` | `employee.updated` | id + changed fields | time, expense, workflow |
| `employee.deactivated` | `employee.deactivated` | id | time, expense, workflow |

Publishing is simple: the handler commits the DB change, then calls `publish("employee.created", {...})`. Consumers upsert by `employee_id`, so a duplicate delivery is harmless (payloads in [`../DATA_CONTRACTS.md §3`](../DATA_CONTRACTS.md#3-event-payloads-data-field-of-the-cloudevents-envelope)).

## 5. Config (`.env.example`)

Local values come from `platform-outerloop/compose/.env` (all `atlas`); this file just documents them.

```
DATABASE_URL=postgresql+psycopg://atlas:atlas@postgres:5432/identity_db
AMQP_URL=amqp://atlas:atlas@rabbitmq:5672/
JWT_SECRET=atlas-dev-secret        # shared by all services (HS256)
JWT_TTL_HOURS=8
DEMO_PASSWORD=atlas                # used only by the seed script
LOG_LEVEL=INFO
```

## 6. Local run

```bash
cd svc-identity
cp .env.example .env
uv sync
uv run alembic upgrade head
uv run uvicorn app.main:app --host 0.0.0.0 --port 8001 --reload
```

Normally started as part of the estate (`docker compose up --build -d`). Standalone needs a reachable Postgres + RabbitMQ. No separate relay process — events publish inline.

## 7. Testing

- **Unit:** token issue/verify, RBAC checks, password hashing.
- **API:** httpx `AsyncClient` against the app with a throwaway Postgres.
- **Contract:** emitted `employee.*` payloads validate against `contracts/events/*.json`; the app's generated OpenAPI equals the committed `identity.v1.yaml` (drift gate).

## 8. CI

`ruff → mypy → pytest → openapi drift check → docker build/push`. A contract change opens a PR against `platform-outerloop/contracts/`, gated by `oasdiff`.

## 9. Implementation build order (Phase P1)

Conventions + shared template: [`../DEVELOPMENT.md`](../DEVELOPMENT.md).

1. Instantiate the service template → app boots with `/healthz` + correlation-id middleware.
2. Author `identity.v1.yaml` (login + employees) and `employee.*` schemas → merge in the registry.
3. Models + migrations: `employees`, `roles`, `employee_roles`, `credentials`.
4. Auth: argon2 password hashing + HS256 login token; the shared verify/RBAC dependency.
5. Employee CRUD + RBAC.
6. `publish` `employee.created/updated/deactivated` after commit; contract-test payloads.
7. Tests; publish image.

**Definition of done:** login returns a token that another service accepts; creating an employee publishes exactly one `employee.created`; `mypy --strict` + contract drift checks green.
