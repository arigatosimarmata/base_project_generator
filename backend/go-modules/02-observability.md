# Go Module: Observability
> **Belongs to:** `backend/clean_architecture_go.md` — Living Document
> **Last Updated:** See Changelog in `clean_architecture_go.md` Section 0
> **Sections:** §7.3 · §7.17 · §7.19

---

### 7.3. Logging & Observability
- Adopt a structured logger (`zap` or `zerolog`) as the standard, injecting a request-id/correlation-id into every log entry via middleware.
- Expose health check endpoints (`/healthz`, `/readyz`) for liveness/readiness probes.
- Optional: integrate distributed tracing (OpenTelemetry) and metrics (Prometheus).

**Default Log Configuration**

- **Storage location**: logs are written to a `logs/` directory at the project root. The application must auto-create this directory on startup if it doesn't exist (`os.MkdirAll("logs", 0755)`).
- **File naming convention**: `{app-name}-{yyyy-mm-dd}.log` (e.g., `base-project-2026-06-21.log`), where `app-name` is read from config/env (`APP_NAME`).
- **Rotation policy**: daily rotation with a default 21-day retention, implemented via [`lumberjack`](https://github.com/natefinch/lumberjack) (`MaxAge: 21`) or an equivalent rolling-file writer. Rotation settings should be exposed via config so derivative projects can override the retention window.
- **Configurable Dual output (stdout + file)**: the logger defaults to writing to both stdout and the rotating log file simultaneously. However, this **must be configurable (parameterized)** so file output or stdout can be independently disabled if not needed (e.g., in environments where stdout alone is sufficient and disk writes are undesirable).

```go
logFile := &lumberjack.Logger{
    Filename: fmt.Sprintf("logs/%s-%s.log", appName, time.Now().Format("2006-01-02")),
    MaxAge:   21, // days
    Compress: true,
}

var cores []zapcore.Core
if config.LogToStdout {
    cores = append(cores, zapcore.NewCore(encoder, zapcore.AddSync(os.Stdout), level))
}
if config.LogToFile {
    cores = append(cores, zapcore.NewCore(encoder, zapcore.AddSync(logFile), level))
}

logger := zap.New(zapcore.NewTee(cores...))
```

- **Git exclusion**: add `logs/` to `.gitignore` at the project root so log files are never committed. A `.gitkeep` (or equivalent placeholder) can be added inside `logs/` if the folder itself needs to exist in version control while its contents stay ignored.

---

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
- The `AuditLogger` (7.16) must read `request_id` from context and store it as a column on `audit_logs`, alongside `actor_id`. This closes the loop: given a `request_id` from a user complaint or incident, an engineer can grep application logs for that id *and* query `audit_logs WHERE request_id = ?` to get the complete picture.
- For async/background work (workers, queue consumers), generate a new request-id per job but optionally carry the originating HTTP request-id as a separate `parent_request_id` field, preserving traceability across service/process boundaries.

**Note on distributed tracing**: if OpenTelemetry tracing (7.3) is adopted later, the request-id can be aligned with (or replaced by) the OTel `trace_id`, since both serve the same correlation purpose — avoid maintaining two parallel, unrelated identifiers once distributed tracing is in place.

---

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
- **Dockerfile wiring**: pass the same values as build args into the multi-stage `Dockerfile` (7.14):

```dockerfile
ARG VERSION=dev
ARG COMMIT=none
ARG BUILD_TIME=unknown
RUN go build -ldflags "-X main.version=${VERSION} -X main.commit=${COMMIT} -X main.buildTime=${BUILD_TIME}" -o /app/bin/app ./main.go
```

- **CLI exposure**: add an `app version` Cobra subcommand (7.13) that prints version/commit/build-time.
- **HTTP exposure**: expose a lightweight `GET /version` endpoint returning the same metadata as JSON, following the standard response envelope (6.3).

**7.19.2 Health Check**

Distinguish **liveness** from **readiness** — they answer different questions and should not be conflated into a single endpoint:

- **`GET /healthz` (liveness)** — answers "is the process alive and not deadlocked?". Should be extremely cheap (no downstream calls), just confirming the HTTP server itself can respond.
- **`GET /readyz` (readiness)** — answers "can this instance actually serve traffic right now?". Actively checks downstream dependencies: DB (`db.PingContext(ctx)`), Redis (`PING`), and any optional data stores in use.

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

- **Response format**: `/healthz` and `/readyz` intentionally bypass the standard envelope (6.3) — orchestrators expect a plain HTTP status code (200/503) and minimal body.
- **Embed version in health responses**: include `version`/`commit` in the `/readyz` payload for immediate incident triage.
- **Timeouts matter**: every downstream check in `/readyz` must run under a short `context.WithTimeout` — a hung dependency must not hang the readiness probe itself.
- **Don't leak internals**: keep payload limited to up/down status per dependency, never raw connection strings or error stack traces.
