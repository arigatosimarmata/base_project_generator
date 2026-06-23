# Go Module: Quality
> **Belongs to:** `backend/clean_architecture_go.md` — Living Document
> **Last Updated:** See Changelog in `clean_architecture_go.md` Section 0
> **Sections:** §7.1 · §7.4 · §7.9 · §7.10 · §7.24

---

### 7.1. Testing Strategy
- Provide a `mocks/` directory (generated via `mockery` or `gomock`) for domain interfaces, so usecases can be unit-tested without a real database dependency.
- Convention: every `*_usecase.go` has a corresponding `*_usecase_test.go` using table-driven tests.
- Separate integration tests (build tag `//go:build integration`) for the repository layer that require a real DB/Redis instance (`testcontainers-go` is recommended).

---

### 7.4. Validation Layer
The PRD already references `internal/validation/`, but the underlying library is unspecified. Recommend `go-playground/validator` with a custom error message formatter, kept consistent with the `httputil` response envelope.

---

### 7.9. Linting & CI
- Add a `golangci-lint` configuration (`.golangci.yml`) with baseline rules (errcheck, govet, staticcheck, cyclop for cognitive complexity).
- Provide an example CI pipeline (GitHub Actions): lint → test → build.

---

### 7.10. API Documentation
Generate Swagger/OpenAPI specs via `swaggo/swag` annotations on handlers, so every derivative project automatically ships with up-to-date API documentation.

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
