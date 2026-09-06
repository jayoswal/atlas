# Atlas HRMS — Runbook (the concrete starter files)

> Zero-to-running wiring. These are the **actual files** that turn the specs into a working estate, so a new developer copies rather than invents. They are intentionally minimal — enough to run, nothing more (per [`ARCHITECTURE.md §0`](./ARCHITECTURE.md#0-design-charter--what-this-project-optimizes-for)). Adapt names/ports per service; the shape is identical.

- **Companions:** [`DEVELOPMENT.md`](./DEVELOPMENT.md) (coding standards), [`repos/platform-outerloop.md`](./repos/platform-outerloop.md) (owns these files), [`DATA_CONTRACTS.md`](./DATA_CONTRACTS.md).

---

## 1. `platform-outerloop/compose/docker-compose.yml`

One file runs everything. The four backend services are the same block with a different build path, `PORT`, `DATABASE_URL` db-name, and Traefik path rule.

```yaml
name: atlas
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - ./postgres-init.sql:/docker-entrypoint-initdb.d/init.sql:ro
      - pgdata:/var/lib/postgresql/data
    ports: ["5432:5432"]
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "${POSTGRES_USER}"]
      interval: 5s
      retries: 12

  rabbitmq:
    image: rabbitmq:3.13-management
    environment:
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_USER}
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD}
    ports: ["5672:5672", "15672:15672"]
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 5s
      retries: 12

  gateway:
    image: traefik:3
    command:
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
      - --api.dashboard=true
      - --api.insecure=true
    ports: ["8080:80"]
    volumes: ["/var/run/docker.sock:/var/run/docker.sock:ro"]

  svc-identity:
    build: ../../svc-identity
    environment:
      DATABASE_URL: postgresql+psycopg://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/identity_db
      AMQP_URL: amqp://${RABBITMQ_USER}:${RABBITMQ_PASSWORD}@rabbitmq:5672/
      JWT_SECRET: ${JWT_SECRET}
      JWT_TTL_HOURS: ${JWT_TTL_HOURS}
      DEMO_PASSWORD: ${DEMO_PASSWORD}
      PORT: "8001"
    depends_on:
      postgres: { condition: service_healthy }
      rabbitmq: { condition: service_healthy }
    labels:
      - traefik.enable=true
      - traefik.http.routers.identity.rule=PathPrefix(`/api/v1/auth`) || PathPrefix(`/api/v1/identity`)
      - traefik.http.services.identity.loadbalancer.server.port=8001

  # svc-time (PORT 8002, db time_db, rule PathPrefix(`/api/v1/time`))
  # svc-expense (PORT 8003, db expense_db, rule PathPrefix(`/api/v1/expense`))
  # svc-workflow (PORT 8004, db workflow_db, rule PathPrefix(`/api/v1/workflow`) || PathPrefix(`/api/v1/approvals`))
  #   svc-workflow also gets: SMTP_HOST=mailhog, SMTP_PORT=1025
  #   — identical block, only those four values change.

  hrms-web:
    build: ../../hrms-web
    depends_on: [gateway]
    labels:
      - traefik.enable=true
      - traefik.http.routers.web.rule=PathPrefix(`/`)
      - traefik.http.services.web.loadbalancer.server.port=80

  adminer:
    image: adminer:4
    ports: ["8081:8080"]

  mailhog:
    image: mailhog/mailhog
    ports: ["1025:1025", "8025:8025"]

volumes:
  pgdata: {}
```

**Routing is Traefik labels** (docker provider) — no separate `routes.yml` to maintain. The four path rules above are the entire gateway config.

## 2. `platform-outerloop/compose/postgres-init.sql`

Runs once on first boot to create the four databases in the single server.

```sql
CREATE DATABASE identity_db;
CREATE DATABASE time_db;
CREATE DATABASE expense_db;
CREATE DATABASE workflow_db;
```

(The `atlas` superuser owns all four; the per-service boundary is enforced in code — each service only ever opens its own `DATABASE_URL`.)

## 3. Backend `Dockerfile` (identical in every service)

```dockerfile
FROM python:3.12-slim
WORKDIR /app
RUN pip install --no-cache-dir uv
COPY pyproject.toml uv.lock* ./
RUN uv sync --no-dev
COPY . .
# migrate, then serve — one process (see §5 for the consumer)
CMD ["sh", "-c", "uv run alembic upgrade head && uv run uvicorn app.main:app --host 0.0.0.0 --port ${PORT}"]
```

Migrations run on every start (`alembic upgrade head` is a no-op when up to date), so there is no separate migrate command.

---

## 4. `app/core/auth.py` — the shared HS256 dependency (copied into each service)

Polyrepo reality: there is **no shared runtime library**. This ~20-line file is **copied verbatim** into each of the four services (the *contract* is the only thing that's shared, via the registry). It is small and stable on purpose.

```python
# app/core/auth.py
import jwt
from fastapi import Depends, Header, HTTPException
from .config import settings   # settings.jwt_secret from env JWT_SECRET

def current_user(authorization: str = Header(default="")) -> dict:
    if not authorization.startswith("Bearer "):
        raise HTTPException(401, {"error": {"code": "UNAUTHENTICATED", "message": "Missing bearer token"}})
    try:
        return jwt.decode(authorization[7:], settings.jwt_secret, algorithms=["HS256"])
    except jwt.PyJWTError:
        raise HTTPException(401, {"error": {"code": "UNAUTHENTICATED", "message": "Invalid token"}})

def require(*roles: str):
    """Route dependency: require any of these roles. Ownership checks live in the handler."""
    def dep(user: dict = Depends(current_user)) -> dict:
        if roles and not (set(roles) & set(user.get("roles", []))):
            raise HTTPException(403, {"error": {"code": "FORBIDDEN", "message": "Insufficient role"}})
        return user
    return dep
```

Usage in a router: `@router.get("/timesheets"); def list(user = Depends(require("EMPLOYEE","MANAGER"))): ...`

## 5. `app/events.py` — publish + consume (copied into each service)

Also copied per service (only the exchange name and the bindings differ). Publishing is a plain call after commit; consuming runs as a background task in the same process.

```python
# app/events.py
import json, uuid, asyncio, datetime as dt, aio_pika
from .config import settings   # settings.amqp_url from env AMQP_URL

# The producer passes its own exchange name, e.g. svc-identity publishes to "identity.events".
async def publish(exchange: str, event_type: str, data: dict, correlation_id: str) -> None:
    conn = await aio_pika.connect_robust(settings.amqp_url)
    async with conn:
        ch = await conn.channel()
        ex = await ch.declare_exchange(exchange, aio_pika.ExchangeType.TOPIC, durable=True)
        body = {"id": str(uuid.uuid4()), "type": event_type,
                "time": dt.datetime.utcnow().isoformat() + "Z",
                "correlation_id": correlation_id, "data": data}
        await ex.publish(
            aio_pika.Message(json.dumps(body).encode(), content_type="application/json"),
            routing_key=event_type,
        )

async def consume(bindings: list[tuple[str, str, list[str], "callable"]]) -> None:
    """bindings: (exchange, queue_name, [routing_keys], async handler(evt: dict))"""
    conn = await aio_pika.connect_robust(settings.amqp_url)
    ch = await conn.channel()
    await ch.set_qos(prefetch_count=20)
    for exchange, queue_name, keys, handler in bindings:
        ex = await ch.declare_exchange(exchange, aio_pika.ExchangeType.TOPIC, durable=True)
        q = await ch.declare_queue(queue_name, durable=True)
        for k in keys:
            await q.bind(ex, routing_key=k)
        async def _cb(msg: aio_pika.IncomingMessage, _h=handler):
            async with msg.process():          # ack on success; requeue on exception
                await _h(json.loads(msg.body))
        await q.consume(_cb)
    await asyncio.Future()                      # keep the task alive
```

**Process model (decided): one container, one process.** The consumer is started as a background task in FastAPI's lifespan — no separate worker container, no relay:

```python
# app/main.py (excerpt)
from contextlib import asynccontextmanager
from fastapi import FastAPI
from .events import consume
from .consumers import BINDINGS   # this service's (exchange, queue, keys, handler) list

@asynccontextmanager
async def lifespan(app: FastAPI):
    task = asyncio.create_task(consume(BINDINGS))
    yield
    task.cancel()

app = FastAPI(lifespan=lifespan)
```

A service that only produces events (`svc-identity`) passes `BINDINGS = []`. Consumers are **idempotent by upsert** (e.g. `INSERT ... ON CONFLICT (employee_id) DO UPDATE`), so a redelivered message is harmless — that is why no dedupe table is needed.

---

## 6. Frontend: Vite dev proxy

`hrms-web/vite.config.ts` proxies `/api` to the gateway so the browser and the API share an origin in dev:

```ts
server: { port: 3000, proxy: { "/api": "http://localhost:8080" } }
```

In the composed estate the gateway already serves both the SPA and `/api`, so no proxy is needed there — the proxy is only for `npm run dev` against a running estate.

## 7. What is NOT here (on purpose)

No Kubernetes manifests, Helm charts, observability collector, object storage, outbox tables, or dead-letter config. Those are prod notes in [`ARCHITECTURE.md`](./ARCHITECTURE.md), never built. This runbook is the whole runnable surface.
