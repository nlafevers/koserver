# KOPDS / KOSYNC Uniformity Plan

This file tracks the rounds of cross-project uniformity work for KOPDS and KOSYNC.  Each round aligns equivalent behavior so that functions doing the same job have the same names and identical implementations wherever practical.

## Uniformity Round 1 — Completed

Round 1 covered the initial harmonization and uniformity work that brought KOPDS and KOSYNC to a shared operational baseline.  It is fully documented in `project-history.md`.

Highlights of what was standardized:

- `cmd/` and `internal/` project structure
- Viper-based configuration with aligned field names and environment variables
- `database_path`, log-level, and storage-cap settings
- Bcrypt password hashing helpers (`HashPassword`, `CheckPassword`)
- `logger.New` constructor
- SQLite open helpers with WAL mode, permissions, and ping
- `EnforceStorageCap` with shared control flow
- CLI user-management commands (`create-user`, `delete-user`, `change-password`)
- `net/http.ServeMux` routing (replacing chi in KOPDS)
- Request-logging middleware with request IDs and stable log fields
- Docker and compose deployment patterns
- Unit and integration tests covering shared behavior

The current inventory of identical functions and intentional project boundaries is maintained in the [Current Uniformity Inventory](#current-uniformity-inventory) section of this file.

## Uniformity Round 2 — Completed

Round 2 addressed remaining inconsistencies found in a fresh audit.  The full implementation plan, with per-step commits, is in `ur2-roadmap.md`.  Every checklist item (UR2-1.1 through UR2-8.5) and every final-verification item (FINAL-9.1 through FINAL-9.3) is complete.

Phases covered:

1. Baseline cleanup and formatting
2. Toolchain and release security
3. Uniform CLI and database lifecycle
4. Uniform server lifecycle and shared middleware
5. Security behavior
6. KOPDS correctness and performance
7. KOSYNC correctness and performance
8. Module, docs, deployment, and integration
9. Final verification

See `ur2-roadmap.md` for the complete checklist, per-step instructions, and acceptance criteria.

## Uniformity Round 3 — Completed

Round 3 addressed a fresh third audit focused on the risk that two rounds of uniformity work left behind **uniform code that is never wired in** and **legacy code that was never removed**.  The full implementation plan, with per-step commits, is in `ur3-roadmap.md`.  Every checklist item (UR3-1.1 through UR3-4.2) and every final-verification item (FINAL-3.1 through FINAL-3.3) is complete.

The findings below were implemented in `ur3-roadmap.md`.  Each finding names the symbol, its location, and the recommended action.

The audit cross-checked three lenses: per-repo dead-code tracing (every symbol's non-test callers), cross-repo uniformity drift, and security behavior.  The headline result is reassuring on two fronts and actionable on a third:

- **Security:** No new vulnerabilities.  The Go standard-library `govulncheck` finding was already closed in Round 2 (Go 1.26.x).  The KOSYNC registration path was specifically re-checked for an account-takeover hole (a duplicate registration overwriting an existing password hash) and is **safe**: `HandleUserCreate` checks `GetUserHash` first and returns a fake `201` for duplicates without touching the stored hash (`kosync/internal/api/handlers.go:54`).  UR2-5.5 holds.
- **Uniformity:** Largely intact.  Every rate-limit, logging-middleware, storage-cap, SQLite-lifecycle, password, and config-path helper claimed identical is in fact byte-identical — except two entries the inventory over-claims (see Drift Corrections).
- **Bloat:** This is the real Round 3 work.  Both repos carry dead uniformity scaffolding and one duplicated storage-cap code path.

### 3.1 Dead code to prune — KOPDS

Production KOPDS routes all user management through `sqliteUserRepository` (`internal/database/user_repository.go`), wired into auth middleware (`GetByUsername`, `middleware.go:111`) and the CLI (`openCLIStorage`).  The parallel `*Storage` user surface that was copied in to mirror KOSYNC was never wired in.

- **`(*Storage)` user methods** — `CreateUserIfNotExists`, `SaveUser`, `GetUserHash`, `UpdatePassword`, `DeleteUser` (`internal/database/sqlite.go:163-260`).  Zero non-test callers.  **Remove.**
- **`NewStorage`** (`internal/database/sqlite.go:61`).  Zero non-test callers; the live storage-cap path builds `&Storage{db: r.db}` directly (`book_repository.go:811`).  **Remove** (keep the `Storage` struct itself — it carries `EnforceStorageCap`).
- **`UserRepository.Save` / `sqliteUserRepository.Save`** (`internal/domain/interfaces.go:26`, `internal/database/user_repository.go:37`).  Zero non-test callers; the create path uses `CreateUserIfNotExists`, the update path uses `UpdatePassword`.  **Remove** the interface method and its implementation.
- **`(*BookService).GetLinkGenerator`** (`internal/service/book_service.go:103`).  Zero callers anywhere, including tests; handlers receive the `LinkGenerator` directly via `api.NewHandler`.  **Remove.**
- **Never-populated OPDS Atom symbols** (`internal/opds/atom.go`): the `AtomNamespace` const, the `IndirectAcquisition` type and `Link.IndirectAcquisitions` field, and the `Category` type and `Entry.Categories` field are never referenced.  **Remove.**  Lower priority: scalar `Link.Count/Price/Currency`, `Entry.Published/Rights`, `Feed.Icon` are never assigned (harmless `omitempty` slots) — optional cleanup.
- **`go.mod` cosmetic:** `golang.org/x/time` is marked `// indirect` but is a direct dependency (`rate.Every` in `main.go`); `go mod tidy` would promote it.  One-line tidy.

### 3.2 Dead code to prune — KOSYNC

- **`NewSQLite`** (`internal/database/sqlite.go:68`).  Zero callers; production calls `OpenSQLite` directly.  This is the mirror image of KOPDS, which calls `NewSQLite` and leaves `OpenSQLite` wrapper-only.  **Resolve the asymmetry** (see Drift Corrections) and remove the unused wrapper.
- **`GetRequestID`** (`internal/api/context.go:33`).  Zero callers; the request ID is stored in context and emitted through the bound logger, never read back.  **Remove.**
- **`models.User`** (`internal/models/models.go:4`).  Zero references; auth and registration work with raw username + hash strings.  **Remove** (keep `models.Progress`).
- **`UpdateUserPassword`** (`internal/database/sqlite.go:244`).  Reached only by the one-line `UpdatePassword` wrapper added in UR2-3.3.  **Collapse:** inline the body into `UpdatePassword` and delete the legacy name.
- **`SaveUser` indirection** (`internal/database/sqlite.go:209`).  Only caller is `CreateUser`.  Not dead, but a redundant hop; consider inlining.  Also note `CreateUser` (API, blind upsert) and `CreateUserIfNotExists` (CLI, duplicate-guarded) have inconsistent names for adjacent create paths.

### 3.3 Duplicated storage-cap logic — both repos

In both `internal/database/sqlite.go`, the `(*Storage).EnforceStorageCap` method stats the file, computes size, and gates on `capMB`/size — then calls the package-level `enforceStorageCap` free function, which **re-checks `capMB <= 0`, re-stats the file, and re-checks the size threshold** before pruning.  The free function's guards are dead in practice (it is only reached after the method already proved the file is over cap).  **Collapse** the free function into the method so the stat and gate run once.  Keep both apps identical through the change.

### 3.4 Drift corrections (inventory was over-claiming)

Two entries that had been listed under "Currently Identical Functions" in the uniformity inventory had actually diverged and were corrected:

- **`openCLIStorage`** — KOPDS returns `(*sql.DB, domain.UserRepository)` and injects `NewUserRepository`; KOSYNC returns `(*sql.DB, *database.Storage)` and injects `NewStorage`.  Different return type, different injection, different open call (`NewSQLite` vs `OpenSQLite`).  This is the documented clean-architecture split, moved from the identical list into the intentional-boundaries section.
- **`UpdatePassword(username, passwordHash string) error`** — not the same live function in each app.  KOPDS's live update is `sqliteUserRepository.UpdatePassword` (context-based); the `Storage.UpdatePassword` listed here is the dead method slated for removal in 3.1.  KOSYNC's `Storage.UpdatePassword` is live but delegates to `UpdateUserPassword`.  Correct the entry once the dead KOPDS method is removed.

### 3.5 Recommended boundary additions

After pruning, the only intentional cross-app divergence in user management is the storage abstraction itself: **KOPDS owns a `domain.UserRepository` (clean-architecture interface, context-based); KOSYNC owns flat `*Storage` methods.**  This is correct per the roadmap's "do not force domain-specific code into artificial sameness" rule and should be documented as a boundary rather than chased into sameness.  Do **not** rip out the KOPDS repository to match KOSYNC; the minimal correct prune is deleting the unused mirror methods listed in 3.1.

The corrected per-symbol inventory and boundaries are maintained in the [Current Uniformity Inventory](#current-uniformity-inventory) section of this file.

---

## Current Uniformity Inventory

This section is the single source of truth for which functions are identical across KOPDS and KOSYNC, which differences are intentional boundaries, and the implementation details that make them so.  Update this section whenever a cross-project change is made.

### Currently Identical Functions

- `runCLI`
- `printUsage`
- `passwordFromArgs`
- `readPasswordInteractively`
- `HashPassword`
- `CheckPassword`
- `logger.New`
- `logger.NewCLI`
- `OpenSQLite`
- `Migrate`
- `EnforceStorageCap`
- `vacuum`
- `resolveExecutablePaths`
- `resolvePath`
- `generateRequestID`
- `IPRateLimiter`
- `NewIPRateLimiter`
- `GetLimiter`
- `clientIP`
- `RateLimitMiddleware`
- HTTP Routing (native `net/http.ServeMux`)

### Currently Identical Config Fields

- `RateLimitEnabled`
- `RateLimitPerMinute`
- `RateLimitBurst`
- `TrustProxyHeaders`

### Intentional Project Boundaries

- KOPDS owns OPDS catalog, Calibre scanner, image cache, book repository, and link-generation behavior.
- KOSYNC owns KOReader sync protocol handlers, progress storage, registration, and header authentication behavior.
- Database schemas may differ when the stored domain differs, but shared lifecycle helpers must remain identical.
- **User-storage abstraction differs by design:** KOPDS owns a `domain.UserRepository` (clean-architecture interface, context-based, `sqliteUserRepository`); KOSYNC owns flat `*Storage` user methods.  This is a documented boundary, not drift — do not force one app onto the other's shape.
- **`openCLIStorage` differs** as a consequence of the boundary above: KOPDS returns `(*sql.DB, domain.UserRepository)` and injects `NewUserRepository`; KOSYNC returns `(*sql.DB, *database.Storage)` and injects `NewStorage`.  The surrounding CLI flow (`runCLI`, `printUsage`, password helpers) stays identical.
- **KOSYNC owns `NewStorage`:** its `Storage` is the primary storage object; KOPDS builds `&Storage{...}` inline as a thin storage-cap adapter and has no `NewStorage`.  Do not re-add `NewStorage` to KOPDS for the sake of symmetry.
- Both apps open SQLite via `OpenSQLite` directly.  There is no `NewSQLite` wrapper in either app.
- **`UpdatePassword` is intentionally NOT cross-identical:** KOPDS's live update is `sqliteUserRepository.UpdatePassword` (context-based, clean-architecture interface); KOSYNC's is `(*Storage).UpdatePassword`.  This follows the standing rule "do not force domain-specific code into artificial sameness."

### Implementation Notes

- `pruneStorageCapRecords` intentionally differs: KOPDS prunes catalog sync-state rows; KOSYNC prunes progress rows.
- `config.Load` intentionally differs because each project has different domain settings; shared path-resolution helpers remain identical.
- KOPDS has a repository-level `EnforceStorageCap` adapter to satisfy the book-repository interface; KOSYNC calls storage directly.
- `EnforceStorageCap` returns early if `capMB <= 0` at the top of the method, before any file I/O.  Both apps share this guard.
- The storage-cap file is stat-ed exactly once per `EnforceStorageCap` invocation; the package-level `enforceStorageCap` free function is the test seam and its signature must remain stable.
- Database lifecycle is `OpenSQLite` → `Migrate` → inject (`NewBookRepository` / `NewUserRepository` on KOPDS; `NewStorage` on KOSYNC).  There is no `InitDB` helper and no `NewSQLite` wrapper.
- `generateRequestID` allocates 16 bytes via `crypto/rand.Read`, encodes as hex, and falls back to a timestamp-based ID on failure.  Identical in both apps.
- `PRAGMA foreign_keys=ON` is set during SQLite connection init alongside WAL mode in both apps.
- Rate-limit helpers (`IPRateLimiter`, `NewIPRateLimiter`, `GetLimiter`, `clientIP`, `RateLimitMiddleware`) are copy-paste identical.  Config fields `RateLimitEnabled`, `RateLimitPerMinute`, `RateLimitBurst`, and `TrustProxyHeaders` use the same names and defaults.
- Both `create-user` CLI commands fail if the user already exists to prevent accidental overwrites; use `change-password` to update credentials.
- `logger.New` writes to `io.MultiWriter(os.Stderr, file)` when a log path is configured (server mode).  `logger.NewCLI` writes to the log file only, or `io.Discard` when no path is set (CLI mode), keeping terminal output clean.
- Similarity matches involving unrelated `Close` methods are false positives and are not uniformity targets.
