# Go Module: Infrastructure
> **Belongs to:** [clean_architecture_go.md](../clean_architecture_go.md) — Living Document
> **Last Updated:** See Changelog in `clean_architecture_go.md` Section 0
> **Sections:** §7.2 · §7.8 · §7.13 · §7.14
> **Keywords:** migration, goose, golang-migrate, seeder, graceful shutdown, signal, CLI, cobra, Dockerfile, docker-compose, infrastructure
> **Target Folder/Packages:** `cmd/`, `database/migration/`, `deploy/`

> **MANDATORY — AI Agent Directive:** Apply all rules in this module when:
> - Setting up a new CLI subcommand or entrypoint (§7.13)
> - Writing or modifying database migration files (§7.2)
> - Configuring graceful shutdown behavior (§7.8)
> - Authoring or modifying Dockerfile or docker-compose (§7.14)
>
> **BLOCKING:** All CLI subcommands MUST reuse the same composition root (DI bootstrap). Never duplicate DB/Redis initialization across subcommands.

---

### 7.2. Database Migration
- No migration strategy is currently defined. Add a tool such as `golang-migrate` or `goose`, plus a `migrations/` directory.
- Provide a separate seeder for dummy/test data.

---

### 7.8. Graceful Shutdown
- `main.go` must handle `SIGTERM`/`SIGINT` and gracefully close DB/Redis connections and the HTTP server (using `context.WithTimeout` during shutdown) to avoid dropping in-flight requests.

---

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

---

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
