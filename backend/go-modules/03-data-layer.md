# Go Module: Data Layer
> **Belongs to:** [clean_architecture_go.md](../clean_architecture_go.md) — Living Document
> **Last Updated:** See Changelog in `clean_architecture_go.md` Section 0
> **Sections:** §7.18 · §7.20 · §7.23
> **Keywords:** database, sql, mysql, postgresql, redis, cache, mongodb, elasticsearch, singleflight, sql tuning, explain, connection pool, semaphore
> **Target Folder/Packages:** `internal/repositories/`

---

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
- An `internal/repositories/mongo/` package implementing the relevant domain repository interface against `mongo-driver/v2`, following the exact same port/adapter contract used by the SQL repositories.
- Connection setup (`MONGO_URI`, `MONGO_DB_NAME`) added to `config/` and `docker-compose.yml` only for that project, not the shared base.
- A minimal index-management convention (e.g., an `EnsureIndexes(ctx) error` method called once at startup).
- Note: MongoDB is additive, not a replacement for the relational DB — typical use cases are semi-structured/high-write-throughput data (audit/event logs as an alternative to 7.16's relational table, activity feeds).

**7.18.3 Elasticsearch — Generated On Demand**

When full-text search or analytics is needed, the generator scaffolds:
- An `internal/repositories/search/` package implementing a `domain.SearchRepository` port against the official `elasticsearch-go` client.
- A write-side sync strategy must be chosen explicitly per project — either **dual-write** from the usecase layer (simplest, but risks drift) or an **outbox/event-driven sync** (recommended once correctness matters).
- Index mapping/settings should be version-controlled (e.g., as JSON files under `internal/repositories/search/mappings/`) and applied via a CLI subcommand (`app search reindex`), consistent with the Cobra pattern in 7.13.

**Why generate instead of always-include**: keeping MongoDB and Elasticsearch out of the default dependency tree avoids forcing every derivative project to run containers it doesn't need — see 7.14's container-per-project-need principle.

---

### 7.20. Singleflight — Deduplicating Concurrent Identical Requests
When many concurrent requests ask for the exact same data at the same moment, each one independently hitting the DB/downstream service is wasteful and can cause a **thundering herd / cache stampede** — especially right after a cache miss or TTL expiry (7.18.1). The `singleflight` pattern collapses concurrent identical calls into a single in-flight execution, sharing the result across all callers.

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
- **`context.Context` caution**: `singleflight.Do` executes the function once for the *first* caller's context; if that caller cancels, the shared in-flight call can be cancelled for *all* waiting callers too. For request-scoped work where this matters, prefer `singleflight.DoChan`.
- **Error sharing is intentional**: if the underlying call fails, all concurrent callers receive the same error — don't retry inside the singleflight function itself.
- **Pairs with, doesn't replace, caching**: singleflight only deduplicates *concurrent* calls during the brief window a value is being fetched; the cache (7.18.1) still does the steady-state heavy lifting between misses.

**Typical use cases**: cache-miss repopulation, expensive aggregation/report endpoints, external API calls with rate limits, and any `GET` usecase that's read-heavy and idempotent. Avoid applying it to non-idempotent operations (writes, mutations).

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
