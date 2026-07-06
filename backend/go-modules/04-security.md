# Go Module: Security
> **Belongs to:** [clean_architecture_go.md](../clean_architecture_go.md) — Living Document
> **Last Updated:** See Changelog in `clean_architecture_go.md` Section 0
> **Sections:** §7.7 · §7.11 · §7.12 · §7.16
> **Keywords:** security, authentication, auth, authorization, token, jwt, api key, rate limit, versioning, API version, audit, audit trail, audit log, middleware
> **Target Folder/Packages:** `internal/middleware/auth/`, `internal/middleware/ratelimit/`

---

### 7.7. Authentication & Authorization Hook
Since this is a base project, the auth middleware should be generic/pluggable (e.g., a `TokenValidator` interface) rather than hardcoded to a single mechanism, allowing derivative projects to plug in opaque tokens, JWTs, or API keys as needed.

---

### 7.11. Rate Limiting & Security Headers
Add middleware for rate limiting (token bucket algorithm, backed by Redis), CORS configuration, and security headers (helmet-equivalent), enabled by default and configurable via environment variables.

---

### 7.12. Versioning Strategy
Define an API versioning strategy upfront (URL path `/v1/`, header-based, or query parameter) to keep it consistent across all derivative projects.

---

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
