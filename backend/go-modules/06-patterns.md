# Go Module: Patterns
> **Belongs to:** [clean_architecture_go.md](../clean_architecture_go.md) — Living Document
> **Last Updated:** See Changelog in `clean_architecture_go.md` Section 0
> **Sections:** §7.5 · §7.6 · §7.15 · §7.21 · §7.22
> **Keywords:** context, transaction, atomic, db debug, broker, kafka, rabbitmq, resilience, retry, circuit breaker, timeout, bulkhead, fail-fast
> **Target Folder/Packages:** `internal/usecases/`, `internal/domain/`

---

### 7.5. Context Propagation
- Enforce the convention: `context.Context` must be the first parameter in every usecase, repository, and handler-invoked function signature, to support timeout, cancellation, and trace propagation.
- `context.Context` is the single transport mechanism for cross-cutting, per-request values (request-id, actor/user-id, trace span) — never use global variables or package-level state to pass request-scoped data between layers.
- Use unexported, typed context keys (e.g., `type ctxKey string; const requestIDKey ctxKey = "request_id"`) instead of raw strings, to avoid key collisions across packages.
- Provide a small `ctxutil` (or extend `httputil`) package with typed getter/setter helpers — e.g., `ctxutil.WithRequestID(ctx, id)` / `ctxutil.RequestID(ctx)` — so every layer reads/writes context values through the same accessor instead of duplicating `context.Value` calls.
- Set sensible default timeouts at the handler/CLI entry point (e.g., `context.WithTimeout(ctx, 30*time.Second)`) so a hung downstream call (DB, external API) doesn't block indefinitely; usecases and repositories must respect `ctx.Done()`/`ctx.Err()` rather than ignoring it.

---

### 7.6. Transaction Management
There is currently no pattern for cross-repository database transactions (e.g., a usecase that must insert into two tables atomically). Introduce a `UnitOfWork` pattern or transaction manager interface in the domain layer, so the usecase can control commit/rollback without depending on driver-specific transaction details.

---

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

---

### 7.21. Message Broker — Kafka & RabbitMQ (Generated On Demand)
Same principle as MongoDB and Elasticsearch in 7.18: not every derivative project needs asynchronous messaging, so Kafka and RabbitMQ are **not wired into the base project by default**. They are scaffolded only when a project explicitly needs event streaming or task/queue-based processing.

**Folder placement**: messaging adapters live in `internal/clients/messaging/` for producers, plus a dedicated `internal/consumers/` package for the consumption side.

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

Define the publishing contract in the domain layer — usecases never import `segmentio/kafka-go` or `amqp091-go` directly:

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
- Each consumed message must carry/propagate a `request_id` (or generate a new one if absent) into context, so the message-processing usecase still benefits from the tracing chain in 7.17.
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

---

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
