# Product Requirements Document (PRD): Generic Python Clean Architecture Boilerplate

## 1. Overview
This document defines the design and standards for a Generic Python Clean Architecture Base Project. The boilerplate provides a reusable, modular, and scalable foundation for future backend services, so teams don't need to re-architect from scratch for every new project, improving development velocity and cross-project consistency.

## 2. Goals & Objectives
- **Standardization**: Enforce a consistent architectural pattern across services to reduce onboarding friction between teams.
- **Maintainability**: Decouple business logic from infrastructure concerns (HTTP framework, database drivers), keeping the codebase readable, testable, and easy to maintain.
- **Extensibility**: Allow new features or transport layers (e.g., gRPC, GraphQL) to be added, or underlying libraries swapped (e.g., MySQL → PostgreSQL), without breaking changes to the core business logic.

## 3. Technology Stack
- **Language**: Python (minimum 3.11+)
- **HTTP Framework**: FastAPI (async-first, native OpenAPI generation, built-in dependency injection) or Flask with Blueprints for simpler synchronous services.
- **Database**:
  - Relational (core, always present): MySQL / PostgreSQL via `asyncpg` or `psycopg3`, raw SQL with `databases` library or `SQLAlchemy Core` (no ORM/Active Record pattern — see 6.1)
  - Caching (core, always present): Redis via `redis-py` async client (`redis.asyncio`)
  - Document store (optional, scaffolded on demand): MongoDB via `motor` (async)
  - Search/analytics (optional, scaffolded on demand): Elasticsearch via `elasticsearch-py` (async)
- **Dependency Management**: `uv` (preferred for speed and lockfile determinism) or `pip` + `pip-tools` with `pyproject.toml`
- **Configuration**: `pydantic-settings` for typed, validated config loaded from `.env` and environment variables — no raw `os.environ` access scattered across the codebase.
- **Type Checking**: `mypy` or `pyright` enforced in CI — Python's dynamic typing is not a license to skip type annotations.

## 4. Architecture Design (Clean Architecture)
The application is structured into 4 concentric layers, enforcing the **Dependency Rule**: source code dependencies must point inward only, from outer layers toward inner layers. Inner layers expose abstract interfaces (via Python's `abc.ABC` or `typing.Protocol`); outer layers provide concrete implementations (Dependency Inversion Principle).

1. **Domain Layer** (`app/domain/`) — The innermost layer, framework-agnostic and persistence-agnostic. Contains business entities (Python `dataclass` or Pydantic `BaseModel` with no ORM coupling) and abstract interface contracts (ports) that outer layers must implement — including repository interfaces and usecase interfaces. Must not import from any other layer or any third-party framework.

2. **Repository Layer** (`app/repositories/`) — The data access layer (adapter). Implements the repository abstract classes declared in the Domain layer, handling persistence concerns for external storage (RDBMS, NoSQL, cache). Translates storage-specific errors/types into domain-level exceptions.

3. **Usecase Layer** (`app/usecases/`) — The application/business logic layer. Orchestrates one or more repositories to fulfill a business operation, applying domain rules and invariants. Must remain transport-agnostic — no HTTP `Request`/`Response` objects, status codes, or framework-specific constructs leak into this layer.

4. **Presentation / Handler Layer** (`app/handlers/`) — The outermost layer (delivery mechanism / adapter). Owns request parsing and response serialization: validates incoming requests via DTOs (Pydantic schemas in `app/dto/`), invokes the corresponding usecase, and maps the result into the standardized HTTP JSON response.

## 5. Folder Structure Standard

```
base-project/
├── app/                        # Core application code
│   ├── domain/                 # Entities and Abstract Interface contracts (ports)
│   ├── dto/                    # Pydantic schemas for request validation and response serialization
│   ├── handlers/               # Route/endpoint controllers (e.g., user_handler.py)
│   ├── repositories/           # Persistence adapters: DB/cache implementations (e.g., user_repository.py)
│   ├── clients/                # Outbound adapters: third-party/external API consumers (e.g., payment_client.py)
│   ├── usecases/               # Business logic implementations (e.g., user_usecase.py)
│   └── validation/             # Custom validators and reusable validation helpers
├── consumers/                  # Long-running message consumer processes (Kafka/RabbitMQ, on demand)
├── config/                     # Centralized configuration (pydantic-settings classes)
├── middleware/                 # HTTP middleware/interceptors (auth, logging, rate limiting)
├── errs/                       # Custom application exception definitions
├── httputil/                   # Standard response builders (Success/Error envelope helpers)
├── ctxutil/                    # ContextVar helpers for request-scoped value propagation
├── migrations/                 # Alembic migration files
├── logs/                       # Log output directory (git-ignored, auto-created on startup)
├── tests/                      # Test suite mirroring app/ structure
│   ├── unit/
│   └── integration/
├── cli.py                      # CLI entry point (Typer commands)
├── main.py                     # Application factory and composition root
├── pyproject.toml              # Project metadata and dependencies
├── .env.example                # Environment variable template
├── Dockerfile
├── docker-compose.yml
└── .gitignore
```

**`app/clients/` — consuming external/third-party APIs**

`repositories/` and `clients/` are architecturally identical — both are **adapters** implementing an abstract class (port) declared in the Domain layer. They are split into separate folders purely for *semantic clarity*:

- `repositories/` — adapters for **this application's own data** (DB, cache, document store, search index). The application owns the schema/data model.
- `clients/` — adapters for **someone else's service** (payment gateway, SMS/email provider, another microservice's API, third-party identity provider). The application conforms to the external contract, not the other way around.

`clients/` typically carries concerns that `repositories/` doesn't: HTTP timeouts/retries with backoff, circuit breaking, request signing/auth headers, and mapping from the third party's response schema into the domain's own entity.

```python
# app/domain/payment_gateway.py — port, defined in domain layer
from abc import ABC, abstractmethod
from app.domain.entities import ChargeRequest, ChargeResult

class PaymentGateway(ABC):
    @abstractmethod
    async def charge(self, req: ChargeRequest) -> ChargeResult: ...

# app/clients/stripe_client.py — adapter, implements the port
import httpx
from app.domain.payment_gateway import PaymentGateway
from app.domain.entities import ChargeRequest, ChargeResult

class StripeClient(PaymentGateway):
    def __init__(self, api_key: str, http_client: httpx.AsyncClient) -> None:
        self._api_key = api_key
        self._http_client = http_client

    async def charge(self, req: ChargeRequest) -> ChargeResult:
        # call Stripe API, map response -> ChargeResult
        ...
```

The usecase layer depends only on `PaymentGateway`, never on `StripeClient` directly.

## 6. Development Rules & Guidelines

### 6.1. Dependency Injection (DI)
Every handler and usecase must declare its dependencies in `__init__` (constructor injection). FastAPI's built-in `Depends()` is the wiring mechanism for handlers; usecases are composed at the composition root (`main.py`).

```python
# app/usecases/user_usecase.py
class UserUsecase:
    def __init__(self, repo: UserRepository) -> None:
        self._repo = repo

# app/handlers/user_handler.py
def get_user_usecase() -> UserUsecase:
    return user_usecase  # resolved from composition root

@router.get("/users/{id}")
async def get_user(id: str, usecase: UserUsecase = Depends(get_user_usecase)):
    ...
```

Global module-level state and singleton instances directly accessing infrastructure resources are prohibited outside the composition root. For larger projects, consider `dependency-injector` to manage the wiring graph.

No ORM / Active Record pattern: domain entities must not inherit from any framework model class (e.g., no `SQLAlchemy.Model`, no Django `Model`). Keep the domain layer free of infrastructure coupling.

### 6.2. DTO (Data Transfer Object)
Domain entities must never be directly exposed as the API request/response schema. Define dedicated Pydantic models in `app/dto/` for request validation and response serialization, keeping the API contract decoupled from internal data structures.

- **Request DTOs**: inherit from `pydantic.BaseModel` with field validators. FastAPI automatically returns HTTP 422 on schema mismatch.
- **Response DTOs**: explicit, never expose internal domain fields (e.g., hashed passwords) directly.
- **Domain entities**: use Python `dataclass` or `pydantic.BaseModel` with `model_config = ConfigDict(from_attributes=False)` to prevent accidental ORM coupling.

### 6.3. Response Standardization
All API responses must conform to a single envelope schema, implemented in the `httputil/` module, with a distinct shape for success vs. error responses.

**Success response**

```json
{
  "code": "00",
  "message": "Success",
  "data": { ... },
  "meta": { ... }
}
```

**Error response**

```json
{
  "code": "40001",
  "message": "Validation failed",
  "errors": [
    { "field": "email", "message": "must be a valid email address" },
    { "field": "password", "message": "must be at least 8 characters" }
  ],
  "request_id": "01J8X9Z3K7Q2N4P5R6S7T8U9V0",
  "meta": null
}
```

Conventions:
- `data` is `None` on error responses — never mix partial success data with an error payload.
- `errors` is a list, present only for multi-field validation failures (HTTP 422). For single errors, omit `errors` and rely on `code`/`message`.
- `code` is an **internal application error code** (not the HTTP status code), defined as constants in `errs/` — e.g., `40001` = validation, `40401` = not found, `50001` = internal error.
- `request_id` is always included on error responses, sourced from `ContextVar` (7.17).
- HTTP status code still follows standard semantics — the envelope `code` is a supplement, not a replacement.
- Never leak raw exception details or stack traces in `message` for 5xx errors.

```python
# httputil/response.py
from pydantic import BaseModel
from typing import Any

class FieldError(BaseModel):
    field: str
    message: str

class SuccessResponse(BaseModel):
    code: str = "00"
    message: str = "Success"
    data: Any = None
    meta: Any = None

class ErrorResponse(BaseModel):
    code: str
    message: str
    errors: list[FieldError] | None = None
    request_id: str
    meta: Any = None
```

### 6.4. Error Handling
Define a hierarchy of custom exceptions in `errs/` that map to specific HTTP responses. Register a global exception handler at the framework level (FastAPI `@app.exception_handler`) to catch `AppException` and serialize it into the standard `ErrorResponse` envelope (6.3).

```python
# errs/exceptions.py
class AppException(Exception):
    def __init__(self, code: str, message: str, http_status: int) -> None:
        self.code = code
        self.message = message
        self.http_status = http_status

class NotFoundException(AppException):
    def __init__(self, message: str = "Resource not found") -> None:
        super().__init__(code="40401", message=message, http_status=404)

class UnauthorizedException(AppException):
    def __init__(self, message: str = "Unauthorized") -> None:
        super().__init__(code="40101", message=message, http_status=401)
```

Repository-layer database exceptions are caught and re-raised as domain-level `AppException` subclasses — a raw `asyncpg.PostgresError` must never propagate past the repository layer.

---

## 7. Suggested Additions

### 7.1. Testing Strategy
- Use `pytest` with `pytest-asyncio` for async test support.
- Directory structure mirrors `app/`: `tests/unit/usecases/`, `tests/unit/repositories/`, etc.
- Mock external dependencies using `unittest.mock.AsyncMock` or `pytest-mock` so usecases can be unit-tested without a real database.
- Convention: every `*_usecase.py` has a corresponding `test_*_usecase.py` using `@pytest.mark.parametrize`-driven test cases.
- Integration tests under `tests/integration/` using `testcontainers-python` (spins up real DB/Redis containers), gated by a `pytest` marker (`@pytest.mark.integration`) and excluded from the default run.
- Coverage enforced via `pytest-cov` with a minimum threshold defined in `pyproject.toml`.

### 7.2. Database Migration
- Use **Alembic** for schema migration management.
- Migration files stored in `migrations/versions/`.
- CLI commands wired through Typer (7.13): `app migrate upgrade`, `app migrate downgrade`, `app migrate current`.
- Separate `seeds/` directory for test/dummy data, invoked via `app seed`.
- Never auto-generate migration files from models without reviewing the generated SQL — every migration file is a code review artifact.

### 7.3. Logging & Observability
- Use `structlog` (preferred for structured, context-aware logging) or Python's `logging` module with a JSON formatter (`python-json-logger`).
- Inject `request_id`/`correlation_id` into every log entry within a request's lifecycle via middleware (7.17).
- Expose health check endpoints `/healthz` and `/readyz` for liveness/readiness probes (7.19).
- Optional: integrate distributed tracing (OpenTelemetry via `opentelemetry-sdk`) and metrics (Prometheus via `prometheus-client`).

**Default Log Configuration**

- **Storage**: logs written to `logs/` directory, auto-created on startup:

```python
import os
os.makedirs("logs", exist_ok=True)
```

- **File naming**: `{app_name}_{yyyy-mm-dd}.log` (e.g., `base-project_2026-06-21.log`), where `app_name` is read from `settings.APP_NAME`.
- **Rotation**: daily rotation with 7-day retention via Python's built-in `TimedRotatingFileHandler` (`when="midnight"`, `backupCount=7`).
- **Dual output**: configure both a `StreamHandler(sys.stdout)` and a `TimedRotatingFileHandler` on the root logger simultaneously — terminal and file output remain identical.

```python
import logging, sys, os
from logging.handlers import TimedRotatingFileHandler
from datetime import datetime

def configure_logging(app_name: str) -> None:
    os.makedirs("logs", exist_ok=True)
    log_filename = f"logs/{app_name}_{datetime.now().strftime('%Y-%m-%d')}.log"

    file_handler = TimedRotatingFileHandler(
        filename=log_filename, when="midnight", backupCount=7, encoding="utf-8"
    )
    stdout_handler = logging.StreamHandler(sys.stdout)
    fmt = logging.Formatter("%(asctime)s %(levelname)s %(name)s %(message)s")
    file_handler.setFormatter(fmt)
    stdout_handler.setFormatter(fmt)

    logging.basicConfig(level=logging.INFO, handlers=[file_handler, stdout_handler])
```

- **Git exclusion**: add `logs/` to `.gitignore`. Add `.gitkeep` inside `logs/` if the folder structure needs to be tracked.

### 7.4. Validation Layer
- FastAPI + Pydantic handles schema validation automatically, returning HTTP 422 with field-level detail on mismatch.
- Use `@field_validator` and `@model_validator` inside `app/dto/` for business-rule validation (e.g., "start_date must be before end_date").
- Extract reusable validators into `app/validation/` as standalone functions/classes shared across multiple DTOs.
- Normalize Pydantic's `ValidationError` into the standard `ErrorResponse` envelope (6.3) via the global exception handler (6.4).

### 7.5. Context Propagation
Python's equivalent of Go's `context.Context` is `contextvars.ContextVar` — use it to propagate per-request values (request-id, actor-id, trace span) without threading them explicitly through every function call.

For `asyncio`-based code (FastAPI), `ContextVar` values are automatically inherited by child coroutines — the framework resets the context per request, preventing cross-request leakage.

Provide a `ctxutil/` module with typed getter/setter helpers:

```python
# ctxutil/context.py
from contextvars import ContextVar

_request_id: ContextVar[str] = ContextVar("request_id", default="")
_actor_id: ContextVar[str] = ContextVar("actor_id", default="")

def set_request_id(value: str) -> None: _request_id.set(value)
def get_request_id() -> str: return _request_id.get()

def set_actor_id(value: str) -> None: _actor_id.set(value)
def get_actor_id() -> str: return _actor_id.get()
```

Set timeouts at the entry point via `asyncio.wait_for()` or `anyio.move_on_after()` — don't rely on the HTTP framework's default timeout alone.

### 7.6. Transaction Management
Define a `UnitOfWork` abstract class in the domain layer (port); the concrete implementation wraps the actual DB connection/transaction. The usecase controls `commit`/`rollback` without knowing which DB driver is underneath.

```python
# app/domain/unit_of_work.py
from abc import ABC, abstractmethod

class UnitOfWork(ABC):
    @abstractmethod
    async def __aenter__(self) -> "UnitOfWork": ...
    @abstractmethod
    async def __aexit__(self, *args) -> None: ...
    @abstractmethod
    async def commit(self) -> None: ...
    @abstractmethod
    async def rollback(self) -> None: ...

# Usage in usecase — controls transaction without knowing the DB driver
async def create_order(self, req: CreateOrderRequest) -> Order:
    async with self._uow as uow:
        order = await self._order_repo.create(req)
        await self._payment_repo.reserve(order.id, req.amount)
        await uow.commit()
        return order
```

### 7.7. Authentication & Authorization Hook
Auth middleware must be generic and pluggable (an abstract `TokenValidator` class in the domain layer) — derivative projects plug in opaque token validation, JWT verification, or API key checks without modifying the middleware itself.

```python
# app/domain/auth.py
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class AuthContext:
    actor_id: str
    scopes: list[str]

class TokenValidator(ABC):
    @abstractmethod
    async def validate(self, token: str) -> AuthContext: ...

# middleware/auth.py — FastAPI security dependency
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

async def require_auth(
    credentials: HTTPAuthorizationCredentials = Security(HTTPBearer()),
    validator: TokenValidator = Depends(get_token_validator),
) -> AuthContext:
    ctx = await validator.validate(credentials.credentials)
    ctxutil.set_actor_id(ctx.actor_id)
    return ctx
```

`AuthContext` is stored in `ContextVar` via `ctxutil` (7.5) and consumed by the audit trail (7.16) without being passed explicitly through every function.

### 7.8. Graceful Shutdown
Use FastAPI's lifespan context manager for startup/shutdown lifecycle:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    await startup()   # init DB pool, Redis client, etc.
    yield
    await shutdown()  # close all connections gracefully

app = FastAPI(lifespan=lifespan)
```

Uvicorn drains in-flight requests before triggering the shutdown phase on `SIGTERM`/`SIGINT`. For worker/consumer processes, install signal handlers explicitly via `signal.signal(signal.SIGTERM, ...)` to stop pulling new messages and finish the current one before exiting.

### 7.9. Linting & CI
- **`ruff`**: unified linter and formatter (replaces `flake8`, `isort`, `black`) — single config in `pyproject.toml`, significantly faster.
- **`mypy`** or **`pyright`**: static type checking enforced in CI.
- Enforce cognitive complexity via `ruff`'s `C901` rule (`max-complexity = 10`).
- Example CI pipeline (GitHub Actions): lint → type-check → test → build Docker image.

```toml
# pyproject.toml
[tool.ruff]
line-length = 100
select = ["E", "F", "I", "C901"]

[tool.ruff.mccabe]
max-complexity = 10
```

### 7.10. API Documentation
- FastAPI generates OpenAPI/Swagger docs automatically from route definitions and Pydantic schemas — no additional annotation library needed.
- Configure `title`, `version`, `description`, and `servers` on the `FastAPI()` instance from `pydantic-settings` (7.19.1).
- Disable `/docs` and `/redoc` in production (`docs_url=None, redoc_url=None`) if public API docs are not desired.
- For Flask-based projects, use `flask-openapi3` or `flasgger` with manual schema annotations.

### 7.11. Rate Limiting & Security Headers
- **Rate limiting**: `slowapi` (integrates with FastAPI, wraps the `limits` library) backed by Redis for distributed rate limiting across multiple instances.
- **Security headers**: `secure.py` or a custom `BaseHTTPMiddleware` adding `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security`, `Content-Security-Policy` — toggled via `settings.SECURITY_HEADERS_ENABLED`.
- **CORS**: FastAPI's built-in `CORSMiddleware` with allowed origins/methods read from `pydantic-settings`.

### 7.12. Versioning Strategy
- Define an API versioning strategy upfront — URL path prefix (`/v1/`), header-based (`Accept-Version`), or query parameter — and keep it consistent across all derivative projects.
- FastAPI supports prefix-based versioning cleanly via `APIRouter(prefix="/v1")` mounted at the app level.

### 7.13. CLI Tooling with Typer
Use [`typer`](https://typer.tiangolo.com/) (built on `click`, type-annotation-driven) to expose operational commands as subcommands of a single entry point:

```
cli.py
commands/
├── serve.py        # `app serve` — starts the HTTP server via uvicorn programmatically
├── migrate.py      # `app migrate upgrade|downgrade|current` — wraps Alembic
├── seed.py         # `app seed` — runs database seeders
└── worker.py       # `app worker kafka|rabbitmq` — starts background consumer
```

```python
# cli.py
import typer
from commands import serve, migrate, seed, worker

app = typer.Typer()
app.add_typer(serve.app, name="serve")
app.add_typer(migrate.app, name="migrate")
app.add_typer(seed.app, name="seed")
app.add_typer(worker.app, name="worker")

if __name__ == "__main__":
    app()
```

- Each subcommand reuses the same composition root (config loading, DB/Redis setup) as the HTTP server — no duplicated bootstrap logic.
- Global options (`--env`, `--config-path`) registered as Typer `callback` parameters at the root app level.
- Pairs naturally with `pydantic-settings` for config resolution (env files, environment variables, CLI overrides through a single source of truth).

### 7.14. Containerization with Docker
The base project ships container-ready by default.

```dockerfile
# ---- builder stage ----
FROM python:3.11-slim AS builder
WORKDIR /app
RUN pip install uv
COPY pyproject.toml .
RUN uv pip install --system --no-cache .

# ---- runtime stage ----
FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /usr/local/lib/python3.11 /usr/local/lib/python3.11
COPY . .
ENTRYPOINT ["python", "cli.py", "serve"]
```

- **`docker-compose.yml`** for local development: app service + MySQL/PostgreSQL + Redis, with health checks and volume mounts — single `docker compose up` to run the full stack.
- **`.dockerignore`**: exclude `.git`, `logs/`, `__pycache__`, `*.pyc`, `.env`, `tests/` to keep build context minimal.
- 12-factor compliance: configuration via environment variables only, logs to stdout/stderr, no secrets baked into the image.
- The `app migrate` and `app seed` Typer subcommands (7.13) can be invoked as one-off containers in CI/CD or as Kubernetes Jobs/init containers, reusing the same image as the running service.

### 7.15. Query Debug Logging
Toggle raw SQL query logging via configuration — no code changes required.

- **Config flag**: `DB_DEBUG=true` read from `.env`/environment variables at startup.
- When enabled, every executed query — including bound parameters and execution duration — is printed to the configured logger (respecting the dual stdout+file output from 7.3).
- When disabled (default in production), query logging is fully skipped to avoid leaking sensitive data.

```python
# For raw asyncpg/psycopg3 — wrap at the repository level
async def _execute(self, query: str, *args) -> list:
    if self._settings.DB_DEBUG:
        start = time.monotonic()
        result = await self._conn.fetch(query, *args)
        logger.debug("query",
            query=query, args=args,
            duration_ms=round((time.monotonic() - start) * 1000, 2),
        )
        return result
    return await self._conn.fetch(query, *args)

# For SQLAlchemy Core — built-in echo flag
engine = create_async_engine(settings.DATABASE_URL, echo=settings.DB_DEBUG)
```

- **Security**: `DB_DEBUG` must always default to `False` — bound parameters may contain passwords, tokens, or PII.

### 7.16. Audit Trail (Optional)
Audit trail is **not a requirement for every derivative project** — enable it only for domains where "who did what, when" carries compliance or accountability value. For simple CRUD services, skip it entirely.

Design as an **opt-in module**: inject `NoopAuditLogger` by default; derivative projects that need it swap in `DatabaseAuditLogger` at the composition root with zero changes to usecase code.

```python
# app/domain/audit_logger.py
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class AuditEntry:
    actor_id: str
    action: str
    resource_type: str
    resource_id: str | None = None
    before: dict | None = None
    after: dict | None = None

class AuditLogger(ABC):
    @abstractmethod
    async def record(self, entry: AuditEntry) -> None: ...

class NoopAuditLogger(AuditLogger):
    async def record(self, entry: AuditEntry) -> None:
        pass  # opt-out: inject this for projects that don't need audit trail
```

The `AuditLogger.record()` implementation automatically reads `request_id` and `actor_id` from `ContextVar` (7.5/7.17) — no explicit parameter threading needed. Audit writes must participate in the same DB transaction as the business mutation (UnitOfWork from 7.6). Never store raw secrets/passwords/tokens in `before`/`after` snapshots.

**Operational considerations**
- Audit table is insert-only at the application level — `UPDATE`/`DELETE` on `audit_logs` are never exposed as usecases.
- Retain far longer than operational logs (compliance-driven, often 1+ years) — separate retention policy from the 7-day log rotation in 7.3.

### 7.17. Request ID / Correlation ID for Tracing
Every request carries a unique identifier from the moment it enters the system — correlating application logs, query debug logs (7.15), and audit trail entries (7.16) into a single traceable thread.

**Generation & propagation**

```python
# middleware/request_id.py
import uuid
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from ctxutil.context import set_request_id

class RequestIDMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        req_id = request.headers.get("X-Request-ID") or str(uuid.uuid4())
        set_request_id(req_id)
        response = await call_next(request)
        response.headers["X-Request-ID"] = req_id
        return response
```

- Use UUID v4 by default, or `python-ulid` for time-sortable IDs (easier log inspection).
- Reuse incoming `X-Request-ID` header if present (from upstream gateway/load balancer); otherwise generate fresh.
- Echo `request_id` back in the response header so clients can reference it when reporting issues.

**Wiring into logs and audit trail**
- The structured logger reads `get_request_id()` from `ContextVar` and attaches it to every log line — no function-argument threading needed.
- Query debug logs (7.15) inherit the same context, so a slow/suspicious query traces back to the triggering request.
- `AuditLogger.record()` reads `request_id` from context and stores it in the `audit_logs` table — given a `request_id` from a user complaint, an engineer can grep logs *and* query `SELECT * FROM audit_logs WHERE request_id = ?` for the complete picture.
- For async background tasks, capture context before scheduling: `ctx = contextvars.copy_context()` then `ctx.run(task_fn)` — preserves `request_id` in the detached coroutine.
- **Note on distributed tracing**: if OpenTelemetry is adopted later, align `request_id` with the OTel `trace_id` — avoid maintaining two parallel, unrelated identifiers.

### 7.18. Redis Caching, MongoDB & Elasticsearch (Core vs. On-Demand)

- **Redis (core, always wired)** — included in `config/` and `docker-compose.yml` by default.
- **MongoDB & Elasticsearch (optional, generated on demand)** — not wired by default; scaffolded via `app generate datastore mongo` / `app generate datastore elasticsearch`.

**7.18.1 Redis — Caching**

- **Connection**: `redis.asyncio.from_url(settings.REDIS_URL)` initialized in the composition root.
- **Port/adapter**: `domain.Cache` abstract class in the domain layer; implementation in `app/repositories/cache/redis_cache.py`. Usecases depend on the abstract class, never on the Redis client directly.
- **Cache-aside pattern**: check cache → miss → fetch from DB → populate cache — implemented at the usecase layer since cache invalidation is a business decision.
- **Key naming**: `{domain}:{entity}:{id}` (e.g., `app:user:42`), TTL defined in config.
- **Health check**: Redis `PING` included in `/readyz` probe (7.19).

```python
# app/domain/cache.py
from abc import ABC, abstractmethod
from typing import Any

class Cache(ABC):
    @abstractmethod
    async def get(self, key: str) -> str | None: ...
    @abstractmethod
    async def set(self, key: str, value: Any, ttl_seconds: int) -> None: ...
    @abstractmethod
    async def delete(self, key: str) -> None: ...
    @abstractmethod
    async def ping(self) -> bool: ...
```

**7.18.2 MongoDB — Generated On Demand**

When scaffolded: `app/repositories/mongo/` using `motor` (async driver), implementing the relevant domain repository abstract class. Connection setup (`MONGO_URI`, `MONGO_DB_NAME`) added to `config/` and `docker-compose.yml` only for that project. An `ensure_indexes()` coroutine called once at startup handles index management.

**7.18.3 Elasticsearch — Generated On Demand**

When scaffolded: `app/repositories/search/` using `elasticsearch-py` async client, implementing a `domain.SearchRepository` abstract class. Index mappings stored as JSON files under `app/repositories/search/mappings/`, applied via `app search reindex` CLI subcommand. Sync strategy chosen per project: dual-write (simple) or outbox/event-driven (recommended for correctness).

### 7.19. Application Versioning & Health Check

**7.19.1 Application Versioning**

- Single source of truth: `version` field in `pyproject.toml` under `[project]`.
- Inject `git` commit hash and build timestamp as Docker build args at CI build time:

```dockerfile
ARG VERSION=dev
ARG COMMIT=none
ARG BUILD_TIME=unknown
ENV APP_VERSION=$VERSION APP_COMMIT=$COMMIT APP_BUILD_TIME=$BUILD_TIME
```

- Read in `pydantic-settings` and expose via:
  - `app version` Typer subcommand: prints version/commit/build-time to stdout.
  - `GET /version` endpoint returning the standard success envelope (6.3):

```json
{
  "code": "00",
  "message": "Success",
  "data": {
    "version": "v1.4.2",
    "commit": "a1b2c3d",
    "build_time": "2026-06-20T08:15:00Z"
  }
}
```

**7.19.2 Health Check**

- **`GET /healthz` (liveness)**: process alive — no downstream calls. Orchestrator uses this to decide restart.
- **`GET /readyz` (readiness)**: actively checks DB (`SELECT 1`), Redis (`PING`), and any optional stores in use, each under a short `asyncio.wait_for(..., timeout=2.0)`. Orchestrator/load balancer uses this to route traffic.

```python
@router.get("/healthz")
async def liveness():
    return {"status": "ok"}

@router.get("/readyz")
async def readiness(db=Depends(get_db), cache=Depends(get_cache)):
    checks, status_code = {}, 200
    try:
        await asyncio.wait_for(db.execute("SELECT 1"), timeout=2.0)
        checks["database"] = "up"
    except Exception:
        checks["database"] = "down"
        status_code = 503
    try:
        await asyncio.wait_for(cache.ping(), timeout=2.0)
        checks["redis"] = "up"
    except Exception:
        checks["redis"] = "down"
        status_code = 503

    return JSONResponse(status_code=status_code, content={
        "checks": checks,
        "version": settings.APP_VERSION,
        "commit": settings.APP_COMMIT,
    })
```

- Response format intentionally bypasses the standard envelope (6.3) — orchestrators expect a plain HTTP status code and minimal body.
- Include `version`/`commit` in `/readyz` for quick incident triage without a separate call to `/version`.
- Never expose connection strings, internal hostnames, or stack traces in health check responses.

### 7.20. Singleflight — Deduplicating Concurrent Identical Requests
When many concurrent requests ask for the same data simultaneously (cache miss, popular endpoint, TTL just expired), each independently hitting the DB causes a **thundering herd / cache stampede**. Singleflight collapses concurrent identical calls into a single in-flight execution, sharing the result across all callers.

Python has no built-in singleflight, but the pattern is implemented via `asyncio.Future` and a shared in-flight registry:

```python
# app/usecases/singleflight.py
import asyncio
from typing import Any, Callable, Awaitable

class AsyncSingleflight:
    def __init__(self) -> None:
        self._inflight: dict[str, asyncio.Future] = {}
        self._lock = asyncio.Lock()

    async def do(self, key: str, fn: Callable[[], Awaitable[Any]]) -> Any:
        async with self._lock:
            if key in self._inflight:
                return await asyncio.shield(self._inflight[key])
            future: asyncio.Future = asyncio.get_event_loop().create_future()
            self._inflight[key] = future
        try:
            result = await fn()
            future.set_result(result)
            return result
        except Exception as e:
            future.set_exception(e)
            raise
        finally:
            async with self._lock:
                self._inflight.pop(key, None)
```

Usage at the **usecase layer** (never inside the repository):

```python
class ProductUsecase:
    def __init__(self, repo: ProductRepository, cache: Cache) -> None:
        self._repo = repo
        self._cache = cache
        self._sf = AsyncSingleflight()

    async def get_product(self, product_id: str) -> Product:
        key = f"product:{product_id}"
        cached = await self._cache.get(key)
        if cached:
            return Product.model_validate_json(cached)

        async def fetch():
            product = await self._repo.find_by_id(product_id)
            await self._cache.set(key, product.model_dump_json(), ttl_seconds=300)
            return product

        return await self._sf.do(key, fetch)
```

**Key conventions**
- **Key must encode all differentiating parameters** (e.g., `product:{id}:{locale}`, not just `product:{id}`).
- **One `AsyncSingleflight` instance per usecase**, injected from the composition root — not a module-level global.
- **Error sharing**: all concurrent callers for the same key receive the same exception — don't retry inside `fn`.
- **Pairs with, doesn't replace, caching**: singleflight deduplicates only during the brief miss window; the cache handles steady-state.
- **Idempotent reads only** — never apply to mutations or commands with side effects.

### 7.21. Message Broker — Kafka & RabbitMQ (Generated On Demand)
**Not wired into the base project by default** — scaffolded only when a project explicitly needs event streaming or task queuing, via `app generate broker kafka` / `app generate broker rabbitmq`.

**Folder placement**:

```
app/
└── clients/
    └── messaging/
        ├── kafka_producer.py       # implements domain.EventPublisher
        └── rabbitmq_producer.py    # implements domain.EventPublisher
consumers/
├── kafka_consumer.py               # long-running consumer loop
└── rabbitmq_consumer.py
```

**Port/adapter pattern (producer side)**

```python
# app/domain/event_publisher.py — port
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class Event:
    id: str
    type: str
    payload: dict
    timestamp: datetime = field(default_factory=datetime.utcnow)

class EventPublisher(ABC):
    @abstractmethod
    async def publish(self, topic: str, event: Event) -> None: ...
```

**7.21.1 Kafka — when scaffolded**

- Client: `aiokafka` (pure Python async, no native library dependency).
- Config (project-only): `KAFKA_BOOTSTRAP_SERVERS`, `KAFKA_CONSUMER_GROUP`, `KAFKA_TOPIC_PREFIX`.
- Consumer runs as `app worker kafka` Typer subcommand — separate process from `app serve`, independent scaling.
- Each consumed message carries/generates a `request_id` set via `ctxutil.set_request_id()` before invoking the usecase — maintaining the full tracing chain from 7.17.
- Use Kafka for high-throughput event streaming, multiple independent consumers per topic, or offset-based replay.

**7.21.2 RabbitMQ — when scaffolded**

- Client: `aio-pika` (async AMQP client, actively maintained).
- Config (project-only): `RABBITMQ_URL`, `RABBITMQ_EXCHANGE`, `RABBITMQ_QUEUE`.
- Consumer runs as `app worker rabbitmq` subcommand — separate process.
- Use RabbitMQ for task/job queueing with acknowledgment semantics, or routing-based dispatch via exchanges/routing keys.

**Shared conventions regardless of broker**
- **Idempotency**: at-least-once delivery is the realistic default — usecases must be safely re-runnable (upsert, dedupe on `event.id`).
- **Dead-letter handling**: `{topic}.dlq` (Kafka) or DLX/dead-letter exchange (RabbitMQ) defined from the start — poison messages must not block the consumer loop.
- **Graceful shutdown**: `signal.signal(signal.SIGTERM, ...)` in consumer process — stop consuming, finish current message, exit cleanly.
- **Health check**: broker connectivity check added to `/readyz` (7.19.2) when a broker is scaffolded.
- **`request_id` propagation**: every consumed message carries a `request_id` in its headers or generates a fresh one — set via `ctxutil` before invoking the usecase, keeping the audit trail (7.16) correlatable across service/process boundaries.