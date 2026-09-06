# Atlas HRMS — Development & Implementation Guide

> How engineers turn these specs into running software. Estate-wide conventions live here; per-repo build steps live in each `repos/*.md`. Read [`ARCHITECTURE.md`](./ARCHITECTURE.md) for the standards this guide operationalizes.

- **Audience:** the frontend, backend, and DevOps engineers implementing the six repos.
- **Companion docs:** [`PROJECT_OVERVIEW.md`](./PROJECT_OVERVIEW.md), [`ARCHITECTURE.md`](./ARCHITECTURE.md), [`SCENARIOS.md`](./SCENARIOS.md).

---

## 1. Prerequisites (developer workstation)

| Tool | Version | Notes |
|---|---|---|
| Docker + Compose v2 | 26+ | runs the whole estate locally |
| Node | 20 LTS | `hrms-web` |
| Python | 3.12 | the four services |
| `uv` | latest | Python env + dependency manager (fast, lockfile-based) |
| `git` | 2.40+ | — |

`uv` installs the pinned Python 3.12 runtime when the host Python differs.
Postgres, RabbitMQ, MailHog, and Adminer all run in containers.

## 2. Repository & workspace layout

Six independent Git repositories are cloned side-by-side under
`/home/oswa/atlas-repos`. The separate `/home/oswa/atlas` repository contains
documentation only. `platform-outerloop`'s compose file references the code
repositories by relative path:

```
atlas-repos/
  hrms-web/
  svc-identity/
  svc-time/
  svc-expense/
  svc-workflow/
  platform-outerloop/        # docker-compose builds ../svc-* and ../hrms-web
```

Bootstrap the existing public repositories:

```bash
mkdir -p /home/oswa/atlas-repos && cd /home/oswa/atlas-repos
for repo in hrms-web svc-identity svc-time svc-expense svc-workflow platform-outerloop; do
  git clone "https://github.com/jayoswal/${repo}.git"
done
cd platform-outerloop/compose
cp .env.example .env                 # one config file, values already filled (atlas/atlas)
docker compose up --build -d         # whole estate
uv run --project .. python ../scripts/seed.py
uv run --project .. python ../scripts/smoke.py
```

Then open the app at http://localhost:8080 (log in `ada@atlas.dev` / `atlas`) and the dev dashboards in [`repos/platform-outerloop.md §2.3`](./repos/platform-outerloop.md#23-local-dev-dashboards-visualize-everything--local-only).

## 3. Service template (all four backends share one skeleton)

Backends are generated from a **`cookiecutter` service template** kept in `platform-outerloop/templates/service/` so every service is structurally identical (see [`repos/svc-identity.md` §1](./repos/svc-identity.md#1-standard-service-skeleton-shared-by-all-four-services)). The template ships, pre-wired:

- FastAPI app factory + routers + `/healthz`
- correlation-id middleware, a ~10-line HS256 JWT verify + RBAC dependency
- SQLAlchemy 2 session, Alembic configured, base models
- a tiny event helper: `publish(type, data)` (publish-after-commit) and a consumer loop with idempotent-upsert handlers
- structured JSON logging
- pytest harness (unit + httpx API + contract), `ruff` + `mypy` config, Dockerfile, `pyproject.toml`

New service = instantiate template → define models/routers/events → author the OpenAPI/event contracts. **Do not hand-roll cross-cutting concerns; they come from the template.** Keep each service thin — a few hundred lines; there is intentionally no outbox, no dedupe table, no tracing SDK to wire up.

**Pinned backend dependencies** (`pyproject.toml`, identical across services — short by design):
```toml
[project.dependencies]
fastapi = "0.115.*"
uvicorn = { version = "0.30.*", extras = ["standard"] }
pydantic = "2.*"
pydantic-settings = "2.*"
sqlalchemy = "2.0.*"
alembic = "1.13.*"
psycopg = { version = "3.2.*", extras = ["binary"] }
aio-pika = "9.*"              # RabbitMQ client
pyjwt = "2.*"                 # HS256 encode/verify (symmetric, no key infra)
argon2-cffi = "23.*"         # password hashing
httpx = "0.27.*"             # tests
[dependency-groups.dev]
pytest = "8.*"; pytest-asyncio = "*"
ruff = "*"; mypy = "1.*"; jsonschema = "4.*"
```

The version examples above define the supported dependency families. The
executable repositories additionally pin exact direct and transitive versions
according to [`IMPLEMENTATION_PLAN.md §P0.1`](./IMPLEMENTATION_PLAN.md#p01--reproducible-dependency-baseline).
For this application estate, committed `uv.lock`/`package-lock.json` files and
immutable container digests are part of the source of truth.
Frontend dependency list is in [`repos/hrms-web.md §8`](./repos/hrms-web.md#8-development-guide-how-to-build-it). Field-level schemas engineers code against: [`DATA_CONTRACTS.md`](./DATA_CONTRACTS.md). One config file + seed data: [`repos/platform-outerloop.md §2.1`](./repos/platform-outerloop.md#21-configuration--one-file-uniform-values).

**Two decisions that remove the usual ambiguity (both shown as copy-paste code in [`RUNBOOK.md`](./RUNBOOK.md)):**
- **Process model:** one container, one process per service. The event consumer runs as a background task started in FastAPI's `lifespan`; there is no separate worker/relay. Migrations run in the container entrypoint (`alembic upgrade head`) before `uvicorn`.
- **Shared code in a polyrepo:** there is **no shared runtime library.** The two cross-cutting files — `core/auth.py` (HS256 verify + RBAC) and `events.py` (`publish`/`consume`) — are **copied verbatim** into each service. The only thing genuinely shared across repos is the *contract* in the registry. (This is itself a polyrepo teaching point: coordination happens through contracts, not a common lib.)

## 4. Coding standards

### 4.1 Python (services)
- **Style/lint:** `ruff` (lint + format), line length 100. **Types:** `mypy --strict`; no untyped defs.
- **Layout:** routers thin (I/O + validation only) → `services/` hold domain logic → `models/` are persistence only. Pydantic v2 schemas map 1:1 to the OpenAPI contract.
- **Errors:** raise typed domain exceptions mapped to the standard error envelope by one exception handler; `code` is a service-prefixed enum ([`ARCHITECTURE.md §3.1`](./ARCHITECTURE.md#31-standard-error-envelope)).
- **DB + events:** commit the domain change, then `publish(...)` the event in the same handler. Consumers upsert by natural key so redelivery is harmless. (No outbox, no dedupe table — see [`ARCHITECTURE.md §5`](./ARCHITECTURE.md#5-asynchronous-communication-events).)
- **Money:** integer minor units + ISO-4217 code. No floats, ever.
- **Tests:** pytest; `httpx.AsyncClient` for API tests against a disposable Postgres (testcontainers). Contract tests validate emitted payloads against JSON Schemas and the app's generated OpenAPI against the committed contract.

### 4.2 TypeScript (web) — see [`repos/hrms-web.md §8`](./repos/hrms-web.md#8-development-guide-how-to-build-it)
`strict` + `noUncheckedIndexedAccess`; no `any`; data access only through RTK Query hooks; UI reads design **tokens**, never literal colors; every string via i18n.

### 4.3 Universal
- **12-factor config:** all config from env vars; no secrets in code or images; `.env.example` documents every var.
- **Logging:** structured JSON to stdout; include `correlation_id`; no PII beyond `employee_id`.
- **Public interfaces are the contract:** you may refactor internals freely; you may not change an OpenAPI path or event payload without a registry PR.

## 5. Contract-first workflow (the coordination protocol)

A feature that crosses repos is driven by the contract, in this order:

1. **Author/modify the contract** in `platform-outerloop/contracts/` (OpenAPI for REST, a JSON Schema per event) → PR. CI runs `oasdiff` (breaking-change block) + JSON-Schema validation. CODEOWNERS = the producing team; consumers are auto-notified.
2. **Merge the contract** before writing implementation code. It is the source of truth.
3. **Producers** implement to the contract (new endpoint/field/event), backward-compatible.
4. **Consumers** (including `hrms-web` via `gen:api`) implement against the merged contract.
5. **Integration** pipeline composes the estate and runs the scenario's E2E test.

> Rule of thumb: *if two engineers need to talk to coordinate a change, the thing they are coordinating is a contract — write it down and merge it first.*

## 6. Git & pull-request conventions

- **Branching:** trunk-based. Short-lived `feature/<ticket>-slug` branches off `main`; squash-merge; `main` always deployable.
- **Commits:** Conventional Commits (`feat:`, `fix:`, `chore:`, `refactor:`, `test:`, `docs:`) → drives semantic version bump per repo.
- **Versioning:** each repo is independently **SemVer**-tagged; images tagged with the commit SHA (traceable) and the SemVer tag (releasable).
- **PR template (every repo):** summary · linked ticket · **contract impact** (which OpenAPI/event changed, producer/consumer direction) · **backward-compatibility** statement · **deploy-order note** · test evidence · screenshots (web).
- **CODEOWNERS:** each service team owns its code + its contract file; platform team owns `charts/`, `gateway/`, `ci/`. A contract PR requires the producing team's review.
- **Definition of Done (repo-agnostic):** lint + types clean · unit/contract tests green · migration is expand-only · contract in sync · PR template complete · CI green.

## 7. Cross-repo feature: worked delivery flow

Using **Scenario 2 — overtime approval** ([`SCENARIOS.md`](./SCENARIOS.md)) as the reference cadence:

```mermaid
sequenceDiagram
  participant C as Contracts (outerloop)
  participant T as svc-time
  participant W as svc-workflow
  participant U as hrms-web
  C->>C: PR: timesheet.submitted 1-0-0 -> 1-1-0 (add overtime_hours), workflow OpenAPI (approvals)
  Note over C: merged first (lint-gated)
  T->>T: migration (add overtime_hours) + emit new field
  W->>W: consume 1-1-0, add OVERTIME_THRESHOLD policy + approvals API
  U->>U: gen:api, build Approvals queue + status badge (flagged behind a feature flag)
  Note over T,U: deploy order: migration -> producer(time) -> consumer(workflow) -> UI
```

The **assessable outcome** is that engineers (a) name the exact contract deltas, (b) keep each step backward-compatible, and (c) sequence the rollout by dependency direction — not by who finished first.

## 8. Estate build sequencing (greenfield, phase by phase)

For teams standing the platform up from zero:

| Phase | Goal | Repos | Exit criteria |
|---|---|---|---|
| **P0 — Platform skeleton** | compose up gateway + infra | `platform-outerloop` | `docker compose up` starts Postgres (4 DBs), RabbitMQ, MailHog, Adminer, gateway |
| **P1 — Identity + auth** | login works end-to-end | `svc-identity`, `hrms-web` | login → HS256 token → an authorized request to a protected route succeeds |
| **P2 — Core domains** | submit timesheets & expenses | `svc-time`, `svc-expense`, `hrms-web` | employee can create & submit both; events land in RabbitMQ (watch the UI) |
| **P3 — Workflow** | approvals + notifications | `svc-workflow`, `hrms-web` | submit → approval task → manager decision → decision event → state finalized; email in MailHog |
| **P4 — Event fan-out** | onboarding auto-provisioning | all | `POST /identity/employees` provisions time+expense profiles (Scenario 5) via events |
| **P5 — Scenario features** | S1, S3, S4 | targeted repos | each scenario's E2E test green |

There is no separate "prod path" phase — the prod mapping is conceptual only ([`repos/platform-outerloop.md §6`](./repos/platform-outerloop.md#6-how-this-maps-to-production-conceptual--not-built)). Each phase ends with `scripts/smoke.py` extended to cover the new capability.

## 9. Environment configuration

There is **one** environment to configure: local, via `compose/.env`. Every value is filled and uniform (`atlas`/`atlas`); a developer copies `.env.example` → `.env` and never touches per-service files. Representative variables:

| Variable | Value (local) | Prod equivalent (not built) |
|---|---|---|
| `DATABASE_URL` | `postgresql+psycopg://atlas:atlas@postgres:5432/<db>` | managed Postgres, secret |
| `AMQP_URL` | `amqp://atlas:atlas@rabbitmq:5672/` | managed broker, secret |
| `JWT_SECRET` | `atlas-local-development-secret-32` (shared HS256) | real secret; RS256 + JWKS at the edge |
| `SMTP_HOST` | `mailhog` | real SMTP relay |

The same container images are the unit of deploy everywhere — that independence is the enterprise lesson; only these values would change.

## 10. Quality gates (must pass to merge)

1. Lint + format (`ruff` / ESLint) and types (`mypy --strict` / `tsc`) clean.
2. Unit + contract tests green; web adds an a11y (axe) check on key screens.
3. `oasdiff` finds **no breaking change** without a new API major (the coordination gate).
4. Migration verified expand-only (no destructive op against the running version).
5. Generated web client in sync with the pinned contract.
6. Integration: `docker compose up` + `scripts/smoke.py` + scenario E2E green.

**Image registry:** GitHub Container Registry (`ghcr.io/<org>/<repo>`), tagged `:<sha>`. Local `docker compose up --build` builds from source instead of pulling. (Coverage thresholds and image scanning are welcome additions but not required for the teaching goal.)
