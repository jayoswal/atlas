# Repo Spec — `platform-outerloop` (Platform / DevOps)

> **Owner:** Senior DevOps Engineer · **Stack:** Traefik 3 (routing) · docker-compose v2 · GitHub Actions. This repo owns the **outer loop**: wire the repos together, run them locally, gate contracts, and ship. Read [`../ARCHITECTURE.md`](../ARCHITECTURE.md) — especially the **§0 Design charter** — for the deliberate-simplicity constraints this repo honors.

- **Owns:** the gateway config, the **one** docker-compose that runs the whole estate, the **contract registry**, CI, and the seed/smoke scripts.
- **Depends on:** the five code repos (built as images by compose).
- **Does not own:** application code or the per-service business contracts (owned by each service team; this repo hosts and gates them).
- **Deliberately does NOT include:** Kubernetes/Helm charts, an observability stack, MinIO, or key-management tooling. Those are prod concerns captured as short notes (§6), never built here.

## 1. Repository layout

```
platform-outerloop/
  compose/
    docker-compose.yml            # the whole estate (the only orchestration)
    postgres-init.sql             # creates the 4 databases on first boot
    .env.example                  # one config file (copy to .env)
  contracts/                      # THE registry (contract-first)
    openapi/{identity,time,expense,workflow}.v1.yaml
    schemas/{employee.created,timesheet.submitted,expense.submitted,...}/
  ci/
    service.yml                   # reusable: lint + test + build image (each backend)
    web.yml                       # reusable: lint + typecheck + gen:api drift + build
    contracts.yml                 # oasdiff + jsonschema check on any contracts PR
    integration.yml               # compose up + seed + smoke on every image change
  scripts/
    seed.py  smoke.py
  pyproject.toml                  # deps for seed/smoke (httpx only), run via `uv run`
```

## 2. Local orchestration (one compose, one command)

`compose/docker-compose.yml` — every box in the [container diagram](../ARCHITECTURE.md#1-container-diagram-c4-level-2):

| Service | Image / build | Host port |
|---|---|---|
| `gateway` (Traefik) | traefik:3 | 8080 (`/`→web, `/api`→services) |
| `hrms-web` | build ../hrms-web | via gateway |
| `svc-identity` | build ../svc-identity | 8001 |
| `svc-time` | build ../svc-time | 8002 |
| `svc-expense` | build ../svc-expense | 8003 |
| `svc-workflow` | build ../svc-workflow | 8004 |
| `postgres` | postgres:16 (creates 4 DBs on init) | 5432 |
| `rabbitmq` | rabbitmq:3.13-management | 5672 / 15672 (UI) |
| `adminer` | adminer:4 | 8081 (DB browser) |
| `mailhog` | mailhog/mailhog | 1025 / 8025 (UI) |

**Ten containers, one Postgres.** The four databases are created by `postgres-init.sql` (mounted into `postgres`'s `/docker-entrypoint-initdb.d/`); each service is handed only its own `DATABASE_URL` and never opens another — the boundary is by convention in code, the footprint is one container ([`../ARCHITECTURE.md §6`](../ARCHITECTURE.md#6-data-management)). `depends_on: { condition: service_healthy }` on Postgres and RabbitMQ orders startup — no wait-for script needed.

### 2.1 Configuration — one file, uniform values

All local config is a **single** `compose/.env` (copied from `.env.example`), interpolated into every service. One rule: **username `atlas`, password `atlas` everywhere.**

```dotenv
# compose/.env  — LOCAL ONLY, committed as .env.example
COMPOSE_PROJECT_NAME=atlas

POSTGRES_USER=atlas
POSTGRES_PASSWORD=atlas
# init creates 4 databases: identity_db, time_db, expense_db, workflow_db

RABBITMQ_USER=atlas
RABBITMQ_PASSWORD=atlas

# Auth — one shared HS256 secret, verified in-process by every service
JWT_SECRET=atlas-local-development-secret-32
JWT_TTL_HOURS=8

SMTP_HOST=mailhog
SMTP_PORT=1025
LOG_LEVEL=INFO
DEMO_PASSWORD=atlas          # every seeded demo user logs in with this
```

Each service composes its own connection strings from these, e.g. `svc-identity` gets
`DATABASE_URL=postgresql+psycopg://atlas:atlas@postgres:5432/identity_db` and
`AMQP_URL=amqp://atlas:atlas@rabbitmq:5672/`. Every repo keeps a `.env.example` documenting its variables; **local values all come from this one file** — nothing else to maintain, no secrets, no keypairs.

### 2.2 Commands (plain — no Makefile)

Run from `platform-outerloop/compose/`. Seed/smoke use `uv run` (this repo's `pyproject.toml` pins only httpx).

```bash
docker compose up --build -d            # start / rebuild the whole estate
docker compose down                     # stop (add -v to wipe database volumes)
docker compose logs -f svc-time         # tail one service
uv run --project .. python ../scripts/seed.py
uv run --project .. python ../scripts/smoke.py
docker compose exec postgres psql -U atlas -d identity_db   # SQL shell
```

**Migrations** are not a separate command — each backend container runs `alembic upgrade head` in its entrypoint before starting. **UI contract regen** is `npm run gen:api` in `hrms-web`.

**Acceptance:** on a clean machine with only Docker installed,
`docker compose up --build -d` → `uv run --project .. python ../scripts/seed.py` → `uv run --project .. python ../scripts/smoke.py`
brings up all 6 repos and passes a cross-service assertion.

### 2.3 Local dashboards (see everything — no extra stack)

The value here is *visibility with zero added infrastructure*: the tools we already run each have a UI. Same `atlas`/`atlas` login where one is needed.

| Tool | URL | Login | See |
|---|---|---|---|
| **hrms-web** (the app) | http://localhost:8080 | `ada@atlas.dev` / `atlas` | the product |
| **Adminer** (Postgres) | http://localhost:8081 | server `postgres` · `atlas`/`atlas` | all four databases, run SQL |
| **RabbitMQ** | http://localhost:15672 | `atlas`/`atlas` | exchanges, queues, messages in flight |
| **MailHog** | http://localhost:8025 | none | approval / notification emails |
| **Traefik** | http://localhost:8082/dashboard/ | none | live routing rules |

*(Optional, off by default: an `observability` compose profile could add Jaeger/Prometheus/Grafana for the curious — not required to run or understand Atlas. See [`../ARCHITECTURE.md §7`](../ARCHITECTURE.md#7-observability--just-enough-to-see-the-flow).)*

### 2.4 Seed dataset & demo accounts (`scripts/seed.py`)

Deterministic fixtures so the estate is demo-ready and the smoke test is stable. **Every demo user logs in with `atlas`**, email `<first-name>@atlas.dev`.

| Email | Password | Role | Reports to | Used in |
|---|---|---|---|---|
| `admin@atlas.dev` | `atlas` | EMPLOYEE, HR_ADMIN | — | manages employees |
| `grace@atlas.dev` | `atlas` | EMPLOYEE, MANAGER | — | manages Ada |
| `finance@atlas.dev` | `atlas` | EMPLOYEE, FINANCE | — | finance workflows |
| `ada@atlas.dev` | `atlas` | EMPLOYEE | grace | smoke test and employee workflows |

P1 seeds identity-owned users, roles, credentials, and their cost-center values.
P2 seeds Time employee projections plus Expense employee profiles, categories,
and FX rates. Workflow fixtures arrive with P3. Each service owns its seed and
inserts missing natural keys without overwriting existing domain state, so the
platform seed is safe to re-run.

## 3. API Gateway (Traefik) — routing only

- Configured entirely by **Docker labels** on each service (the `docker` provider) — no `routes.yml` file to maintain. The concrete labels are in [`../RUNBOOK.md §1`](../RUNBOOK.md#1-platform-outerloopcomposedocker-composeyml). Path prefixes:
  - `/api/v1/auth`, `/api/v1/identity` → `svc-identity`
  - `/api/v1/time` → `svc-time`
  - `/api/v1/expense` → `svc-expense`
  - `/api/v1/workflow`, `/api/v1/approvals` → `svc-workflow`
  - `/` → `hrms-web`
- **No auth middleware, no plugins.** Token verification lives in each service (see [`../ARCHITECTURE.md §4`](../ARCHITECTURE.md#4-authn--authz)); a CORS header for the web origin is the only middleware.

> **The concrete starter files** — `docker-compose.yml`, `postgres-init.sql`, the backend `Dockerfile`/entrypoint, and the copied `auth.py` / `events.py` — live in [`../RUNBOOK.md`](../RUNBOOK.md). Build from those rather than inventing wiring.

## 4. Contract registry & gating (the coordination lesson)

This is the part that *is* the point — keep it, keep it strict.

- Each service team owns its files under `contracts/` (`openapi/<svc>.v1.yaml`, `events/<event>.json`). A change is a **PR to this repo**, reviewed via CODEOWNERS by the producing team, which notifies consumers.
- `ci/contracts.yml` runs on every contracts PR: `oasdiff` (blocks a breaking REST change unless it's a new `/v2`) and JSON-Schema validation of event payloads.
- **A contract merges before any implementation PR.** The UI's `gen:api` drift check enforces that the client is regenerated from the merged spec. This is how a feature stays coordinated across repos.

## 5. CI (GitHub Actions) — small on purpose

- **Each backend repo** calls `ci/service.yml`: `ruff` → `mypy` → `pytest` → build image → push to GHCR (`:sha`).
- **`hrms-web`** calls `ci/web.yml`: ESLint → `tsc` → `gen:api` drift check → `vitest` → build → push.
- **Any contracts PR** runs `ci/contracts.yml` (above).
- **`ci/integration.yml`** (on image change): `docker compose up` at the new tags → `seed.py` → `smoke.py` → Playwright scenario suite.

No Trivy/coverage/deploy gates required for the teaching goal — add them if you want, they don't change the lesson.

## 6. How this maps to production (conceptual — not built)

Kept as a one-page note so readers see the enterprise endpoint without the maintenance burden:

- **Compose service → Kubernetes Deployment + Service** (one per app); Postgres → a managed database with four schemas/DBs; RabbitMQ → a managed broker.
- **The single `.env` → ConfigMaps + Secrets**; `JWT_SECRET` → a real secret; HS256 → RS256 + JWKS at the edge.
- **`docker compose up` ordering → a deploy pipeline** that applies the same rule from [`../ARCHITECTURE.md §9`](../ARCHITECTURE.md#9-rollout--deploy-ordering-rules): migrate → producers → consumers → UI.
- **The dashboards → a real observability stack** (OpenTelemetry/Jaeger, Prometheus, Grafana).

Atlas is intentionally the *left* side of every arrow. The value of the exercise is the coordination model, which is identical in both columns.

## 7. Definition of done (DevOps)

- `docker compose up --build -d` → `uv run --project .. python ../scripts/seed.py` → `uv run --project .. python ../scripts/smoke.py` green on a clean machine with Docker and `uv` installed.
- All five dashboards in §2.3 reachable; you can watch a submitted expense appear in the RabbitMQ UI and its approval email in MailHog.
- `ci/contracts.yml` blocks a breaking contract change; `ci/integration.yml` runs the [`../SCENARIOS.md`](../SCENARIOS.md) suite.
- One `compose/.env` is the only configuration a new developer edits.
