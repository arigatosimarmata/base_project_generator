# Product Requirements Document (PRD): Generic Go Clean Architecture Boilerplate
You are a Senior Back-End Engineer with 10+ years of production-grade experience.
You specialize in building scalable, secure, and high-performance server-side systems
at Enterprise scale — where correctness, resilience, and maintainability are
non-negotiable, not afterthoughts.

> **CRITICAL INSTRUCTION FOR AI AGENTS (ANTI-FALSE-NEGATIVE & TOKEN EFFICIENCY PROTOCOL):**
> As a Senior Engineer, you MUST strictly balance Token Efficiency with Contextual Accuracy:
> 1. **Index-First Routing:** Never read this entire monolithic document blindly. Check the `Quick Reference Index`, identify the relevant topic, and locate the exact `go-modules/*.md` file.
> 2. **Module-Level Deep Dive (No Blind Grepping):** Order of operations is STRICT and NON-NEGOTIABLE:
>    - FIRST: Use `view_file` on the identified module file to read it fully.
>    - ONLY THEN: Use `grep_search` as a supplementary tool to locate line numbers, NOT as the primary source of truth.
>    - NEVER: Use `grep_search` alone to conclude that a feature/rule does NOT exist.
> 3. **Contextual Read:** You are MANDATED to use `view_file` to read the entirety of that specific module file. Since modules are small and focused, reading the whole module is highly token-efficient and prevents missing conceptual implementations.
> 4. **Challenge Assumptions:** If asked about standard enterprise features (log rotation, resilience, rate limits), assume the concept exists. Search for the *concept/mechanisms*, not just specific strings.
> 5. **Ambiguity Resolution:** If the relevant module file is UNCLEAR from the Index alone, check BOTH plausible module files using `view_file` before drawing any conclusion. Uncertainty is never a justification for a false negative.

---

## AI Agent Prompting Guide (How to Get Best Results)

To maximize this modular PRD design and ensure AI agents (like Gemini/Antigravity) do not miss instructions due to context prioritization or fuzzy search limits, developers MUST use this structured prompt template when initiating tasks:

### 💡 Recommended Prompt Template

```markdown
[1. CONTEXT ANCHORING - Core References]
Reference:
- Parent PRD: @[backend/clean_architecture_go.md]
- Coding Rules: @[backend/rules_qualitycode.md]

[2. ACTION - Specific Task]
Please [write/create/refactor/verify] the [feature name / file path] to...

[3. STRICT CONSTRAINTS - PRD Constraints]
IMPORTANT: You MUST strictly adhere to the following project standards:
- Use pattern [A] (e.g., early return / SELECT only / lumberjack).
- DO NOT use [B] (e.g., no snake_case, no external frameworks without approval).
- Verify if this rule is already documented in the [observability/security] module.
```

### 🛠️ AI Prompt Interpretation & Execution Rules

When executing developer instructions structured using the template above, the AI agent MUST adhere to the following behaviors:

1. **Explicit Reference Resolution (Context Anchoring):** When the developer mentions specific files (e.g., using `@[filename]`), you MUST immediately load and read the contents of those files using `view_file` to capture their requirements before writing any code.
2. **Mandated Cross-Checking:** When the developer instructs you to "cross-check" or "verify" with a rules file (e.g., `@rules_qualitycode.md`), you MUST open that file, identify its constraints, and perform a strict compliance check against your proposed code.
3. **Strict Adherence to Negative Constraints:** Any negative constraint (e.g., "DO NOT", "JANGAN", "never", "prohibited") in the prompt or referenced PRD files represents a blocking rule. You MUST explicitly verify that your final output does not violate any of these constraints.
4. **Triggered Deep Analysis (Keywords: "Analyze/Analisa", "Check/Cek"):** If the developer's prompt contains analysis keywords (e.g., "Analyze", "Analisa", "Check", "Cek", "Investigate"):
   - You MUST first inspect the `Quick Reference Index` to see if the topic is mapped to any module.
   - If mapped, you MUST use `view_file` to read that module file completely.
   - If the topic is NOT in the Index, you MUST perform a repository-wide fuzzy/synonym search across all `go-modules/` files using `grep_search` to verify if the concept is discussed elsewhere under a different name.
   - You are strictly prohibited from declaring a feature or rule "not found" without performing this double-check sequence.

---



## 📌 Quick Reference Index (AI Agent Navigator)

> **AI Agent Instructions**: Read this index first. Use the **"When to Read"** column to identify which module file to load for the current task. Never load the full document when a specific module covers the task — only read the relevant module file.

### Core Rules — Always Read (Section 1–6)

| Section | Topic | Summary |
|---|---|---|
| §1–2 | Overview & Goals | Purpose of the boilerplate: standardization, maintainability, extensibility |
| §3 | Tech Stack | Go 1.20+, Fiber/net-http, MySQL/PostgreSQL, Redis, MongoDB (optional) |
| §4 | Architecture Design | 4 layers: Domain → Repository → Usecase → Handler. Dependency Rule: inward only. |
| §5 | Folder Structure | `internal/domain`, `repositories`, `usecases`, `handlers`, `clients/` |
| §6.1 | Dependency Injection | Constructor injection mandatory; global state prohibited for infra resources |
| §6.2 | DTO | Domain entities must NOT be directly bound to JSON request/response payloads |
| §6.3 | Response Standardization | Envelope: `{code, message, data, meta}` / error: `{code, message, errors, request_id}` |
| §6.4 | Error Handling | Wrap errors per layer, sentinel errors in `errs/`, never swallow silently |

---

### Extended Rules — Read the Module File (`go-modules/`)

| Module File | Target Folder / Packages | Sections Covered | When to Read |
|---|---|---|---|
| [01-infrastructure.md](go-modules/01-infrastructure.md) | `cmd/`, `database/migration/`, `deploy/` | §7.2 DB Migration · §7.8 Graceful Shutdown · §7.13 CLI Cobra · §7.14 Docker | Setting up infra, migration, containerization, CLI subcommands |
| [02-observability.md](go-modules/02-observability.md) | `logger/`, `internal/handlers/health/` | §7.3 Logging · §7.17 Request ID/Tracing · §7.19 Health/Version endpoints | Logger setup, dual-output, `/healthz`, `/readyz`, `X-Request-ID` middleware |
| [03-data-layer.md](go-modules/03-data-layer.md) | `internal/repositories/` | §7.18 Redis/MongoDB/ES · §7.20 Singleflight · §7.23 SQL Tuning | Caching, datastore setup, thundering herd, query performance, EXPLAIN analysis |
| [04-security.md](go-modules/04-security.md) | `internal/middleware/auth/`, `internal/middleware/ratelimit/` | §7.7 Auth Hook · §7.11 Rate Limiting · §7.12 API Versioning · §7.16 Audit Trail | Auth middleware, audit log, rate limiter, security headers, API version strategy |
| [05-quality.md](go-modules/05-quality.md) | `tests/`, `internal/...` (validation/complexity) | §7.1 Testing · §7.4 Validation · §7.9 Linting/CI · §7.10 API Docs · §7.24 Complexity | Unit tests, SonarQube S3776 alert, golangci-lint, swaggo annotations |
| [06-patterns.md](go-modules/06-patterns.md) | `internal/usecases/`, `internal/domain/` | §7.5 Context · §7.6 Transaction · §7.15 Query Debug · §7.21 Broker · §7.22 Resilience | Context propagation, atomic writes, retry/circuit breaker, Kafka/RMQ, DB_DEBUG |
| [07-design-patterns.md](go-modules/07-design-patterns.md) | `internal/domain/`, `internal/usecases/`, `errs/` | §7.26 SOLID Taxonomy · §7.27 DRY Boundary · §7.28 Functional Options · §7.29 Error Taxonomy | SOLID principle enforcement, DRY vs decoupling, ISP/LSP/OCP rules, Functional Options pattern, `errs/` package definition |

### Adding a New Module

When a new architectural pattern is discovered that does not fit any existing module:
1. **Fits an existing module** (thematically related) → append to that module file.
2. **Standalone concern** (> 40 lines, distinct domain) → create a new file: `go-modules/{next-number}-{concern-name}.md`, then add a new row to the table above.
3. **After any module change**: update [CHANGELOG.md](CHANGELOG.md), update this index table if needed, then execute the AI Agent Git Automation Protocol (add → commit → push all affected files).

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

## 7. Extended Architecture Modules

All extended architecture standards are maintained as separate module files in `backend/go-modules/`. Refer to the `📌 Quick Reference Index` above to identify which module to read for the current task.

> **Do not add new content directly to this file below this point.** All new architectural patterns, standards, and guidelines must be added to the appropriate module file in `go-modules/`, with a corresponding Changelog entry in `CHANGELOG.md` and an updated row in the Quick Reference Index above.

---

## 8. Continuous Knowledge Integration (Living Document Protocol)

> **MANDATORY RULE FOR ALL ENGINEERS AND AI AGENTS:** 
> This architecture document is a **Living Knowledge Base**. It is strictly prohibited to leave newly discovered architectural improvements, structural bug fixes, or optimized patterns undocumented. 

During the software development lifecycle, if an AI agent or engineer discovers a new optimization, identifies an architectural anti-pattern, or formulates a highly effective technical standard (knowledge/skill), **it MUST be retroactively integrated into this document immediately**. This ensures the boilerplate evolves continuously toward greater efficiency, maintainability, and code quality.

Every new integration MUST be logged in [CHANGELOG.md](CHANGELOG.md).

### Required Update Format:
Every update entry must strictly follow this list-based schema in `CHANGELOG.md`:
- **[YYYY-MM-DD]** - **[Short Title of Knowledge/Update]**: [Detailed explanation of WHAT the knowledge is, the anti-pattern resolved, or efficiency gained]. *(See Section X.Y for details)*.

### AI Agent Git Automation Protocol
When an AI agent updates this document or any module, the agent MUST immediately execute the following isolated workflow:
1. **Validate**: Ensure the changelog format in `CHANGELOG.md` is strictly followed.
2. **Isolate**: Stage the affected documentation files (`git add backend/clean_architecture_go.md backend/CHANGELOG.md backend/go-modules/...`).
3. **Commit**: Use a semantic commit message, e.g., `docs(arch): update clean architecture base rules with [topic]`.
4. **Push**: Push to the current remote branch (`git push`).
This guarantees the living document is continuously synchronized with the remote repository without risking application code stability.