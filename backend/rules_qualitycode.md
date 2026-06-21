# Project Coding Rules

> These rules apply to **all AI assistants and developers**. Every rule is enforced by SonarQube. Non-compliance = scan failure.

---

## 1. Database Query Safety

- **SELECT only** by default. Any `UPDATE`, `DELETE`, or destructive operation **must** show an alert/confirmation to the user before execution.

---

## 2. Cognitive Complexity — Max Score: 10 (`go:S3776`)

Every function must stay below complexity score 10. Violating this = SonarQube failure.

| Rule | Detail |
|------|--------|
| **No deep nesting** | Max 2 levels of `if` nesting. Never `if` inside `if` inside `if`. |
| **Early return** | Use `if err != nil { return err }` — never wrap main logic in `else`. |
| **Single responsibility** | One function = one job. If a function exceeds **40 lines**, split it. |
| **Extract switch cases** | Each `case` in a large `switch` must delegate to its own handler function. |
| **Extract sequential blocks** | Long chains of `if err != nil` checks? Each logical step → its own private helper. |

> **Tip:** Calling helper functions does **NOT** add to complexity score. Use helpers freely.

---

## 3. No Duplicate String Literals (`go:S1192`)

If a string literal appears **3 or more times** in a file → extract it to a `const`.

```go
// ❌ Wrong
fmt.Println("application/json")
w.Header().Set("Content-Type", "application/json")
// ... used again

// ✅ Correct
const contentTypeJSON = "application/json"
```

---

## 4. Naming Conventions

### 4a. Variables & Functions — camelCase / PascalCase only (`go:S117`, `go:S100`)

| Scope | Style | Example |
|-------|-------|---------|
| Local variable / param | `camelCase` | `filenameJob`, `responseCode` |
| Private function | `camelCase` | `parseData()` |
| Exported struct / func | `PascalCase` | `JobLogger`, `ExportAtmZona` |
| Acronyms (exported) | All caps | `userID`, `parseJSON`, `apiURL` |

**❌ Never use `snake_case`** — no underscores in variable or function names.
```go
// ❌ Wrong
func DBConx_OldJke() {}
var filename_job string

// ✅ Correct
func DBConxOldJke() {}
var filenameJob string
```

### 4b. Max 7 Parameters Per Function (`go:S107`)

If a function needs more than 7 params → group them into a `struct`.

```go
// ❌ Wrong
func NewUsecase(a, b, c, d, e, f, g, h Repo) {}

// ✅ Correct
type UsecaseDeps struct { A, B, C, D, E, F, G, H Repo }
func NewUsecase(deps UsecaseDeps) {}
```

---

## 5. Git — Ignore AI-Generated Temporary Files

Add these patterns to `.gitignore` to prevent accidental commits of debug/temp files:

```gitignore
# Prefix patterns
debug_*
temp_*
tmp_*
scratch_*
check_*
ai_debug_*
ai_test_*
ai_temp_*

# Suffix patterns
*_backup.*
*_copy.*
*_old.*
*_temp.*
*_debug.*

# Common AI one-off scripts
debug.py
debug.js
debug.ts
debug.sh
test.py
test.js
test.ts
test.sh
scratch.py
scratch.js
scratch.ts
scratch.sh
check.py
check.js
check.ts
check.sh
temp.py
temp.js
temp.ts
temp.sh
verify.py
debug.txt
scratch.txt
temp.txt
notes_temp.txt

# Extensions
*.tmp
*.bak
*.debug
*.scratch
```

---

## 6. AI Sandbox — Folder `medocs/`

Semua file yang dibuat oleh AI untuk keperluan **debugging, testing, eksplorasi, atau percobaan** yang **tidak berkaitan langsung dengan kode produksi** wajib ditempatkan di folder `medocs/`.

### Aturan Wajib

| Aturan | Detail |
|--------|--------|
| **Lokasi wajib** | Semua file debug/test/scratch AI → `medocs/` (bukan root, bukan folder project) |
| **Tidak boleh di-commit** | Folder `medocs/` masuk ke `.gitignore` — tidak boleh masuk ke version control |
| **Wajib catat aktivitas** | Setiap aksi yang dilakukan AI di dalam `medocs/` harus direcord ke `medocs/activity.log` |
| **Tidak ada exception** | Meskipun file "kelihatan kecil" atau "hanya sementara", tetap wajib masuk `medocs/` |

### Struktur Folder

```
medocs/
├── activity.log        ← wajib ada, record semua aksi AI
├── debug/              ← file debugging
├── test/               ← file testing sementara
└── scratch/            ← file eksplorasi / coba-coba
```

### Format `activity.log`

Setiap kali AI membuat, mengubah, atau menghapus file di `medocs/`, wajib append ke `activity.log` dengan format:

```
[YYYY-MM-DD HH:MM] | ACTION   | PATH                        | REASON
[2025-01-15 10:23] | CREATE   | medocs/debug/check_query.go | Debug query pagination issue
[2025-01-15 10:45] | MODIFY   | medocs/test/api_test.py     | Add edge case for null response
[2025-01-15 11:00] | DELETE   | medocs/scratch/temp_calc.js | No longer needed
```

**Field wajib:** `timestamp | action (CREATE/MODIFY/DELETE/RUN) | file path | alasan singkat`

### Tambahkan ke `.gitignore`

```gitignore
# AI Sandbox — jangan pernah commit
medocs/
```

### Contoh Skenario

```
# ✅ Benar — AI ingin test koneksi database
→ Buat file: medocs/debug/check_db_conn.go
→ Append log: [2025-01-15 09:00] | CREATE | medocs/debug/check_db_conn.go | Test DB connection to staging

# ❌ Salah — AI langsung buat file di root atau folder project
→ Buat file: check_db.go  ← DILARANG
→ Buat file: internal/utils/test_temp.go  ← DILARANG
```

---

## Quick Reference Checklist

Before submitting any code, verify:

- [ ] No function exceeds cognitive complexity 10
- [ ] No nesting deeper than 2 levels
- [ ] All functions use early return pattern
- [ ] No function exceeds 40 lines (extract helpers if so)
- [ ] No duplicate string literals (3+ uses → `const`)
- [ ] No `snake_case` names anywhere
- [ ] No function with more than 7 parameters
- [ ] Destructive DB operations have alerts/confirmation
- [ ] Temp/debug files are in `.gitignore`
- [ ] Semua file debug/test/scratch AI dibuat di dalam `medocs/`
- [ ] Setiap aksi AI di `medocs/` tercatat di `medocs/activity.log`
- [ ] Folder `medocs/` sudah masuk `.gitignore`
