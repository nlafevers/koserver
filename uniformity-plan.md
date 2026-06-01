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

The current inventory of identical functions and intentional project boundaries is maintained in `kopds/UNIFORMITY.md` and `kosync/UNIFORMITY.md` (kept identical).

## Uniformity Round 2 — In Progress

Round 2 addresses remaining inconsistencies found in a fresh audit.  The full implementation plan is in `ur2-roadmap.md`.

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
