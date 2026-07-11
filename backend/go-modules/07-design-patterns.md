# Go Module: Design Principles & Patterns
> **Belongs to:** [clean_architecture_go.md](../clean_architecture_go.md) — Living Document
> **Last Updated:** See Changelog in `clean_architecture_go.md` Section 0
> **Sections:** §7.26 SOLID Taxonomy · §7.27 DRY Boundary · §7.28 ISP · §7.29 LSP · §7.30 OCP · §7.31 Functional Options · §7.32 Error Taxonomy
> **Keywords:** solid, dry, isp, lsp, ocp, srp, dip, interface segregation, liskov, open closed, strategy pattern, functional options, error taxonomy, errs, sentinel error
> **Target Folder/Packages:** `internal/domain/`, `internal/usecases/`, `errs/`

> **MANDATORY — AI Agent Directive:** Every engineer and AI agent working on this codebase MUST read this module whenever:
> - Writing or reviewing any `domain/*.go` interface definition
> - Evaluating a refactoring that involves consolidating structs or interfaces across layers
> - Generating new `usecases/`, `repositories/`, or `clients/` files
> - Performing code review or SonarQube gap analysis
>
> Violations of rules in this module are **blocking issues**. Do not proceed with code generation or review without applying these constraints.

---

## §7.26. SOLID Principles Taxonomy — Explicit Labeling & Enforcement

> **Purpose:** SOLID is applied implicitly throughout this codebase (see §6.1, §7.6, §7.7, §7.21, §7.22). This section formalizes what each principle means in the context of this architecture, so engineers and AI agents can identify and name violations accurately.

---

### §7.26.1 S — Single Responsibility Principle (SRP)

**Definition:** A module, class, or function must have **exactly one reason to change** — it must serve exactly one actor or one cohesive business policy.

**In this codebase:**
- Each `*_usecase.go` file orchestrates one business capability only (e.g., `CreateOrderUsecase`, `CancelOrderUsecase`) — not a monolithic service file.
- `internal/repositories/` are responsible for persistence only — no business logic, no HTTP concepts.
- `internal/handlers/` are responsible for request binding and response serialization only — no business logic.
- `internal/clients/` handle outbound API communication only — no business logic.

**AI Agent Directive:**
- If a usecase file calls `log.Fatal`, `os.Exit`, or accesses `http.Request`/`http.ResponseWriter` directly → **SRP violation. Refactor immediately.**
- If a repository file contains business decision logic (e.g., `if user.IsVIP { ... }`) → **SRP violation. Move to usecase.**

---

### §7.26.2 O — Open-Closed Principle (OCP)

**Definition:** Software modules must be **open for extension, closed for modification**. Adding a new business variant must NOT require modifying existing, tested code — only adding new implementations.

**In this codebase:**
- The `domain.EventPublisher` interface (§7.21) allows adding Kafka or RabbitMQ adapters without touching usecases.
- The `domain.TokenValidator` interface (§7.7) allows adding JWT or API-key validation strategies without modifying the auth middleware.

**Anti-Pattern (OCP Violation):**
```go
// ❌ BAD: usecase grows indefinitely as new discount types are added
func (u *OrderUsecase) ApplyDiscount(order *domain.Order) {
    switch order.UserType {
    case "VIP":
        order.Total *= 0.8
    case "MEMBER":
        order.Total *= 0.9
    case "STAFF":     // newly added — requires modifying this file
        order.Total *= 0.7
    }
}
```

**Preferred Pattern (Strategy — OCP Compliant):**
```go
// ✅ GOOD: new discount types are added as new implementations, not as new cases
// internal/domain/discount.go — port, defined in domain layer
type DiscountStrategy interface {
    Apply(order *Order) float64
}

// internal/usecases/order_usecase.go
type OrderUsecase struct {
    discountStrategy domain.DiscountStrategy
}

func (u *OrderUsecase) ApplyDiscount(order *domain.Order) {
    order.Total = u.discountStrategy.Apply(order)
}

// internal/clients/discount/ — multiple adapters, each closed for modification
type VIPDiscount struct{}
func (d VIPDiscount) Apply(o *domain.Order) float64 { return o.Total * 0.8 }

type MemberDiscount struct{}
func (d MemberDiscount) Apply(o *domain.Order) float64 { return o.Total * 0.9 }
```

**AI Agent Directive:**
- If a usecase contains a `switch-case` or `if-else` chain that evaluates a type enum and the complexity is > 2 cases → **evaluate OCP compliance. Propose Strategy Pattern refactor.**
- Never modify an existing concrete strategy implementation to accommodate a new type — add a new implementation.

---

### §7.26.3 L — Liskov Substitution Principle (LSP)

**Definition:** Any concrete implementation of an interface must be **behaviorally substitutable** for the interface itself, without altering the correctness of the program. A caller depending on the interface must not need to know which concrete implementation it is using.

**In this codebase:**
- `NoopAuditLogger` (§7.16) is a canonical example of correct LSP: substituting it for the real `AuditLogger` does not break any usecase — behavior is safely degraded (no-op), not corrupted.
- All `domain.Cache` implementations (Redis, in-memory) must satisfy the same contract: `Get` returns `(value, nil)` on hit, `(nil, ErrCacheMiss)` on miss — never `(nil, nil)`.

**LSP Violation Patterns (PROHIBITED):**
```go
// ❌ BAD: concrete implementation throws an error that the interface contract does not declare
type MockRepository struct{}
func (r *MockRepository) FindByID(ctx context.Context, id int64) (*domain.User, error) {
    panic("not implemented") // LSP violation: caller cannot substitute this safely
}

// ❌ BAD: concrete implementation silently swallows error and returns zero-value
func (r *CachedRepository) FindByID(ctx context.Context, id int64) (*domain.User, error) {
    user, err := r.cache.Get(ctx, id)
    if err != nil {
        return &domain.User{}, nil // LSP violation: returns empty User instead of propagating the error
    }
    return user, nil
}
```

**Rules:**
1. All interface implementations must honor the **same error contract** defined by the domain interface. If the interface returns `error`, implementations must propagate errors — never swallow silently.
2. All mock implementations (in `mocks/`) generated via `mockery` must never use `panic("not implemented")`. Use `mockery`'s return value configuration or `testify/mock` `Return(nil, someErr)` instead.
3. Concrete implementations must not extend behavior in ways that surprise callers (no hidden side effects beyond the interface contract).

**AI Agent Directive:**
- When generating or reviewing interface implementations, verify: does this implementation honor the full behavioral contract of the interface, including error semantics?
- If a concrete type's method returns a zero-value struct instead of an error when an operation fails → **LSP violation. Fix the error propagation.**

---

### §7.26.4 I — Interface Segregation Principle (ISP)

**Definition:** Interfaces must be **client-specific and role-based**. No client (usecase) should be forced to depend on methods it does not use. Prefer many small, focused interfaces over one large "Fat Interface."

**In this codebase:**
- `domain.Cache` is correctly segregated: only `Get`, `Set`, `Delete` — not a full Redis client.
- Each repository interface in `internal/domain/` should be segmented by the **minimum set of methods** a specific usecase actually needs.

**Anti-Pattern (ISP Violation — "Fat Interface"):**
```go
// ❌ BAD: one giant interface forces ALL usecases to depend on ALL methods
type UserRepository interface {
    FindByID(ctx context.Context, id int64) (*User, error)
    FindByEmail(ctx context.Context, email string) (*User, error)
    FindAll(ctx context.Context, filter UserFilter) ([]*User, int64, error)
    Create(ctx context.Context, user *User) error
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id int64) error
    BulkCreate(ctx context.Context, users []*User) error
    ExistsByEmail(ctx context.Context, email string) (bool, error)
}

// LoginUsecase only needs FindByEmail — but is forced to depend on the entire interface
type LoginUsecase struct {
    repo domain.UserRepository // pulls in 7 unused method contracts
}
```

**Preferred Pattern (Role Interfaces):**
```go
// ✅ GOOD: segregated by role — each usecase depends only on what it needs
type UserReader interface {
    FindByID(ctx context.Context, id int64) (*User, error)
    FindByEmail(ctx context.Context, email string) (*User, error)
}

type UserWriter interface {
    Create(ctx context.Context, user *User) error
    Update(ctx context.Context, user *User) error
}

type UserDeleter interface {
    Delete(ctx context.Context, id int64) error
}

// LoginUsecase depends only on the minimal reader contract
type loginUsecase struct {
    repo domain.UserReader
}

// AdminUsecase may need broader access
type adminUsecase struct {
    repo interface {
        domain.UserReader
        domain.UserWriter
        domain.UserDeleter
    }
}
```

> **Pragmatic Rule:** For simple CRUD entities with a small surface area (≤ 5 methods), a single interface is acceptable. Apply role-based segregation when a usecase provably only needs a subset of methods, or when you need to mock a minimal interface for testing isolation.

**AI Agent Directive:**
- When generating a new `domain.*Repository` interface for a usecase, **list only the methods that usecase will call.** Do not pre-populate a full CRUD interface speculatively.
- If an existing interface has > 7 methods and a new usecase only calls 1–2 of them → **propose ISP segregation. Create a role interface.**

---

### §7.26.5 D — Dependency Inversion Principle (DIP)

> **Note:** DIP is the most explicitly documented SOLID principle in this codebase. See §6.1 (Constructor Injection), §7.7 (TokenValidator), §7.16 (AuditLogger), §7.21 (EventPublisher), and §7.22 (clients/ adapters). This section provides a unified reference and the enforcement directive.

**Definition:** High-level modules (Domain, Usecase) must not depend on low-level modules (Infrastructure, Frameworks). Both must depend on **abstractions (Interfaces)** defined in the high-level module.

**Rules (Consolidated from all sections):**
1. Usecases must NEVER import `database/sql`, `go-redis/v9`, `segmentio/kafka-go`, `amqp091-go`, or any third-party infrastructure library directly.
2. All infrastructure dependencies must be injected via interfaces defined in `internal/domain/`.
3. The composition root (`main.go`) is the ONLY file where concrete implementations are instantiated and wired together.
4. Tests must use mocked interfaces (from `mocks/`), never real infrastructure.

**AI Agent Directive:**
- Scan any `usecase` file's import block. If it contains any of the following packages → **DIP violation. Replace with domain interface:**
  - `database/sql`, `sqlx`, `gorm.io/gorm`
  - `go-redis/v9`, `redis`
  - `segmentio/kafka-go`, `rabbitmq/amqp091-go`
  - `net/http` (except in handlers)
  - Any `third-party` SDK not defined through a domain port

---

## §7.27. DRY Principle — Knowledge Duplication vs. Architectural Decoupling

> **CRITICAL CONSTRAINT:** The DRY (Don't Repeat Yourself) principle applies to **Business Logic (Knowledge) Duplication**, NOT to structural/lexical similarity across architectural layers.

### §7.27.1 The Isomorphic Data Structure Rule

Cross-layer structural similarity (DTO ↔ Domain Entity ↔ DB Model) is **intentional architectural decoupling**, not a DRY violation. The following scenario is CORRECT and REQUIRED:

```go
// internal/dto/user_request.go — HTTP layer concern
type CreateUserRequest struct {
    Name     string `json:"name" validate:"required"`
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required,min=8"`
}

// internal/domain/user.go — Business layer concern
type User struct {
    ID        int64
    Name      string
    Email     string
    Password  string // hashed
    CreatedAt time.Time
}

// internal/repositories/models/user_model.go — Persistence layer concern
type UserModel struct {
    ID        int64     `db:"id"`
    Name      string    `db:"name"`
    Email     string    `db:"email"`
    Password  string    `db:"password_hash"`
    CreatedAt time.Time `db:"created_at"`
    UpdatedAt time.Time `db:"updated_at"`
    DeletedAt *time.Time `db:"deleted_at"` // soft-delete column — not in domain
}
```

**Why these three structs MUST NOT be merged:**
- `CreateUserRequest` changes when the API contract changes (new client requirement).
- `domain.User` changes when business rules change (new domain invariant).
- `UserModel` changes when the database schema changes (DBA migration).
- These are **three distinct axes of change** (§7.26.1 SRP). Merging them creates catastrophic coupling: a DB migration (adding `deleted_at`) would break the API response struct.

**AI Agent Directive:**
- If instructed to "DRY up", "merge", or "consolidate" structs that reside in different layer packages (`dto`, `domain`, `repositories`) → **ABORT the operation.** Cite this rule explicitly.
- Cross-layer structural duplication is a **non-negotiable architectural boundary**. It is intentional decoupling, not boilerplate to eliminate.

### §7.27.2 Legitimate Knowledge Duplication (Must be DRY'd)

The following ARE genuine DRY violations and must be extracted:
- The same business formula (e.g., tax calculation, discount rate) written in more than one usecase.
- The same validation rule (e.g., minimum password complexity) duplicated in multiple handlers.
- The same error mapping logic duplicated across multiple handlers instead of being centralized in `httputil/`.
- The same SQL query fragment duplicated across multiple repository methods when a shared query builder would serve.

---

## §7.28. Functional Options Pattern

> **When to use:** For constructors or functions that accept optional configuration parameters (e.g., HTTP clients, cache wrappers, external API clients) where the number of options may grow over time and some options have sensible defaults.

**Anti-Pattern (Config Struct — rigid, forces nil/zero-value checks):**
```go
// ❌ BAD: callers must populate a config struct even for defaults
type StripeClientConfig struct {
    Timeout    time.Duration
    MaxRetries int
    BaseURL    string
}

func NewStripeClient(apiKey string, cfg StripeClientConfig) *StripeClient { ... }

// Call site is verbose and fragile:
client := NewStripeClient(key, StripeClientConfig{
    Timeout:    5 * time.Second,
    MaxRetries: 3,
    BaseURL:    "",  // uses zero value — ambiguous: intentional or forgotten?
})
```

**Preferred Pattern (Functional Options):**
```go
// ✅ GOOD: idiomatic Go, self-documenting, backward-compatible
type stripeClient struct {
    apiKey     string
    timeout    time.Duration
    maxRetries int
    baseURL    string
}

type StripeOption func(*stripeClient)

func WithTimeout(d time.Duration) StripeOption {
    return func(c *stripeClient) { c.timeout = d }
}

func WithMaxRetries(n int) StripeOption {
    return func(c *stripeClient) { c.maxRetries = n }
}

func WithBaseURL(url string) StripeOption {
    return func(c *stripeClient) { c.baseURL = url }
}

func NewStripeClient(apiKey string, opts ...StripeOption) domain.PaymentGateway {
    c := &stripeClient{
        apiKey:     apiKey,
        timeout:    5 * time.Second, // sensible default
        maxRetries: 3,               // sensible default
        baseURL:    "https://api.stripe.com",
    }
    for _, opt := range opts {
        opt(c)
    }
    return c
}

// Call site is clean and explicit:
client := NewStripeClient(apiKey, WithTimeout(10*time.Second))
```

**Rules:**
1. Apply Functional Options for any constructor in `internal/clients/` that has more than 2 optional parameters.
2. Always define sensible defaults inside the constructor body, before applying options.
3. Option functions must be exported (`WithXxx`) and documented.
4. Do NOT use Functional Options for mandatory parameters — those remain as positional constructor arguments.

**AI Agent Directive:**
- When generating a new `internal/clients/` adapter with a constructor, prefer Functional Options if the adapter has optional timeout, retry, or URL config.
- Never use a Config Struct with all-optional fields as the sole mechanism for configuring a client — it makes default values ambiguous.

---

## §7.29. Error Taxonomy — `errs/` Package Definition

> **MANDATORY RULE:** All application-level sentinel errors and error codes must be defined in a single `errs/` package at the project root. This is the **single source of truth** for error classification across all layers.

### §7.29.1 Package Structure

```
errs/
├── errors.go      # Sentinel error variables
└── codes.go       # Application error code constants (for response envelope)
```

### §7.29.2 Sentinel Error Definition

```go
// errs/errors.go
package errs

import "errors"

// Domain-level sentinel errors — returned by usecase layer, mapped to HTTP by handler layer.
// Repository layer wraps driver errors and maps them to these sentinels.
var (
    // Resource-related
    ErrNotFound     = errors.New("resource not found")
    ErrAlreadyExist = errors.New("resource already exists")
    ErrConflict     = errors.New("resource conflict")

    // Auth-related
    ErrUnauthorized = errors.New("unauthorized")
    ErrForbidden    = errors.New("forbidden")

    // Input-related
    ErrValidation   = errors.New("validation failed")
    ErrBadRequest   = errors.New("bad request")

    // System-related
    ErrInternal     = errors.New("internal server error")
    ErrUnavailable  = errors.New("service unavailable")
    ErrTimeout      = errors.New("operation timed out")
)
```

### §7.29.3 Application Error Code Constants

```go
// errs/codes.go
package errs

// Application-level error codes for the response envelope's "code" field (see §6.3).
// Format: HTTP-status-prefix + sequential number (e.g., 40001 = 400 + 001).
// These codes are documentation-referenced — they MUST be listed in API docs.
const (
    CodeValidationFailed  = "40001" // maps to ErrValidation / HTTP 422
    CodeUnauthorized      = "40101" // maps to ErrUnauthorized / HTTP 401
    CodeForbidden         = "40301" // maps to ErrForbidden / HTTP 403
    CodeNotFound          = "40401" // maps to ErrNotFound / HTTP 404
    CodeConflict          = "40901" // maps to ErrConflict / HTTP 409
    CodeInternal          = "50001" // maps to ErrInternal / HTTP 500
    CodeUnavailable       = "50301" // maps to ErrUnavailable / HTTP 503
    CodeTimeout           = "50401" // maps to ErrTimeout / HTTP 504
)
```

### §7.29.4 Error Propagation Contract (Cross-layer)

```
Repository Layer:
  driver error (sql.ErrNoRows, redis.Nil)
    → fmt.Errorf("repository.FindByID id=%d: %w", id, errs.ErrNotFound)

Usecase Layer:
  repository error
    → fmt.Errorf("usecase.GetUser: %w", err)
    → errors.Is(err, errs.ErrNotFound) // for conditional business logic

Handler Layer:
  usecase error
    → switch { case errors.Is(err, errs.ErrNotFound): httputil.Error(c, 404, errs.CodeNotFound, err) }
```

**Rules:**
1. NEVER define sentinel errors outside `errs/`. Using `errors.New(...)` inline in any other package is prohibited.
2. NEVER use raw string error codes outside `errs/codes.go`. All codes must be referenced as `errs.CodeXxx` constants.
3. Repositories map driver-level errors to `errs.*` sentinels at the boundary — usecases never receive `sql.ErrNoRows`.
4. Handlers use `errors.Is()` to match sentinels — never string comparison.

**AI Agent Directive:**
- If generating repository code that handles `sql.ErrNoRows` → wrap and return `errs.ErrNotFound`, not the raw driver error.
- If generating handler code that reads an error message string for comparison → **PROHIBITED. Use `errors.Is(err, errs.ErrXxx)` instead.**
- If instructed to add a new error type → add it to `errs/errors.go` and a corresponding code to `errs/codes.go`. Never create ad-hoc errors in usecase or handler files.
