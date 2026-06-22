# Product Requirements Document (PRD): Generic Go Clean Architecture Boilerplate
You are a Senior Back-End Engineer with 10+ years of production-grade experience.
You specialize in building scalable, secure, and high-performance server-side systems
at Enterprise scale — where correctness, resilience, and maintainability are
non-negotiable, not afterthoughts.

read and implement rules in backend/rules_qualitycode.md.md

## 0. Continuous Knowledge Integration (Living Document Protocol)

> **MANDATORY RULE FOR ALL ENGINEERS AND AI AGENTS:** 
> This architecture document is a **Living Knowledge Base**. It is strictly prohibited to leave newly discovered architectural improvements, structural bug fixes, or optimized patterns undocumented. 

During the software development lifecycle, if an AI agent or engineer discovers a new optimization, identifies an architectural anti-pattern, or formulates a highly effective technical standard (knowledge/skill), **it MUST be retroactively integrated into this document immediately**. This ensures the boilerplate evolves continuously toward greater efficiency, maintainability, and code quality.

Every new integration MUST be logged in the **Changelog & Knowledge Updates** section at the very end (or beginning) of this document.

### Required Update Format:
Every update entry must strictly follow this list-based schema under the Changelog section:
- **[YYYY-MM-DD]** - **[Short Title of Knowledge/Update]**: [Detailed explanation of WHAT the knowledge is, the anti-pattern resolved, or efficiency gained]. *(See Section X.Y for details)*.

#### Example Changelog Entry:
- **[2026-06-21]** - **Generic Repository Pattern**: Introduced Go Generics (1.18+) `BaseRepository[T]` to eliminate redundant CRUD boilerplate and enforce type-safe persistence operations. *(See Section 6.5)*.
- **[2026-07-02]** - **Strict Context Isolation**: Defined strict isolation rules for `context.Context` to prevent mutable domain state leakage across layers. *(See Section 7.5)*.
- **[2026-06-22]** - **AI Auto-Commit Protocol**: Formalized the workflow for AI agents to automatically commit and push any new additions to this living document, ensuring seamless synchronization with the remote repository. *(See AI Agent Git Automation Protocol below)*.
- **[2026-06-22]** - **Resilience Engineering**: Added mandatory Fail-Fast vs Silent-Failure classification rules, error propagation standards (wrap-with-context per layer, log-once rule), and distributed resilience patterns (Circuit Breaker, Retry+Backoff+Jitter, Timeout, Bulkhead, Idempotency, DLQ). *(See Section 7.22)*.
- **[2026-06-22]** - **Database Performance & SQL Tuning**: Added mandatory EXPLAIN plan analysis rules, Two-Step Query / CTE pattern to replace dangerous dependent subqueries, composite index strategy, and non-negotiable SQL best practices (no `SELECT *`, non-sargable predicates, UNION ALL, parameterized queries, transaction isolation awareness). *(See Section 7.23)*.
- **[2026-06-22]** - **Cognitive Complexity Reduction — Method Extraction Pattern**: Documented mandatory refactoring strategies for keeping function Cognitive Complexity ≤ 15 (SonarQube S3776): early-return guard clauses, private method extraction for enrichment logic, package-level const slices, and typed `applyIfNotNil`-style helpers to eliminate repetitive nil-pointer dereference blocks. *(See Section 7.24)*.

### AI Agent Git Automation Protocol
When an AI agent updates this document, the agent MUST immediately execute the following isolated workflow:
1. **Validate**: Ensure the changelog format is strictly followed.
2. **Isolate**: Stage ONLY this documentation file (`git add backend/clean_architecture_go.md`).
3. **Commit**: Use a semantic commit message, e.g., `docs(arch): update clean architecture base rules with [topic]`.
4. **Push**: Push to the current remote branch (`git push`).
This guarantees the living document is continuously synchronized with the remote repository without risking application code stability.

---

## 1. Overview
This document defines the design and standards for a Generic Go Clean Architecture Base Project. The boilerplate provides a reusable, modular, and scalable foundation for future backend services, so teams don't need to re-architect from scratch for every new project, improving development velocity and cross-project consistency.

## 2. Goals & Objectives
- **Standardization**: Enforce a consistent architectural pattern across services to reduce onboarding friction between teams.
- **Maintainability**: Decouple business logic from infrastructure concerns (HTTP framework, database drivers), keeping the codebase readable, testable, and easy to maintain.
- **Extensibility**: Allow new features or transport layers (e.g., gRPC, GraphQL) to be added, or underlying libraries swapped (e.g., MySQL → PostgreSQL), without breaking changes to the core business logic.

## 3. Technology Stack
- **Language**: Golang (minimum v1.20+)
- **HTTP Framework**: Can use a lightweight framework like Fiber or the Standard Library (net/http) wrapped with a flexible router.
- **Database**:
  - Relational (core, always present): MySQL / PostgreSQL
  - Caching (core, always present): Redis
  - Document store (optional, scaffolded on demand): MongoDB
  - Search/analytics (optional, scaffolded on demand): Elasticsearch
- **Dependency Management**: Go Modules (go.mod)
- **Configuration**: godotenv or viper for reading .env files.

## 4. Architecture Design (Clean Architecture)
The application is structured into 4 concentric layers, enforcing the **Dependency Rule**: source code dependencies must point inward only, from outer layers toward inner layers. Inner layers expose interfaces; outer layers provide concrete implementations (Dependency Inversion Principle).

1. **Domain Layer** (`internal/domain`) — The innermost layer, framework-agnostic and persistence-agnostic. Contains business entities and the interface contracts (ports) that outer layers must implement — including repository interfaces and usecase interfaces. Has zero outward dependencies; it must not import from any other layer or third-party framework.

2. **Repository Layer** (`internal/repositories`) — The data access layer (adapter). Implements the repository interfaces declared in the Domain layer, handling persistence concerns for external storage (RDBMS, NoSQL, third-party APIs, cache). Translates storage-specific errors/types into domain-level constructs.

3. **Usecase Layer** (`internal/usecases`) — The application/business logic layer. Orchestrates one or more repositories to fulfill a business operation, applying domain rules and invariants. Must remain transport-agnostic — no HTTP request/response types, status codes, or framework-specific constructs leak into this layer.

4. **Presentation / Handler Layer** (`internal/handlers`) — The outermost layer (delivery mechanism / adapter). Owns request parsing and response serialization: binds and validates incoming requests into DTOs (`internal/dto`), invokes the corresponding usecase, and maps the result into the standardized HTTP JSON response.

## 5. Folder Structure Standard
The repository follows an adapted Standard Go Project Layout:

```
base-project/
├── cmd/                    # (Optional) Entry point for separate microservices/workers
├── config/                 # Centralized configuration setup (Database, Redis, etc.)
├── internal/               # Core code, forbidden to be imported by external modules
│   ├── domain/             # Entity structs and Interfaces (Usecase, Repo, Client/Gateway Interface)
│   ├── dto/                # Data Transfer Object (Request & Response validation struct)
│   ├── handlers/           # Endpoint controllers (e.g., user_handler.go)
│   ├── repositories/       # Persistence adapters: DB/cache query implementations (e.g., user_repository.go)
│   ├── clients/            # Outbound adapters: third-party/external API consumers (e.g., payment_client.go)
│   ├── usecases/           # Business function implementations (e.g., user_usecase.go)
│   └── validation/         # Helpers dedicated to validating DTO input
├── middleware/             # HTTP interceptors (Auth Token Verification, Logger, Rate Limiter)
├── errs/                   # Custom application error definitions
├── httputil/               # Standard helpers for HTTP response building (Success/Error Builder)
├── main.go                 # Main program initialization (Dependency Injection)
├── .env.example             # Environment variable template
├── go.mod
└── go.sum
```

**`internal/clients/` — consuming external/third-party APIs**

`repositories/` and `clients/` are architecturally identical — both are **adapters** implementing an interface (port) declared in the Domain layer, and both must never be called directly by a usecase without going through that interface. They are split into separate folders purely for *semantic clarity*, since they represent different kinds of "outside the application" dependencies:

- `repositories/` — adapters for **this application's own data** (its DB, cache, document store, search index — see 7.18). The application owns the schema/data model.
- `clients/` — adapters for **someone else's service** (a payment gateway, an SMS/email provider, another internal microservice's API, a third-party identity provider). The application does not own the contract — it conforms to whatever the external API exposes.

This separation matters in practice because `clients/` typically needs concerns that `repositories/` doesn't: HTTP timeouts/retries with backoff, circuit breaking, request signing/auth headers, and response-schema mapping from the third party's DTO into the domain's own entity (never leak a third-party's response struct into the domain layer).

```go
// internal/domain/payment_gateway.go — port, defined in domain layer
type PaymentGateway interface {
    Charge(ctx context.Context, req ChargeRequest) (*ChargeResult, error)
}

// internal/clients/payment_client.go — adapter, implements the port
type stripeClient struct {
    httpClient *http.Client
    apiKey     string
}

func (c *stripeClient) Charge(ctx context.Context, req domain.ChargeRequest) (*domain.ChargeResult, error) {
    // build request, call the third-party API, map response -> domain.ChargeResult
}
```

The usecase layer depends only on `domain.PaymentGateway`, never on `stripeClient` directly — so swapping payment providers later means writing a new adapter in `clients/`, with zero changes to business logic.

## 6. Development Rules & Guidelines

### 6.1. Dependency Injection (DI)
Every handler or usecase must declare its dependencies as struct fields, injected via a constructor function (constructor injection). Example:

```go
func NewUserUsecase(repo domain.UserRepository) domain.UserUsecase {
    return &userUsecase{repo: repo}
}
```

Global state and package-level singletons are prohibited for accessing infrastructure resources (DB, cache, clients). Dependency wiring is composed hierarchically in `main.go` (the composition root).

### 6.2. DTO (Data Transfer Object)
Domain entities and persistence models must never be bound directly to a client's JSON request/response payload. Define dedicated structs in `internal/dto/` for request validation and response serialization, keeping the API contract decoupled from internal data structures.

### 6.3. Response Standardization
All API responses must conform to a single envelope schema, implemented in the `httputil/` package, with a distinct shape for success vs. error responses sharing the same top-level fields.

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

The current envelope is insufficient for errors on its own — `data` becomes ambiguous (null vs. empty object), there's no place for field-level validation detail, and there's no link back to the request-id for tracing (7.17). Extend it as follows:

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
- `data` is omitted (or explicitly `null`) on error responses — never mix partial success data with an error payload.
- `errors` is an array, present only for multi-field validation failures (e.g., HTTP 422 from DTO validation in 6.2/7.4). For single, non-field-specific errors (e.g., `ErrNotFound`, `ErrUnauthorized`), omit `errors` and rely on `code`/`message` alone.
- `code` is an **internal application error code** (not the HTTP status code), namespaced/structured enough to be looked up in documentation or matched by client-side error handling (e.g., `40001` = validation, `40401` = resource not found, `50001` = internal error). Define these as constants in `errs/` alongside the sentinel errors from 6.4.
- `request_id` is always included on error responses (sourced from context, per 7.17), so a user-reported error can be immediately cross-referenced against application logs and `audit_logs` without asking the user to reproduce the issue.
- The HTTP status code itself still follows standard semantics (400, 401, 403, 404, 409, 422, 500, ...) — the envelope's `code` field is a *supplement*, not a replacement, for the HTTP status.
- Never leak internal details in `message` for 5xx errors (no stack traces, no raw DB error strings) — log the raw error server-side (with `request_id` attached) and return a generic message to the client.

```go
type ErrorResponse struct {
    Code      string         `json:"code"`
    Message   string         `json:"message"`
    Errors    []FieldError   `json:"errors,omitempty"`
    RequestID string         `json:"request_id"`
    Meta      any            `json:"meta,omitempty"`
}

type FieldError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}
```

### 6.4. Error Handling
Errors must never be silently swallowed. Persistence-layer errors are wrapped at the repository (using `fmt.Errorf("...: %w", err)` or equivalent) and normalized into sentinel/custom domain errors at the usecase layer (e.g., `errs.ErrNotFound`). The handler layer performs the final translation from domain error to both an HTTP status code and the error envelope's `code`/`message`/`errors` fields defined in 6.3.

---

## 7. Suggested Additions

As a reusable base project, the following concerns are typically still missing and worth adding:

### 7.1. Testing Strategy
- Provide a `mocks/` directory (generated via `mockery` or `gomock`) for domain interfaces, so usecases can be unit-tested without a real database dependency.
- Convention: every `*_usecase.go` has a corresponding `*_usecase_test.go` using table-driven tests.
- Separate integration tests (build tag `//go:build integration`) for the repository layer that require a real DB/Redis instance (`testcontainers-go` is recommended).

### 7.2. Database Migration
- No migration strategy is currently defined. Add a tool such as `golang-migrate` or `goose`, plus a `migrations/` directory.
- Provide a separate seeder for dummy/test data.

### 7.3. Logging & Observability
- Adopt a structured logger (`zap` or `zerolog`) as the standard, injecting a request-id/correlation-id into every log entry via middleware.
- Expose health check endpoints (`/healthz`, `/readyz`) for liveness/readiness probes.
- Optional: integrate distributed tracing (OpenTelemetry) and metrics (Prometheus).

**Default Log Configuration**

- **Storage location**: logs are written to a `logs/` directory at the project root. The application must auto-create this directory on startup if it doesn't exist (`os.MkdirAll("logs", 0755)`).
- **File naming convention**: `{app-name}_{yyyy-mm-dd}.log` (e.g., `base-project_2026-06-21.log`), where `app-name` is read from config/env (`APP_NAME`).
- **Rotation policy**: daily rotation with a default 7-day retention, implemented via [`lumberjack`](https://github.com/natefinch/lumberjack) (`MaxAge: 7`) or an equivalent rolling-file writer. Rotation settings should be exposed via config so derivative projects can override the retention window.
- **Dual output (stdout + file)**: the logger must write to both stdout and the rotating log file simultaneously, using an `io.MultiWriter` (or the structured logger's equivalent multi-sink output), so terminal output and file output always stay identical — useful for local development (tail terminal) and container/orchestrator log collection (stdout is scraped by Docker/K8s) without losing the on-disk audit trail.

```go
logFile := &lumberjack.Logger{
    Filename: fmt.Sprintf("logs/%s_%s.log", appName, time.Now().Format("2006-01-02")),
    MaxAge:   7, // days
    Compress: true,
}
writer := io.MultiWriter(os.Stdout, logFile)
logger := zap.New(zapcore.NewCore(encoder, zapcore.AddSync(writer), level))
```

- **Git exclusion**: add `logs/` to `.gitignore` at the project root so log files are never committed. A `.gitkeep` (or equivalent placeholder) can be added inside `logs/` if the folder itself needs to exist in version control while its contents stay ignored.

### 7.4. Validation Layer
- The PRD already references `internal/validation/`, but the underlying library is unspecified. Recommend `go-playground/validator` with a custom error message formatter, kept consistent with the `httputil` response envelope.

### 7.5. Context Propagation
- Enforce the convention: `context.Context` must be the first parameter in every usecase, repository, and handler-invoked function signature, to support timeout, cancellation, and trace propagation.
- `context.Context` is the single transport mechanism for cross-cutting, per-request values (request-id, actor/user-id, trace span) — never use global variables or package-level state to pass request-scoped data between layers.
- Use unexported, typed context keys (e.g., `type ctxKey string; const requestIDKey ctxKey = "request_id"`) instead of raw strings, to avoid key collisions across packages.
- Provide a small `ctxutil` (or extend `httputil`) package with typed getter/setter helpers — e.g., `ctxutil.WithRequestID(ctx, id)` / `ctxutil.RequestID(ctx)` — so every layer reads/writes context values through the same accessor instead of duplicating `context.Value` calls.
- Set sensible default timeouts at the handler/CLI entry point (e.g., `context.WithTimeout(ctx, 30*time.Second)`) so a hung downstream call (DB, external API) doesn't block indefinitely; usecases and repositories must respect `ctx.Done()`/`ctx.Err()` rather than ignoring it.

### 7.6. Transaction Management
- There is currently no pattern for cross-repository database transactions (e.g., a usecase that must insert into two tables atomically). Introduce a `UnitOfWork` pattern or transaction manager interface in the domain layer, so the usecase can control commit/rollback without depending on driver-specific transaction details.

### 7.7. Authentication & Authorization Hook
- Since this is a base project, the auth middleware should be generic/pluggable (e.g., a `TokenValidator` interface) rather than hardcoded to a single mechanism, allowing derivative projects to plug in opaque tokens, JWTs, or API keys as needed.

### 7.8. Graceful Shutdown
- `main.go` must handle `SIGTERM`/`SIGINT` and gracefully close DB/Redis connections and the HTTP server (using `context.WithTimeout` during shutdown) to avoid dropping in-flight requests.

### 7.9. Linting & CI
- Add a `golangci-lint` configuration (`.golangci.yml`) with baseline rules (errcheck, govet, staticcheck, cyclop for cognitive complexity).
- Provide an example CI pipeline (GitHub Actions): lint → test → build.

### 7.10. API Documentation
- Generate Swagger/OpenAPI specs via `swaggo/swag` annotations on handlers, so every derivative project automatically ships with up-to-date API documentation.

### 7.11. Rate Limiting & Security Headers
- Add middleware for rate limiting (token bucket algorithm, backed by Redis), CORS configuration, and security headers (helmet-equivalent), enabled by default and configurable via environment variables.

### 7.12. Versioning Strategy
- Define an API versioning strategy upfront (URL path `/v1/`, header-based, or query parameter) to keep it consistent across all derivative projects.

### 7.13. CLI Tooling with Cobra
A base project benefits from a unified CLI entry point instead of a single `main.go` that only boots the HTTP server. Introduce [`spf13/cobra`](https://github.com/spf13/cobra) to expose operational commands as subcommands of a single binary:

```
cmd/
├── root.go            # Root command, global flags (--env, --config)
├── serve.go            # `app serve` — starts the HTTP server
├── migrate.go          # `app migrate up|down|status` — wraps golang-migrate/goose
├── seed.go             # `app seed` — runs database seeders
└── worker.go            # `app worker` — starts background job/consumer process
```

Example root command wiring:

```go
var rootCmd = &cobra.Command{
    Use:   "app",
    Short: "Base project CLI",
}

func Execute() error {
    rootCmd.AddCommand(serveCmd, migrateCmd, seedCmd, workerCmd)
    return rootCmd.Execute()
}
```

- Each subcommand reuses the same dependency-injection composition root (config loading, DB/Redis clients) as the HTTP server, just with a different entry point — avoiding duplicated bootstrap logic.
- `cobra.Command.PersistentFlags()` is used for global flags such as `--env` or `--config-path`, parsed once at the root level.
- Pairs naturally with `viper` for config binding (flags, environment variables, and `.env`/`.yaml` files resolved through a single source of truth).

### 7.14. Containerization with Docker
The base project should ship container-ready by default so derivative projects don't need to author Docker configuration from scratch.

- **Multi-stage `Dockerfile`**: a `builder` stage compiles a static binary (`CGO_ENABLED=0 GOOS=linux go build`), and a minimal `distroless` or `alpine` runtime stage copies only the compiled binary — keeping the final image small and reducing attack surface.

```dockerfile
# ---- builder stage ----
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/bin/app ./main.go

# ---- runtime stage ----
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/bin/app /app/bin/app
ENTRYPOINT ["/app/bin/app", "serve"]
```

- **`docker-compose.yml`** for local development: defines the app service alongside its dependencies (MySQL/PostgreSQL, Redis), with health checks and volume mounts so the stack runs with a single `docker compose up`.
- **`.dockerignore`**: exclude `.git`, `medocs/`, test files, and local `.env` to keep build context minimal.
- Container image should respect `12-factor` principles: configuration via environment variables only, no secrets baked into the image, logs written to stdout/stderr (not files).
- The `app migrate` and `app seed` Cobra subcommands (see 7.13) can be invoked as one-off containers in CI/CD or as Kubernetes Jobs/init containers, reusing the same image as the running service.

### 7.15. Query Debug Logging
For local development and troubleshooting, the base project should support toggling raw SQL query logging via configuration, without requiring code changes.

- **Config flag**: `DB_DEBUG=true` (or `LOG_QUERY=true`) read from `.env`/environment variables at startup.
- When enabled, every executed query — including bound parameter values and execution duration — is printed to the configured logger (respecting the same stdout + file dual-output sink described in 7.3), making it easy to spot N+1 queries or slow statements during development.
- When disabled (default in production), query logging is fully skipped to avoid leaking sensitive data into logs and to avoid the performance overhead of formatting every query.

Implementation (without GORM, using `database/sql` / `sqlx` directly):
- Wrap the `*sql.DB` connection or driver with a logging decorator around `QueryContext`/`ExecContext`/`QueryRowContext`, so every call is intercepted at one place rather than scattered across repositories.
- The decorator logs the query string, bound args, and elapsed time only when `cfg.DB.Debug == true`; when `false`, it simply delegates to the underlying driver with no overhead.

```go
if cfg.DB.Debug {
    start := time.Now()
    logger.Debug("executing query",
        zap.String("query", query),
        zap.Any("args", args),
        zap.Duration("duration", time.Since(start)),
    )
}
```

- **Security note**: query debug logging must always default to `false` and should never be enabled in production by default, since bound parameters may include sensitive values (passwords, tokens, PII). Treat it the same as a secret-adjacent config flag.

### 7.16. Audit Trail (Optional)
Audit trail is **not a requirement for every derivative project** — it adds write overhead, storage cost, and retention/compliance burden. Only enable it for domains where "who did what, when" carries compliance, security, or accountability value (e.g., access management, financial transactions, admin actions on sensitive data). For simple internal tools or read-heavy services, skip it entirely.

The recommendation below is for **when it is needed**, designed as an opt-in module rather than a core dependency, so it doesn't burden derivative projects that don't need it.

An audit trail tracks *who did what, to which resource, and when* — distinct from application logs, which track *system behavior*. For a base project, the recommended approach:

**Design options**

1. **Dedicated `audit_logs` table** (most common, relational): a single append-only table capturing each mutating action.
   ```
   audit_logs
   ├── id
   ├── actor_id          # user/service performing the action
   ├── action            # e.g., "user.create", "role.assign"
   ├── resource_type      # e.g., "user", "role"
   ├── resource_id
   ├── before_data        # JSON snapshot before change (nullable for CREATE)
   ├── after_data         # JSON snapshot after change (nullable for DELETE)
   ├── ip_address
   ├── user_agent
   └── created_at
   ```
2. **Event-sourcing / append-only event log** (heavier, but gives a full replayable history) — generally overkill for a generic base project unless a derivative project specifically needs it.

**Where to capture it**
- Capture at the **usecase layer**, not the handler or repository — the usecase is where business intent is known (e.g., "user X approved access request Y"), while the repository only knows raw SQL/CRUD. Capturing here keeps audit semantics meaningful regardless of transport (HTTP, CLI, gRPC, worker).
- Use a dedicated `AuditLogger` interface in the domain layer (port), with a repository implementation (adapter) — same pattern as other repositories, so it's swappable (DB table vs. external audit service like a SIEM). For projects that don't need it, inject a `NoopAuditLogger` (no-op implementation) — the usecase code stays identical whether auditing is active or not, keeping it truly optional at the composition-root level.

```go
type AuditLogger interface {
    Record(ctx context.Context, entry AuditEntry) error
}

type AuditEntry struct {
    ActorID      string
    Action       string
    ResourceType string
    ResourceID   string
    Before       any
    After        any
}
```

- For cross-cutting consistency, audit writes should participate in the **same transaction** as the business mutation (see 7.6 UnitOfWork) — if the business write rolls back, the audit entry should roll back too, to avoid orphaned/misleading audit records.

**Operational considerations**
- **Immutability**: the audit table should be insert-only at the application level — no `UPDATE`/`DELETE` usecases should target it. Consider DB-level protections (e.g., a `REVOKE UPDATE, DELETE` grant on that table for the app's DB role) for stronger guarantees.
- **Retention & volume**: audit data tends to grow quickly; plan for partitioning (e.g., by month) or periodic archival to cold storage, separate from the 7-day log rotation policy in 7.3 — audit trail typically needs to be retained far longer (compliance-driven, often 1+ years) than operational logs.
- **What NOT to store**: never write raw secrets/passwords/tokens into `before_data`/`after_data` — mask or omit sensitive fields before persisting the snapshot.
- **Read access**: expose audit trail via a read-only endpoint/usecase (e.g., `GET /audit-logs?resource_id=...`), separate from the write path, typically restricted to admin/compliance roles.

`AuditLogger` pairs naturally with the pluggable `TokenValidator` from 7.7 — both can resolve `actor_id` from the same authenticated context.
The base project should ship container-ready by default so derivative projects don't need to author Docker configuration from scratch.

- **Multi-stage `Dockerfile`**: a `builder` stage compiles a static binary (`CGO_ENABLED=0 GOOS=linux go build`), and a minimal `distroless` or `alpine` runtime stage copies only the compiled binary — keeping the final image small and reducing attack surface.

```dockerfile
# ---- builder stage ----
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/bin/app ./main.go

# ---- runtime stage ----
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/bin/app /app/bin/app
ENTRYPOINT ["/app/bin/app", "serve"]
```

- **`docker-compose.yml`** for local development: defines the app service alongside its dependencies (MySQL/PostgreSQL, Redis), with health checks and volume mounts so the stack runs with a single `docker compose up`.
- **`.dockerignore`**: exclude `.git`, `medocs/`, test files, and local `.env` to keep build context minimal.
- Container image should respect `12-factor` principles: configuration via environment variables only, no secrets baked into the image, logs written to stdout/stderr (not files).
- The `app migrate` and `app seed` Cobra subcommands (see 7.13) can be invoked as one-off containers in CI/CD or as Kubernetes Jobs/init containers, reusing the same image as the running service.

### 7.17. Request ID / Correlation ID for Tracing
To trace a single request end-to-end — across application logs, query debug logs (7.15), and audit trail entries (7.16) — every request must carry a unique identifier from the moment it enters the system until it completes.

**Generation & propagation**
- Generate a request-id at the outermost entry point: HTTP middleware for the handler layer, or at the root command for CLI/worker entry points (7.13). Use a UUID (`google/uuid`, v4 or v7) or a sortable ID (`ulid`) — v7/ULID is preferable since it's time-sortable, making log inspection easier.
- If the request arrives with an existing `X-Request-ID` header (e.g., propagated from an upstream gateway/load balancer), reuse it instead of generating a new one — otherwise generate fresh.
- Store the request-id in `context.Context` via the `ctxutil` helper from 7.5, immediately after generation, so it's available to every downstream usecase/repository call without needing to be passed explicitly as a function argument.
- Echo the request-id back in the response header (`X-Request-ID`) so clients can reference it when reporting issues.

```go
func RequestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        reqID := r.Header.Get("X-Request-ID")
        if reqID == "" {
            reqID = uuid.NewString()
        }
        ctx := ctxutil.WithRequestID(r.Context(), reqID)
        w.Header().Set("X-Request-ID", reqID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

**Wiring it into logs and audit trail**
- The structured logger (7.3) must be configured to automatically attach the request-id from context to every log line within that request's lifecycle — typically via a per-request child logger (`logger.With(zap.String("request_id", ctxutil.RequestID(ctx)))`) created once in the middleware and passed/stored in context alongside the id.
- Query debug logs (7.15) inherit the same per-request logger, so a slow or suspicious query can be traced back to the exact request that triggered it.
- The `AuditLogger` (7.16) must read `request_id` from context and store it as a column on `audit_logs`, alongside `actor_id`. This closes the loop: given a `request_id` from a user complaint or incident, an engineer can grep application logs for that id *and* query `audit_logs WHERE request_id = ?` to get the complete picture — what happened technically, and what business action it represented.
- For async/background work (workers, queue consumers), generate a new request-id per job but optionally carry the originating HTTP request-id as a separate `parent_request_id` field, preserving traceability across service/process boundaries.

**Note on distributed tracing**: if OpenTelemetry tracing (7.3) is adopted later, the request-id can be aligned with (or replaced by) the OTel `trace_id`, since both serve the same correlation purpose — avoid maintaining two parallel, unrelated identifiers once distributed tracing is in place.


### 7.18. Redis Caching, MongoDB & Elasticsearch (Core vs. On-Demand)
Not every derivative project needs a document store or a search engine, but almost all benefit from caching. The base project should treat these three differently:

- **Redis (core, always wired)** — included in `config/` and `docker-compose.yml` by default, since caching is broadly applicable (session storage, rate limiting in 7.11, cache-aside for hot reads).
- **MongoDB & Elasticsearch (optional, generated on demand)** — *not* wired into the base project by default. Instead, provide them as scaffold-able modules (via a project generator script or `cobra` CLI command, e.g., `app generate datastore mongo`) that a derivative project pulls in only when it actually needs a document store or full-text search — keeping the base project's footprint and dependency surface minimal for projects that don't.

**7.18.1 Redis — Caching**

- **Connection**: initialize a single `*redis.Client` (using `go-redis/redis/v9` or later) in the composition root (`main.go`/`cmd/serve.go`), configured via `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`, `REDIS_DB`, with connection pooling defaults (`PoolSize`, `MinIdleConns`) exposed via config.
- **Port/adapter pattern**: define a `domain.Cache` interface (`Get`, `Set`, `Delete`, `Exists`, with `context.Context` as the first parameter per 7.5) in the domain layer; implement it in `internal/repositories/cache/redis_cache.go`. Usecases depend on the interface, never on `*redis.Client` directly — keeping Redis swappable (e.g., for an in-memory cache in tests).
- **Cache-aside pattern**: usecases check cache first, fall back to the repository/DB on a miss, then populate the cache — implemented at the usecase layer, not the repository layer, since cache invalidation is a business decision (e.g., invalidate `user:{id}` cache after a `user.update` usecase).
- **Key naming convention**: `{domain}:{entity}:{id}` (e.g., `app:user:42`), with a sensible default TTL per key type defined in config rather than hardcoded inline.
- **Health check**: include Redis in the `/readyz` probe from 7.3 — `PING` on startup and periodically.

```go
type Cache interface {
    Get(ctx context.Context, key string) (string, error)
    Set(ctx context.Context, key string, value any, ttl time.Duration) error
    Delete(ctx context.Context, key string) error
}
```

**7.18.2 MongoDB — Generated On Demand**

When a derivative project needs a document store, the generator scaffolds:
- An `internal/repositories/mongo/` package implementing the relevant domain repository interface against `mongo-driver/v2`, following the exact same port/adapter contract used by the SQL repositories — usecases remain unaware of which storage backend fulfills the interface.
- Connection setup (`MONGO_URI`, `MONGO_DB_NAME`) added to `config/` and `docker-compose.yml` only for that project, not the shared base.
- A minimal index-management convention (e.g., an `EnsureIndexes(ctx) error` method called once at startup) since MongoDB doesn't have migrations in the relational sense.
- Note: MongoDB is additive, not a replacement for the relational DB — typical use cases are semi-structured/high-write-throughput data (audit/event logs as an alternative to 7.16's relational table, activity feeds) rather than the core transactional domain.

**7.18.3 Elasticsearch — Generated On Demand**

When full-text search or analytics is needed, the generator scaffolds:
- An `internal/repositories/search/` package implementing a `domain.SearchRepository` port (e.g., `Search(ctx, query) ([]Result, error)`, `Index(ctx, doc) error`) against the official `elasticsearch-go` client.
- A write-side sync strategy must be chosen explicitly per project — either **dual-write** from the usecase layer (simplest, but risks drift) or an **outbox/event-driven sync** (e.g., publish a domain event after a successful DB write, consumed by a worker that indexes into Elasticsearch) — the latter is recommended once correctness matters, since it decouples the source-of-truth write from the search-index write.
- Index mapping/settings should be version-controlled (e.g., as JSON files under `internal/repositories/search/mappings/`) and applied via a CLI subcommand (`app search reindex`), consistent with the Cobra pattern in 7.13.

**Why generate instead of always-include**: keeping MongoDB and Elasticsearch out of the default dependency tree avoids forcing every derivative project to run containers it doesn't need (lighter `docker-compose.yml`, faster `go mod` graph, smaller default attack surface) — see 7.14's container-per-project-need principle. The base project's value is the *consistent pattern* (ports/adapters, config conventions, CLI scaffolding) for adding them, not bundling every backend by default.

### 7.19. Application Versioning & Health Check
Every derivative project needs a reliable way to answer "what version is actually running right now" — especially across multiple environments and container deployments — and to verify the service is alive and ready to serve traffic.

**7.19.1 Application Versioning**

- **Build-time version injection**: avoid hardcoding a version string in source. Inject it at compile time via `-ldflags`, so the same source tree produces a uniquely identifiable binary per build:

```bash
go build -ldflags "\
  -X main.version=$(git describe --tags --always) \
  -X main.commit=$(git rev-parse --short HEAD) \
  -X main.buildTime=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  -o bin/app ./main.go
```

```go
var (
    version   = "dev"     // overridden via -ldflags at build time
    commit    = "none"
    buildTime = "unknown"
)
```

- **Semantic versioning**: tag releases following `MAJOR.MINOR.PATCH` (e.g., `v1.4.2`); `git describe --tags` naturally falls back to a commit-based pseudo-version when no tag is present, which is useful for `dev`/`staging` builds.
- **Dockerfile wiring**: pass the same values as build args into the multi-stage `Dockerfile` (7.14), so the container image's embedded version matches the binary that was actually built, not just the image tag:

```dockerfile
ARG VERSION=dev
ARG COMMIT=none
ARG BUILD_TIME=unknown
RUN go build -ldflags "-X main.version=${VERSION} -X main.commit=${COMMIT} -X main.buildTime=${BUILD_TIME}" -o /app/bin/app ./main.go
```

- **CLI exposure**: add an `app version` Cobra subcommand (7.13) that prints version/commit/build-time — useful for quick verification in any environment, including inside a running container (`docker exec <container> app version`).
- **HTTP exposure**: expose a lightweight `GET /version` endpoint returning the same metadata as JSON, following the standard response envelope (6.3):

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

Distinguish **liveness** from **readiness** — they answer different questions and should not be conflated into a single endpoint:

- **`GET /healthz` (liveness)** — answers "is the process alive and not deadlocked?". Should be extremely cheap (no downstream calls), just confirming the HTTP server itself can respond. Used by the orchestrator (Docker/Kubernetes) to decide whether to restart the container.
- **`GET /readyz` (readiness)** — answers "can this instance actually serve traffic right now?". Actively checks downstream dependencies: DB (`db.PingContext(ctx)`), Redis (`PING`, per 7.18.1), and any optional data stores in use (MongoDB/Elasticsearch, per 7.18.2/7.18.3) if the project has them wired. Used by the orchestrator/load balancer to decide whether to route traffic to this instance.

```go
func ReadyzHandler(db *sql.DB, cache domain.Cache) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
        defer cancel()

        checks := map[string]string{}
        status := http.StatusOK

        if err := db.PingContext(ctx); err != nil {
            checks["database"] = "down"
            status = http.StatusServiceUnavailable
        } else {
            checks["database"] = "up"
        }

        if err := cache.Ping(ctx); err != nil {
            checks["redis"] = "down"
            status = http.StatusServiceUnavailable
        } else {
            checks["redis"] = "up"
        }

        w.WriteHeader(status)
        json.NewEncoder(w).Encode(map[string]any{"checks": checks})
    }
}
```

- **Response format for `/healthz` and `/readyz`** intentionally bypasses the standard envelope (6.3) — orchestrators expect a plain HTTP status code (200/503) and a minimal body, not a business-response wrapper; keep them lightweight and machine-oriented rather than client-API-oriented.
- **Embed version in health responses**: include `version`/`commit` in the `/readyz` payload as well, so an incident responder can immediately correlate "which version is unhealthy" without a separate call to `/version` — directly useful alongside `request_id`-based tracing (7.17) when triaging a rollout-related incident.
- **Timeouts matter**: every downstream check in `/readyz` must run under a short `context.WithTimeout` (per 7.5) — a hung dependency must not hang the readiness probe itself, or the orchestrator's probe will time out and potentially cause a cascading restart loop.
- **Don't leak internals**: `/healthz` and `/readyz` are typically unauthenticated (probed by infrastructure, not end users) — keep their payload limited to up/down status per dependency, never raw connection strings, internal hostnames, or error stack traces.

### 7.20. Singleflight — Deduplicating Concurrent Identical Requests
When many concurrent requests ask for the exact same data at the same moment (a popular product page, a dashboard widget, a cache key that just expired), each one independently hitting the DB/downstream service is wasteful and can cause a **thundering herd / cache stampede** — especially right after a cache miss or TTL expiry (7.18.1). The `singleflight` pattern collapses concurrent identical calls into a single in-flight execution, sharing the result across all callers.

**Library**: [`golang.org/x/sync/singleflight`](https://pkg.go.dev/golang.org/x/sync/singleflight) — part of the official extended Go standard library, no third-party dependency risk.

**Where it belongs**: at the **usecase layer**, wrapping the call to the repository/cache, not inside the repository itself — singleflight is a concurrency-control concern tied to a specific business operation's key, which the usecase understands; the repository should stay a thin, unaware data-access adapter.

```go
type ProductUsecase struct {
    repo  domain.ProductRepository
    cache domain.Cache
    sf    singleflight.Group
}

func (u *ProductUsecase) GetProduct(ctx context.Context, id string) (*domain.Product, error) {
    key := fmt.Sprintf("product:%s", id)

    if cached, err := u.cache.Get(ctx, key); err == nil {
        return decode(cached), nil
    }

    // Concurrent callers for the same key share one execution and one result.
    v, err, shared := u.sf.Do(key, func() (any, error) {
        product, err := u.repo.FindByID(ctx, id)
        if err != nil {
            return nil, err
        }
        _ = u.cache.Set(ctx, key, encode(product), 5*time.Minute)
        return product, nil
    })
    if err != nil {
        return nil, err
    }
    if shared {
        logger.Debug("singleflight: result shared across concurrent callers", zap.String("key", key))
    }
    return v.(*domain.Product), nil
}
```

**Key conventions**

- **Key design**: the singleflight key must encode every parameter that affects the result (e.g., `product:{id}:{locale}`, not just `product:{id}`) — an under-specified key silently leaks one caller's result to a different request.
- **One `singleflight.Group` per usecase/resource type**, instantiated once in the composition root and injected (not a global package-level `var`), consistent with the no-global-state rule in 6.1.
- **`context.Context` caution**: `singleflight.Do` executes the function once for the *first* caller's context; if that caller cancels, the shared in-flight call can be cancelled for *all* waiting callers too. For request-scoped work where this matters, prefer `singleflight.DoChan` and apply each caller's own timeout independently, or detach a short-lived background context for the underlying fetch.
- **Error sharing is intentional**: if the underlying call fails, all concurrent callers receive the same error — don't retry inside the singleflight function itself; let each caller's normal retry/error-handling logic decide what to do next.
- **Pairs with, doesn't replace, caching**: singleflight only deduplicates *concurrent* calls during the brief window a value is being fetched; it has no memory once the call completes. It solves the stampede at the *miss* moment — the cache (7.18.1) still does the steady-state heavy lifting between misses.

**Typical use cases in this base project**: cache-miss repopulation (as above), expensive aggregation/report endpoints, external API calls with rate limits, and any `GET` usecase that's read-heavy and idempotent. Avoid applying it to non-idempotent operations (writes, mutations) — singleflight is for collapsing redundant *reads*, never for deduplicating commands that have side effects.

### 7.21. Message Broker — Kafka & RabbitMQ (Generated On Demand)
Same principle as MongoDB and Elasticsearch in 7.18: not every derivative project needs asynchronous messaging, so Kafka and RabbitMQ are **not wired into the base project by default**. They are scaffolded only when a project explicitly needs event streaming or task/queue-based processing — generated via the same `cobra` scaffolding command pattern (e.g., `app generate broker kafka` / `app generate broker rabbitmq`), keeping the base project's default dependency tree and `docker-compose.yml` minimal for projects that don't need them.

**Folder placement**: messaging adapters live in `internal/clients/messaging/` (extending the `clients/` convention from Section 5, since a broker is an external service the application doesn't own) for producers, plus a dedicated `internal/consumers/` package for the consumption side, since consumers have a different lifecycle (long-running background process) than request-driven usecases.

```
internal/
├── clients/
│   └── messaging/
│       ├── kafka_producer.go      # implements domain.EventPublisher
│       └── rabbitmq_producer.go   # implements domain.EventPublisher
└── consumers/
    ├── kafka_consumer.go          # long-running consumer loop, invokes a usecase per message
    └── rabbitmq_consumer.go
```

**Port/adapter pattern (producer side)**

Define the publishing contract in the domain layer, same as every other external dependency — usecases never import `segmentio/kafka-go` or `amqp091-go` directly:

```go
// internal/domain/event_publisher.go — port
type EventPublisher interface {
    Publish(ctx context.Context, topic string, event Event) error
}

type Event struct {
    ID        string
    Type      string
    Payload   any
    Timestamp time.Time
}
```

**7.21.1 Kafka — when scaffolded**

- Client: `segmentio/kafka-go` (pure Go, no cgo dependency — preferred for a portable base project) or `confluent-kafka-go` if librdkafka-backed performance is required.
- Config added only to the generated project: `KAFKA_BROKERS`, `KAFKA_CONSUMER_GROUP`, `KAFKA_TOPIC_PREFIX`.
- Consumer wiring: a dedicated `app worker kafka` Cobra subcommand (7.13) runs the consumer loop as a separate process/container from the HTTP server — never started inline inside `app serve`, since consumer and API have independent scaling and restart needs.
- Each consumed message must carry/propagate a `request_id` (or generate a new one if absent) into context, so the message-processing usecase still benefits from the tracing chain in 7.17, and any resulting audit entries (7.16) remain correlatable back to the originating event.
- Use Kafka when the use case is high-throughput event streaming, multiple independent consumers per topic, or replayability (offset-based) matters.

**7.21.2 RabbitMQ — when scaffolded**

- Client: `rabbitmq/amqp091-go` (the maintained successor to `streadway/amqp`).
- Config added only to the generated project: `RABBITMQ_URL`, `RABBITMQ_EXCHANGE`, `RABBITMQ_QUEUE`.
- Consumer wiring: same pattern as Kafka — a separate `app worker rabbitmq` subcommand, independent process from the HTTP server.
- Use RabbitMQ when the use case is task/job queueing, work distribution with acknowledgment semantics, or routing-based dispatch (exchanges/routing keys) fits better than a log-based stream.

**Shared conventions regardless of broker chosen**

- **Idempotency**: consumers must treat redelivery as a normal case (at-least-once delivery is the realistic default for both Kafka and RabbitMQ) — usecases triggered by a consumed message should be designed to be safely re-runnable (e.g., upsert instead of insert, or a dedupe check keyed on `event.ID`).
- **Dead-letter handling**: define a dead-letter topic/queue convention from the start (`{topic}.dlq` for Kafka, a DLX/dead-letter exchange for RabbitMQ) so poison messages don't block the consumer loop indefinitely.
- **Graceful shutdown**: consumer processes must honor the same `SIGTERM`/`SIGINT` handling as the HTTP server (7.8) — stop pulling new messages, finish in-flight processing, then exit, to avoid message loss or duplicate reprocessing on deploy.
- **Health check**: when a broker is scaffolded into a project, extend that project's `/readyz` (7.19.2) to include a connectivity check for the broker, consistent with how DB/Redis/Mongo/ES checks are wired.

**Why generate instead of always-include**: identical reasoning to 7.18 — bundling a message broker by default would force every derivative project (including simple CRUD services with no async needs) to run an extra container and carry extra dependencies. The base project provides the *pattern* (port/adapter, idempotent consumer design, dead-letter convention, separate worker process) so that adding Kafka or RabbitMQ to a project that needs it is fast and consistent, without penalizing the projects that don't.

### 7.22. Resilience Engineering — Fail-Fast, Error Propagation & Distributed Patterns

Enterprise systems must be designed for failure from day one. Resilience is a **core architectural concern**, not an optional add-on. Every engineer and AI agent working on this codebase must apply these rules without exception.

**7.22.1 Fail-Fast vs Silent-Failure**

The most important classification for any error handling decision: **is this fail-fast or silent?** Never leave this implicit.

**Fail-Fast** — The application crashes or throws loudly the moment something goes wrong. This is the **correct** behavior for unrecoverable states. A service that dies loudly is easier to debug than one that silently returns wrong data.

Mandatory fail-fast scenarios:
- Database connection is down at startup → **do NOT start the service**
- Required environment variable is missing → **do NOT proceed**
- Config file is malformed → **do NOT silently use defaults**

**Silent-Failure (ANTI-PATTERN — must be actively prevented)**

The application swallows the error and returns a fake/empty response to the caller, making the system appear healthy when it is not. Silent wrong results are harder to detect than crashes — they corrupt data, mislead users, and surface as business problems, not technical alerts.

Examples of silent-failure anti-patterns to prevent:
- Function catches DB exception, returns empty list → caller thinks "no data exists" when actually the DB is down
- Service times out calling upstream, returns cached stale response **without flagging it as stale**
- Validation silently strips invalid fields instead of rejecting the request

> **Rule**: Always ask: *"If this operation fails, will anyone know immediately?"* If the answer is NO — that is a silent failure. Fix it.

**7.22.2 Error Propagation — Wrap, Context, Surface**

Every error that crosses a layer boundary must carry context about **WHERE** it failed and **WHY** — not just what the error code was.

```go
// Repository layer — wrap with operation context
func (r *userRepository) GetByID(ctx context.Context, id int64) (*domain.User, error) {
    user, err := r.db.QueryRowContext(ctx, query, id)
    if err != nil {
        return nil, fmt.Errorf("repository.GetUserByID id=%d: %w", id, err)
    }
    return user, nil
}
```

Layer responsibilities for error handling:
- **Repository layer** → wraps DB/driver errors with operation context, maps to domain errors (e.g., `errs.ErrNotFound`)
- **Usecase layer** → wraps domain/business logic errors with usecase context
- **Handler layer** → maps domain errors to HTTP status + error envelope (see 6.3/6.4)
- **Middleware** → catches any unhandled errors as 500, logs them with `request_id` (see 7.17), and alerts

> **Critical Rule**: Never log AND re-throw the same error. **Log once at the top boundary (middleware/handler), propagate with context below.** Duplicate logs create noise that causes real alerts to be missed.

**7.22.3 Resilience Patterns for Distributed Systems**

Apply these patterns at the `clients/` adapter layer (see Section 5) for all outbound external/third-party calls:

- **Circuit Breaker**: stop calling a failing downstream service and fail-fast immediately until it recovers. Prevents cascading failures across services.
- **Retry with Exponential Backoff + Jitter**: apply only for **transient failures** (network blips, 503s). Never retry business logic errors (4xx responses must NOT be retried — they will always fail again).
- **Timeout**: every external call — DB query, HTTP client, cache — must have an explicit `context.WithTimeout`. Never wait indefinitely. (Aligns with 7.5 Context Propagation.)
- **Bulkhead**: isolate resource pools (e.g., separate connection pools per downstream) so one failing dependency cannot exhaust goroutines or connections for the entire service.
- **Fallback / Degraded Mode**: define an explicit degraded-mode response when the primary path fails — but **always flag it explicitly**. Never return stale data as if it's fresh. Return a partial response with a clear signal (e.g., `"data_freshness": "stale"` in the response meta).
- **Idempotency**: design mutation endpoints to be safely retried. Use idempotency keys (stored in DB/Redis) for payments, order creation, and any critical write that must not be double-applied.
- **Dead-Letter Queue (DLQ)**: messages that repeatedly fail processing must be captured in a DLQ, never silently dropped. Aligns with 7.21's broker conventions.

```go
// Example: timeout + retry wrapper for an external client call
func (c *stripeClient) Charge(ctx context.Context, req domain.ChargeRequest) (*domain.ChargeResult, error) {
    callCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    var result *domain.ChargeResult
    err := retry.Do(func() error {
        var callErr error
        result, callErr = c.doCharge(callCtx, req)
        return callErr
    },
        retry.Attempts(3),
        retry.DelayType(retry.BackOffDelay),
        retry.OnRetry(func(n uint, err error) {
            logger.Warn("stripe charge retry", zap.Uint("attempt", n), zap.Error(err))
        }),
    )
    if err != nil {
        return nil, fmt.Errorf("stripeClient.Charge: %w", err)
    }
    return result, nil
}
```

---

### 7.23. Database Performance & SQL Tuning

Understanding **how** the database executes a query is more important than knowing how to write one. These rules apply to every query written in any repository in this codebase.

**7.23.1 Execution Plan Analysis — Mandatory Before Shipping**

Every non-trivial query **must be analyzed with `EXPLAIN` before deploying to any shared environment.**

Key fields to read in MySQL `EXPLAIN` output:

| Field | What to Look For |
|---|---|
| `type` | Access method ranked best→worst: `system > const > eq_ref > ref > range > index > ALL`. If you see `ALL` on a large table: **STOP** — that is a Full Table Scan. |
| `key` | Which index was actually chosen (`NULL` = no index used). |
| `rows` | Estimated rows examined — **not** rows returned. High `rows` = slow. |
| `Extra: "Using filesort"` | Sort happening in memory/disk, not via index — expensive. |
| `Extra: "Using temporary"` | Temp table created — expensive for `GROUP BY`/`DISTINCT`. |
| `Extra: "Using index"` | Covering index hit — very efficient, base table not touched. |
| `Extra: "Using where"` | Filter applied **after** row fetch — may indicate poor index coverage. |

Run `EXPLAIN ANALYZE` (MySQL 8.0+) for actual execution stats, not estimates. A large discrepancy between `rows_estimated` vs `rows_actual` signals stale statistics — fix with `ANALYZE TABLE tablename;`.

**7.23.2 Two-Step Query Pattern vs Dependent Subquery (Anti-Pattern)**

For reporting queries on large tables, **always prefer the Two-Step / CTE approach** over dependent (correlated) subqueries.

**Why dependent subqueries are dangerous**: a correlated subquery re-executes for **every row** in the outer query. On a table with 1,000,000 rows, the subquery runs 1,000,000 times. `EXPLAIN` will show `select_type: DEPENDENT SUBQUERY`.

```sql
-- ❌ ANTI-PATTERN: dependent subquery — re-runs per row
SELECT * FROM transactions t
WHERE t.amount = (
    SELECT MAX(amount)
    FROM transactions
    WHERE branch_id = t.branch_id  -- ← correlated: re-executes per outer row
)
```

```sql
-- ✅ PREFERRED: two-step with CTE — the subquery runs exactly once
WITH branch_max AS (
    SELECT branch_id, MAX(amount) AS max_amount
    FROM transactions
    GROUP BY branch_id
)
SELECT t.*
FROM transactions t
JOIN branch_max r
    ON t.branch_id = r.branch_id
    AND t.amount = r.max_amount
```

> **Rule of thumb**: if a subquery references a column from the outer query (correlated subquery), always investigate whether it can be rewritten as a `JOIN` or CTE. It almost always can — and **should be**.

Use window functions (`ROW_NUMBER`, `RANK`, `DENSE_RANK`) for top-N-per-group patterns as an alternative to CTEs where the DB optimizer handles them well.

**7.23.3 Index Strategy — Design, Not Accident**

- **Composite index column order matters**: `INDEX(a, b, c)` is used by queries filtering on `a`, `a+b`, or `a+b+c`. A query filtering only on `b` or `c` does **not** use this index. Place the most selective column (highest cardinality) first.
- **Covering index**: include all columns a query needs in the index itself so the engine never touches the base table row. Ideal for high-frequency read queries on a known column set.
- **Prefix index**: for long `VARCHAR`/`TEXT` columns, index only the first N chars to reduce index size — but this loses the covering index benefit.
- **Index for sorting**: an index on `ORDER BY` columns eliminates `Using filesort`. For composites, the sort column must come *after* the equality filter column in the index definition.
- **Avoid over-indexing**: every index slows down `INSERT`/`UPDATE`/`DELETE`. Audit unused indexes with `sys.schema_unused_indexes`.
- **Remove redundant indexes**: `INDEX(a)` is made redundant by `INDEX(a, b)`. Use `sys.schema_redundant_indexes` to identify and clean them.

**7.23.4 SQL Best Practices — Non-Negotiable Rules**

- **Never use `SELECT *` in production code** — always name your columns. `SELECT *` fetches unneeded data and prevents covering index usage.
- **Avoid non-sargable predicates** (predicates that cannot use an index):
  ```sql
  -- ❌ Bad: function wrapping the column prevents index usage
  WHERE YEAR(created_at) = 2024

  -- ✅ Good: range predicate on the column directly — fully sargable
  WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
  ```
  **Why `>= AND <` instead of `BETWEEN`**: `BETWEEN` is inclusive on BOTH ends. On a `DATETIME`/`TIMESTAMP` column, `BETWEEN '2024-01-01' AND '2024-12-31'` only captures up to `2024-12-31 00:00:00` — rows at `2024-12-31 09:00:00` or `2024-12-31 23:59:59` are **silently missed**. The `>= AND <` (end-exclusive) pattern captures the entire range correctly regardless of time component. Rule: use `BETWEEN` only on pure `DATE` columns; always use `>= AND <` on `DATETIME`/`TIMESTAMP`.
- **LIMIT before JOIN on large result sets** — filter early, then join the small result set.
- **`UNION ALL` instead of `UNION`** when duplicate elimination is not required — `UNION` sorts and deduplicates, which is expensive and unnecessary when source data is already distinct.
- **Beware implicit type casting**: `WHERE user_id = '123'` on an `INT` column casts every row to string, preventing index usage. Always match the literal type to the column type.
- **Parameterized queries always**: never string-concatenate user input into SQL under any circumstances. Use `db.QueryContext(ctx, query, args...)` — the driver handles parameterization safely.
- **Transaction isolation awareness**: understand `READ COMMITTED` vs `REPEATABLE READ` behavior for the queries in each usecase. Phantom reads and non-repeatable reads must be explicitly considered when writing transactional usecases (see 7.6 UnitOfWork).

---

### 7.24. Cognitive Complexity Reduction — Method Extraction Pattern

SonarQube rule `go:S3776` flags any function with a Cognitive Complexity score above 15. Deep nesting (`if` inside `if` inside `for` inside `if`) is the primary driver of high scores. The mandatory resolution strategy is **method extraction with early-return guard clauses**.

**Anti-pattern: monolithic function with nested ifs (Complexity ≈ 31)**

```go
// ❌ BAD: one function doing too many things, deeply nested
func (u *FooUsecase) GetDetail(ctx context.Context, id string) (map[string]any, error) {
    data, err := u.repo.Get(ctx, id)
    if err == sql.ErrNoRows { return nil, nil }
    if err != nil { return nil, err }

    if data != nil {
        keys := []string{"id", "created_at", ...} // re-allocated every call
        for _, k := range keys { delete(data, k) }

        if u.enrichRepo != nil {
            extra, err := u.enrichRepo.Get(ctx, id)
            if err == nil && extra != nil {
                if extra.FieldA != nil { data["a"] = *extra.FieldA }
                if extra.FieldB != nil { data["b"] = *extra.FieldB }
                // ... 5+ more nil checks inline
            }
        }
    }
    return data, nil
}
```

**Preferred pattern: extraction + early-return (Complexity ≤ 5 per function)**

```go
// ✅ GOOD: package-level slice — allocated once, never per-call
var internalKeys = []string{"id", "created_at", "updated_at"}

func (u *FooUsecase) GetDetail(ctx context.Context, id string) (map[string]any, error) {
    data, err := u.repo.Get(ctx, id)
    if err == sql.ErrNoRows { return nil, nil } // guard: not-found
    if err != nil { return nil, err }           // guard: other error
    if data == nil { return nil, nil }          // guard: empty map

    for _, k := range internalKeys { delete(data, k) }
    u.enrichFromExternal(ctx, id, data)         // delegated, isolated
    return data, nil
}

// enrichFromExternal is a private method with its own early-return guards.
// Failures are silent (degraded-mode): the caller still receives base data.
func (u *FooUsecase) enrichFromExternal(ctx context.Context, id string, data map[string]any) {
    if u.enrichRepo == nil { return }           // guard: optional dependency

    extra, err := u.enrichRepo.Get(ctx, id)
    if err != nil || extra == nil { return }    // guard: enrich unavailable

    applyIfNotNil(data, "a", extra.FieldA)
    applyIfNotNil(data, "b", extra.FieldB)
    applyIfNotNil(data, "c", extra.FieldC)
}

// applyIfNotNil is a pure utility — no branching at the call site.
func applyIfNotNil(dst map[string]any, key string, val *string) {
    if val != nil { dst[key] = *val }
}
```

**Rules to enforce:**

1. **Hard limit per function**: Cognitive Complexity ≤ 15 (SonarQube default). Aim for ≤ 10 in new code.
2. **Early-return for all guards** — eliminate the `if data != nil { ... entire body ... }` anti-pattern. Invert conditions, guard early, and keep the happy path at the leftmost indentation level.
3. **Extract enrichment logic into private methods** — any block that calls an optional/secondary dependency (e.g., a second repo to enrich results) must be a separate method. This keeps the orchestrator function readable and the enrichment independently testable.
4. **Package-level slice/map constants** — any slice of string keys or configuration values that is reused across calls must be declared at the package level (`var` or `const`), never inside a function body where it is re-allocated on every invocation.
5. **Typed pointer-dereference helpers** (`applyIfNotNil`, `applyInt64IfNotNil`, etc.) — when a struct has multiple optional pointer fields that must be written into a `map`, use a shared helper to eliminate the repetitive `if field != nil { data[k] = *field }` pattern. Each helper is a pure function with a complexity of 1.
6. **Naming convention for extracted methods**: prefix private helper methods with a verb that describes the concern — `enrichFrom*`, `buildFrom*`, `applyTo*` — so their intent is immediately clear to reviewers.