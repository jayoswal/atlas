# Atlas HRMS — Project Overview

> **Atlas** is a reference-grade, multi-repository HRMS platform focused on **Employee Time & Expense management**. It exists to teach senior engineers and students how large organizations build, coordinate, and ship features across a **polyrepo microservice estate** — where one product change deliberately fans out across a UI repo, several backend service repos, and a platform (DevOps) repo.

- **Status:** Specification baseline `v1.0` (2026-09-06)
- **Audience:** Senior Frontend, Backend, and DevOps engineers (implementers); engineering students (learners)
- **Domain:** HRMS → Time & Attendance + Expense Management
- **Topology:** 6 repositories — 1 UI, 4 backend services, 1 outerloop/platform

---

## 1. Why this project exists

Most teaching material models microservices inside a **single monorepo**, which hides the hardest part of real enterprise engineering: **cross-repository coordination**. In practice a large company owns dozens or hundreds of independently versioned, independently deployed repositories. A single feature — "require a manager to approve overtime beyond 10 hours" — is not one pull request. It is a *coordinated set* of pull requests across a UI repo, two or three service repos, and a platform repo, each with its own contract, migration, review, pipeline, and release.

Atlas makes that coordination the **primary learning objective**. Every scenario in [`SCENARIOS.md`](./SCENARIOS.md) is chosen so that the answer **cannot** live in one repo. The UI repo is **always** in the change set.

## 2. What "enterprise-grade" means here

"Enterprise-grade" here means the **coordination shape** a real platform team works in — *not* deep per-service infrastructure. Atlas keeps the practices that carry the multi-repo lesson and deliberately drops production hardening that would only add maintenance drag (see [`ARCHITECTURE.md §0`](./ARCHITECTURE.md#0-design-charter--what-this-project-optimizes-for)).

| Concern | What Atlas adopts (kept simple) |
|---|---|
| Repo strategy | Polyrepo (6 repos), independent build & deploy, SemVer per repo |
| Service boundaries | Domain-driven, **database-per-service** (one Postgres server, four DBs), no shared tables |
| Contracts | Contract-first: **OpenAPI** (REST) + **JSON-Schema events** in a shared registry, breaking-change gated |
| Sync comms | REST/JSON through a single **API Gateway** (Traefik, routing only) |
| Async comms | Event-driven over **RabbitMQ** topic exchanges; publish-after-commit; idempotent consumers |
| AuthN/Z | One **HS256 JWT** verified in each service; **RBAC** by role + ownership |
| Visibility | Correlation IDs + JSON logs, and a browser UI for every tool (RabbitMQ, Adminer, MailHog) |
| Delivery | GitHub Actions per repo (lint · test · build); contract-lint gate |
| Local parity | The whole estate up with one `docker compose` command |

*Production concerns intentionally left as one-line notes, never built: transactional outbox, dead-letter/retry, RS256/JWKS, refresh rotation, OpenTelemetry/Prometheus/Grafana, Kubernetes/Helm, object storage.*

## 3. The six repositories

| # | Repository | Role | Owner | Language / Stack | Owns |
|---|---|---|---|---|---|
| 1 | **`hrms-web`** | Web UI (SPA) | Frontend | React 18, TypeScript 5, Vite 5, Redux Toolkit + RTK Query, MUI 5 | Screens, client-side auth/session, API client generated from OpenAPI |
| 2 | **`svc-identity`** | Identity & Org | Backend | Python 3.12, FastAPI, SQLAlchemy 2, Alembic | Employees, org hierarchy, roles/RBAC, cost centers, JWT issuance |
| 3 | **`svc-time`** | Time & Attendance | Backend | Python 3.12, FastAPI, SQLAlchemy 2, Alembic | Timesheets, clock in/out, PTO/leave, accrual balances |
| 4 | **`svc-expense`** | Expense | Backend | Python 3.12, FastAPI, SQLAlchemy 2, Alembic | Expense reports, line items, receipts, categories, FX, reimbursement |
| 5 | **`svc-workflow`** | Workflow & Notifications | Backend | Python 3.12, FastAPI, SQLAlchemy 2, Alembic | Approval routing engine, policy engine, notifications |
| 6 | **`platform-outerloop`** | Platform / DevOps | DevOps | Traefik, docker-compose, GitHub Actions | Gateway routing, the one compose that runs the estate, CI, contract registry, seed/smoke |

> **Naming note:** "outerloop" refers to the DevOps **outer loop** (build → integrate → deploy → operate), as distinct from a developer's inner loop (edit → run → test). The platform repo owns everything in the outer loop.

## 4. System context (C4 Level 1)

```mermaid
graph LR
  subgraph Actors
    EMP[Employee]
    MGR[Manager]
    FIN[Finance / Admin]
  end

  EMP --> WEB
  MGR --> WEB
  FIN --> WEB

  WEB[hrms-web SPA] -->|HTTPS REST /api/v1| GW[API Gateway - Traefik]

  GW --> ID[svc-identity]
  GW --> TIME[svc-time]
  GW --> EXP[svc-expense]
  GW --> WF[svc-workflow]

  ID -. employee.* .-> MQ[(RabbitMQ)]
  TIME -. timesheet.* .-> MQ
  EXP -. expense.* .-> MQ
  MQ -. events .-> WF
  MQ -. employee.* .-> TIME
  MQ -. employee.* .-> EXP
  WF -. approval.*, notification.* .-> MQ

  ID --> IDDB[(identity_db)]
  TIME --> TIMEDB[(time_db)]
  EXP --> EXPDB[(expense_db)]
  WF --> WFDB[(workflow_db)]
  WF --> MAIL[MailHog SMTP]
```

Full container diagram, event catalog, and cross-cutting standards live in [`ARCHITECTURE.md`](./ARCHITECTURE.md).

## 5. Domain model (bounded contexts)

Each service is a **bounded context** with an anti-corruption boundary; it never reads another service's database. Employee identity is replicated into consuming services as a **read model** kept fresh by `employee.*` events (eventual consistency).

- **Identity & Org** — the source of truth for *who* someone is, *where* they sit in the org, and *what* they may do. Issues JWTs.
- **Time & Attendance** — *when* people work and take leave; owns accrual math.
- **Expense** — *what* people spend and how it is reimbursed; owns money/currency.
- **Workflow & Notifications** — *how* submissions get approved and *who* is told; owns policy + routing, decoupled from the domains it serves.

## 6. Teaching methodology

Learners follow a fixed loop per scenario:

1. **Read the user story** in [`SCENARIOS.md`](./SCENARIOS.md).
2. **Predict** which repos change before revealing the answer.
3. **Trace the contract**: which OpenAPI paths / event schemas change, and who is producer vs. consumer.
4. **Order the rollout**: migrations first, backward-compatible producer next, consumer, then UI (never break a running client).
5. **Implement** the coordinated PRs and run the whole estate locally to see it end-to-end.

The recurring lesson: **a contract is the unit of coordination**, and **deploy order is dictated by dependency direction**, not by which team finishes first.

## 7. Running the whole thing locally

Plain commands — no Makefile (details in [`repos/platform-outerloop.md`](./repos/platform-outerloop.md)):

```bash
# from platform-outerloop/compose/
docker compose up --build -d                 # whole estate: gateway, 4 services, 1 Postgres,
                                             # RabbitMQ, MailHog, Adminer
uv run python ../scripts/seed.py             # load demo data (idempotent)
uv run python ../scripts/smoke.py            # cross-service smoke test
```

**Everything has a local UI to explore** — the app on `:8080`, plus Adminer (databases) `:8081`, RabbitMQ `:15672`, MailHog `:8025`, Traefik routing `:8080/dashboard/`. Config is one `compose/.env` file; demo logins are all `<name>@atlas.dev` / `atlas`. The same container images are the unit of deploy everywhere — that independence is the enterprise lesson; only configuration changes between local and prod.

## 8. Glossary

| Term | Meaning |
|---|---|
| **Outer loop** | Build/integrate/deploy/operate lifecycle owned by `platform-outerloop`. |
| **Bounded context** | A service's independent domain model and database. |
| **Read model** | A local copy of another context's data, upserted from events; rebuildable by replay — no shared database. |
| **Correlation ID** | Request-scoped ID propagated across services, logs, and events to follow one action. |
| **Contract-first** | The OpenAPI / event schema is authored and reviewed before code and is the source of truth. |

## 9. Document map

- [`ARCHITECTURE.md`](./ARCHITECTURE.md) — system architecture, standards, event catalog.
- [`DEVELOPMENT.md`](./DEVELOPMENT.md) — implementation guide: setup, coding standards, git/PR/CI, build sequencing, quality gates.
- [`DATA_CONTRACTS.md`](./DATA_CONTRACTS.md) — field-level entity schemas, event payloads, request/response examples, error registry.
- [`RUNBOOK.md`](./RUNBOOK.md) — the concrete starter files: docker-compose, DB init, Dockerfile, and the copied `auth.py` / `events.py`.
- [`SCENARIOS.md`](./SCENARIOS.md) — the cross-repo feature walkthroughs (core teaching material).
- `repos/hrms-web.md` — frontend engineer spec.
- `repos/svc-identity.md`, `repos/svc-time.md`, `repos/svc-expense.md`, `repos/svc-workflow.md` — backend engineer specs.
- `repos/platform-outerloop.md` — DevOps engineer spec.
