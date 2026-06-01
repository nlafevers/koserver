# Uniformity Round 2 Implementation Roadmap

This roadmap records the next round of work for bringing KOPDS and KOSYNC closer together while also fixing security, edge-case, test, bloat, and performance issues found during the fresh audit.

## How To Use This Roadmap

When running Go commands in this workspace, prefer a writable cache under `/tmp`:

```bash
GOCACHE=/tmp/kopds-gocache go test ./...
GOCACHE=/tmp/kosync-gocache go test ./...
```

## Git Repository Structure

KOSERVER is documentation only. KOPDS and KOSYNC are separate Git repositories inside this workspace. Code changes inside `kopds/` must be committed in the KOPDS repository. Code changes inside `kosync/` must be committed in the KOSYNC repository. Changes to this roadmap or other root documentation must be committed in the KOSERVER repository.  Do not copy commits across repositories.

## Audit Findings To Address

- KOPDS and KOSYNC still have equivalent code paths that are not copy-paste similar, especially CLI storage setup, server lifecycle, and middleware wiring.
- KOSYNC `cmd/kosync/main.go` is not gofmt-formatted.
- KOPDS still carries the unused `github.com/go-chi/chi/v5` dependency even though routing now uses `net/http.ServeMux`.
- `govulncheck` reports `GO-2026-4971` through the local Go standard library version `go1.26.2`; both projects should build and release with Go `1.26.3` or newer.
- KOSYNC has rate-limiting code, but the server does not wire it into request handling.
- KOSYNC public registration is enabled by default; the preferred default is disabled, with opt-in registration.
- KOSYNC protected responses are intended to use `application/vnd.koreader.v1+json`, but `HandleAuth` overrides the content type with `application/json`.
- KOPDS `ListByAuthor` joins `books` to `books_authors_link` on the wrong column.
- Several KOPDS route handlers ignore `strconv.ParseInt` errors for path IDs.
- Some database list/search code closes rows manually without checking `rows.Err()`.
- KOPDS has duplicate FTS reindex helper logic.
- Docker publishing workflows do not explicitly point to `build/Dockerfile`.
- KOSYNC binary release workflow still looks copied from KOPDS and builds/names the wrong binary.

## Phase 1: Baseline Cleanup And Formatting

Goal: make both repositories mechanically clean before behavioral changes.

- [x] **UR2-1.1 Format KOPDS** (Commit: 235c79a)
  - **Repos:** kopds
  - **Read:** none
  - **Edit:** kopds
  - **Instructions:** Run gofmt on the repository.
  - **Verify:** `cd kopds && gofmt -l . && GOCACHE=/tmp/kopds-gocache go test ./...`
  - **Done when:** gofmt prints nothing and tests pass

- [x] **UR2-1.2 Format KOSYNC** (Commit: e837d82)
  - **Repos:** kosync
  - **Read:** none
  - **Edit:** kosync
  - **Instructions:** Run gofmt on the repository.
  - **Verify:** `cd kosync && gofmt -l . && GOCACHE=/tmp/kosync-gocache go test ./...`
  - **Done when:** gofmt prints nothing and tests pass

- [x] **UR2-1.3 Tidy KOPDS dependencies** (Commit: 28c5561)
  - **Repos:** kopds
  - **Read:** kopds/go.mod
  - **Edit:** kopds/go.mod, kopds/go.sum
  - **Instructions:** Run go mod tidy to remove unused `github.com/go-chi/chi/v5`.
  - **Verify:** `cd kopds && GOCACHE=/tmp/kopds-gocache go test ./...`
  - **Done when:** dependency removed and tests pass

- [x] **UR2-1.4 Confirm KOSYNC dependencies are tidy** (Commit: none)
  - **Repos:** kosync
  - **Read:** kosync/go.mod
  - **Edit:** kosync/go.mod, kosync/go.sum
  - **Instructions:** Run go mod tidy.
  - **Verify:** `cd kosync && GOCACHE=/tmp/kosync-gocache go test ./...`
  - **Done when:** tests pass

Acceptance criteria for Phase 1:
- `gofmt -l .` prints nothing in both app repos.
- `go test ./...` passes in both app repos.
- KOPDS no longer depends on `github.com/go-chi/chi/v5`.

## Phase 2: Toolchain And Release Security

Goal: fix the Go standard library vulnerability and make release automation match both projects.

- [x] **UR2-2.1 Update Go versions in KOPDS** (Commit: 3a2d10f)
  - **Repos:** kopds
  - **Read:** kopds/go.mod
  - **Edit:** kopds/.github/workflows/publish-binaries.yml
  - **Instructions:** Change `go-version` from `'1.25.0'` to `'1.26.x'` in the workflow.
  - **Verify:** `cd kopds && GOCACHE=/tmp/kopds-gocache go test ./...`
  - **Done when:** workflows use the correct version and tests pass

- [x] **UR2-2.2 Update Go versions in KOSYNC** (Commit: 9b1b210)
  - **Repos:** kosync
  - **Read:** kosync/go.mod
  - **Edit:** kosync/.github/workflows/publish-binaries.yml
  - **Instructions:** Change `go-version` to `'1.26.x'` in the workflow.
  - **Verify:** `cd kosync && GOCACHE=/tmp/kosync-gocache go test ./...`
  - **Done when:** workflows use the correct version and tests pass

- [x] **UR2-2.3 Fix Docker publish workflows** (Commits: kopds:eaa84e4, kosync:c5e7764)
  - **Repos:** kopds, kosync
  - **Read:** none
  - **Edit:** kopds/.github/workflows/docker-publish.yml, kosync/.github/workflows/docker-publish.yml
  - **Instructions:** Add `file: build/Dockerfile` under the `docker/build-push-action` block in both workflows.
  - **Verify:** visual inspection of yaml
  - **Done when:** file is specified explicitly

- [x] **UR2-2.4 Fix KOSYNC binary release workflow** (Commit: 9a6a7ad)
  - **Repos:** kosync
  - **Read:** none
  - **Edit:** kosync/.github/workflows/publish-binaries.yml
  - **Instructions:** Change binary name from `kopds` to `kosync` and build path from `./cmd/kopds` to `./cmd/kosync`.
  - **Verify:** `cd kosync && GOCACHE=/tmp/kosync-gocache go test ./...`
  - **Done when:** binary builds with correct name

Acceptance criteria for Phase 2:
- Both binary release workflows use the correct app name and package path.

## Phase 3: Uniform CLI And Database Lifecycle

Goal: make the command-line user management code as identical as practical.

- [x] **UR2-3.1 Storage lifecycle wrappers** (Commit: superseded by TDL-001)
  - **Repos:** kopds, kosync
  - **Read:** none
  - **Edit:** none
  - **Instructions:** Implemented via TDL-001: both apps use OpenSQLite, Migrate, NewSQLite.
  - **Verify:** N/A
  - **Done when:** Completed in TDL-001

- [x] **UR2-3.2 Add KOPDS storage user methods** (Commit: 18b5d21)
  - **Repos:** kopds
  - **Read:** kopds/internal/database/user_repository.go
  - **Edit:** kopds/internal/database/sqlite.go, kopds/internal/database/user_repository_test.go
  - **Instructions:** Add storage methods `CreateUserIfNotExists`, `SaveUser`, `GetUserHash`, `UpdatePassword`, `DeleteUser` using the same SQL behavior currently in `sqliteUserRepository`. Keep the repository interface intact so callers continue to work. Add focused tests in `user_repository_test.go` or a new storage test.
  - **Verify:** `cd kopds && GOCACHE=/tmp/kopds-gocache go test ./internal/database ./cmd/kopds`
  - **Done when:** tests pass; storage methods implemented

- [x] **UR2-3.3 Rename or wrap KOSYNC password update** (Commit: 36e05a4)
  - **Repos:** kosync
  - **Read:** none
  - **Edit:** kosync/internal/database/sqlite.go, kosync/cmd/kosync/main.go
  - **Instructions:** Add a method `UpdatePassword(username, passwordHash string) error` that wraps or replaces `UpdateUserPassword`. Update CLI code to call `UpdatePassword`.
  - **Verify:** `cd kosync && GOCACHE=/tmp/kosync-gocache go test ./internal/database ./cmd/kosync`
  - **Done when:** tests pass; new method name in use by CLI

- [x] **UR2-3.4 Make CLI functions copy-paste similar** (Commits: kopds:3191e0c, kosync:7e11170)
  - **Repos:** kopds, kosync
  - **Read:** none
  - **Edit:** kopds/cmd/kopds/main.go, kosync/cmd/kosync/main.go
  - **Instructions:** Compare and align `runCLI`, `printUsage`, `createUser`, `deleteUser`, `changePassword`, `openCLIStorage`, `passwordFromArgs`, `readPasswordInteractively`. Make them identical except for app names and imports. Prefer the KOPDS dispatch style. Use same stdout/error wording.
  - **Verify:** `cd kopds && GOCACHE=/tmp/kopds-gocache go test ./cmd/kopds ./internal/database && cd ../kosync && GOCACHE=/tmp/kosync-gocache go test ./cmd/kosync ./internal/database`
  - **Done when:** functions are virtually identical and tests pass

Acceptance criteria for Phase 3:
- CLI user-management functions are copy-paste similar across KOPDS and KOSYNC.
- Both apps still create missing databases from CLI commands.
- Duplicate `create-user` still fails from the CLI in both apps.

## Phase 4: Uniform Server Lifecycle And Shared Middleware

Goal: make server startup, shutdown, logging, and shared middleware behavior easier to compare.

- [x] **UR2-4.1 Extract KOSYNC `runServer`** (Commit: 7c130ba)
  - **Repos:** kosync
  - **Read:** none
  - **Edit:** kosync/cmd/kosync/main.go
  - **Instructions:** Change `main()` to match KOPDS: load config, create logger, if CLI args exist call `runCLI`, else call `runServer`. Move server setup into `runServer(cfg *config.Config, log *slog.Logger)`. Use a timeout context for shutdown.
  - **Verify:** `cd kosync && GOCACHE=/tmp/kosync-gocache go test ./cmd/kosync ./internal/api`
  - **Done when:** tests pass; main matches KOPDS structure

- [x] **UR2-4.2 Add KOSYNC config validation** (Commit: 71e8c22)
  - **Repos:** kosync
  - **Read:** none
  - **Edit:** kosync/internal/config/config.go, kosync/cmd/kosync/main.go, kosync/internal/config/config_test.go
  - **Instructions:** Add `func (c *Config) Validate() error` validating: `Port` (1-65535 or 0 for tests), `DatabasePath` (not empty), `LogLevel` (debug/info/warn/error), `StorageCapMB` (non-negative). Call this in `runServer`. Add config tests.
  - **Verify:** `cd kosync && GOCACHE=/tmp/kosync-gocache go test ./internal/config ./cmd/kosync`
  - **Done when:** tests pass; validation runs on startup

- [x] **UR2-4.3 Standardize request ID generation** (Commits: kopds:7a6ca01, kosync:8a8c06d)
  - **Repos:** kopds, kosync
  - **Read:** none
  - **Edit:** kopds/internal/api/middleware.go, kosync/internal/api/middleware.go, kopds/internal/api/middleware_test.go, kosync/internal/api/middleware_test.go
  - **Instructions:** In both apps, update `generateRequestID` to allocate 16 bytes and call `rand.Read`, falling back to timestamp-based on failure, returning hex string. Add/update tests confirming `request_id` exists.
  - **Verify:** `cd kopds && GOCACHE=/tmp/kopds-gocache go test ./internal/api && cd ../kosync && GOCACHE=/tmp/kosync-gocache go test ./internal/api`
  - **Done when:** identically implemented in both and tests pass

- [x] **UR2-4.4 Enable SQLite foreign keys uniformly** (Commits: kopds:6d78b4e, kosync:b338a54)
  - **Repos:** kopds, kosync
  - **Read:** none
  - **Edit:** kopds/internal/database/sqlite.go, kosync/internal/database/sqlite.go, kopds/internal/database/sqlite_test.go, kosync/internal/database/sqlite_test.go
  - **Instructions:** Add `PRAGMA foreign_keys=ON;` during SQLite connection init (alongside WAL). Add tests proving FK violations fail.
  - **Verify:** `cd kopds && GOCACHE=/tmp/kopds-gocache go test ./internal/database && cd ../kosync && GOCACHE=/tmp/kosync-gocache go test ./internal/database`
  - **Done when:** PRAGMA is active and tests pass

- [x] **UR2-4.5 Move disabled storage-cap check to the top** (Commits: kopds:6c2cb5b, kosync:82326df)
  - **Repos:** kopds, kosync
  - **Read:** none
  - **Edit:** kopds/internal/database/sqlite.go, kosync/internal/database/sqlite.go, kopds/internal/database/sqlite_test.go, kosync/internal/database/sqlite_test.go
  - **Instructions:** In `Storage.EnforceStorageCap`, return early if `capMB <= 0` right after getting the logger. Add/update tests proving disabled caps skip file stat.
  - **Verify:** `cd kopds && GOCACHE=/tmp/kopds-gocache go test ./internal/database && cd ../kosync && GOCACHE=/tmp/kosync-gocache go test ./internal/database`
  - **Done when:** early return implemented and tests pass

Acceptance criteria for Phase 4:
- KOSYNC has the same `main` to `runServer` shape as KOPDS.
- Middleware helper names remain aligned.
- Storage-cap disabled behavior is identical and cheap.
- SQLite foreign keys are active in both apps.

## Phase 5: Security Behavior

Goal: reduce public attack surface and make KOSYNC protocol responses consistent.

- [x] **UR2-5.1 Disable KOSYNC public registration by default** (Commit: edc151d)
  - **Repos:** kosync
  - **Read:** none
  - **Edit:** kosync/internal/config/config.go, kosync/config/config.yaml, kosync/deploy/docker-compose.yml, kosync/README.md, kosync/internal/config/config_test.go
  - **Instructions:** Change `disable_registration` default from `false` to `true`. Update sample configs and compose file. Update README text. Add config test.
  - **Verify:** `cd kosync && GOCACHE=/tmp/kosync-gocache go test ./internal/config ./internal/api`
  - **Done when:** default is true and tests pass

- [x] **UR2-5.2 Wire KOSYNC rate limiting** (Commit: 735b0ec)
  - **Repos:** kosync
  - **Read:** none
  - **Edit:** kosync/internal/config/config.go, kosync/internal/api/middleware.go, kosync/cmd/kosync/main.go, kosync/internal/api/handlers_test.go
  - **Instructions:** Add config for `RateLimitEnabled`, `RateLimitPerMinute`, `RateLimitBurst`, `TrustProxyHeaders` with defaults (true, 30, 10, false). Update rate limiting middleware to use clean client IP (checking `X-Forwarded-For` only if trusted). Wrap `/users/create` and protected routes. Add rate limiting tests.
  - **Verify:** `cd kosync && GOCACHE=/tmp/kosync-gocache go test ./internal/api ./cmd/kosync`
  - **Done when:** rate limits are active and tests pass

- [x] **UR2-5.3 Add equivalent KOPDS failed-auth rate limiting** (Commit: 9686b11)
  - **Repos:** kopds
  - **Read:** none
  - **Edit:** kopds/internal/config/config.go, kopds/internal/api/middleware.go, kopds/internal/api/middleware_test.go
  - **Instructions:** Add same config fields. Copy rate limit logic from KOSYNC. Apply limiting only to failed Basic Auth attempts. Add tests.
  - **Verify:** `cd kopds && GOCACHE=/tmp/kopds-gocache go test ./internal/api ./internal/config ./cmd/kopds`
  - **Done when:** rate limits are active on failed auth and tests pass

- [x] **UR2-5.4 Fix KOSYNC response content type** (Commit: 9801308)
  - **Repos:** kosync
  - **Read:** none
  - **Edit:** kosync/internal/api/handlers.go, kosync/cmd/kosync/main.go, kosync/internal/api/handlers_test.go
  - **Instructions:** Remove manual Content-Type setting in `HandleAuth`. Ensure `ContentTypeMiddleware` wraps protected routes and `/users/create`. Verify `application/vnd.koreader.v1+json` is used.
  - **Verify:** `cd kosync && GOCACHE=/tmp/kosync-gocache go test ./internal/api ./cmd/kosync`
  - **Done when:** routes use correct content type and tests pass

- [x] **UR2-5.5 Make KOSYNC registration timing less distinguishable** (Commit: 3543be1)
  - **Repos:** kosync
  - **Read:** none
  - **Edit:** kosync/internal/api/handlers.go, kosync/internal/api/handlers_test.go
  - **Instructions:** Use `randomDelay` equally for new and duplicate registrations. Keep fake success for duplicates. Add test verifying `201` for duplicates and no password change.
  - **Verify:** `cd kosync && GOCACHE=/tmp/kosync-gocache go test ./internal/api ./internal/database`
  - **Done when:** timing mitigated and tests pass

Acceptance criteria for Phase 5:
- KOSYNC public registration is disabled unless explicitly enabled.
- KOSYNC rate limiting is wired into real routes.
- KOPDS has comparable failed-auth protection.
- KOSYNC response content type matches the KOReader protocol.

## Phase 6: KOPDS Correctness And Performance

Goal: fix confirmed KOPDS bugs and reduce unnecessary query work.

- [x] **UR2-6.1 Fix KOPDS author listing query** (Commit: 146ff0d)
  - **Repos:** kopds
  - **Read:** none
  - **Edit:** kopds/internal/database/book_repository.go, kopds/internal/database/book_repository_test.go
  - **Instructions:** In `ListByAuthor`, change the join ON clause from `b.id = bal.author_id` to `b.id = bal.book_id`. Add a regression test with two books and two authors showing correctness.
  - **Verify:** `cd kopds && GOCACHE=/tmp/kopds-gocache go test ./internal/database ./internal/api`
  - **Done when:** join is fixed and test passes

- [x] **UR2-6.2 Validate KOPDS path IDs** (Commit: 1f59d8f)
  - **Repos:** kopds
  - **Read:** none
  - **Edit:** kopds/internal/api/handlers.go, kopds/internal/api/handlers_test.go
  - **Instructions:** Add `parsePositiveID(value string) (int64, error)`. Use it in route handlers (AuthorBooks, SeriesBooks, TagBooks, BookDetail, Cover, BookFile) and return `400 Bad Request` if invalid. Add tests.
  - **Verify:** `cd kopds && GOCACHE=/tmp/kopds-gocache go test ./internal/api`
  - **Done when:** IDs are validated and tests pass

- [x] **UR2-6.3 Check row iteration errors** (Commit: 07d1a8a)
  - **Repos:** kopds
  - **Read:** none
  - **Edit:** kopds/internal/database/book_repository.go
  - **Instructions:** In `Search` and `listBooks`, add `if err := rows.Err(); err != nil` checks after iteration. Use `defer rows.Close()` after successful `QueryContext`.
  - **Verify:** `cd kopds && GOCACHE=/tmp/kopds-gocache go test ./internal/database`
  - **Done when:** row errors are checked and tests pass

- [x] **UR2-6.4 Remove duplicate KOPDS reindex helper** (Commit: 1f08e32)
  - **Repos:** kopds
  - **Read:** none
  - **Edit:** kopds/internal/database/book_repository.go
  - **Instructions:** Consolidate `(*sqliteBookRepository).ReindexBook` and package `ReindexBook` into a single SQL execution. Delete unused version if applicable.
  - **Verify:** `cd kopds && GOCACHE=/tmp/kopds-gocache go test ./internal/database ./internal/scanner`
  - **Done when:** duplicate is removed and tests pass

- [x] **UR2-6.5 Escape Calibre SQLite read-only DSN** (Commit: eb6f06f)
  - **Repos:** kopds
  - **Read:** none
  - **Edit:** kopds/internal/scanner/calibre_reader.go, kopds/internal/scanner/calibre_reader_test.go
  - **Instructions:** Replace string concatenation `fmt.Sprintf("file:%s?mode=ro", path)` with a safe SQLite URI format handling spaces and special characters. Add tests.
  - **Verify:** `cd kopds && GOCACHE=/tmp/kopds-gocache go test ./internal/scanner`
  - **Done when:** DSN is escaped safely and tests pass

- [x] **UR2-6.6 Batch hydrate listed books** (Commit: 4608d70)
  - **Repos:** kopds
  - **Read:** none
  - **Edit:** kopds/internal/database/book_repository.go, kopds/internal/database/book_repository_test.go
  - **Instructions:** Refactor `listBooks` and `Search` to fetch relations (authors, tags, etc.) in batches rather than calling `GetByID` in a loop. Retain ID order. Add tests ensuring order.
  - **Verify:** `cd kopds && GOCACHE=/tmp/kopds-gocache go test ./internal/database ./internal/api`
  - **Done when:** batch hydration implemented and tests pass

Acceptance criteria for Phase 6:
- Author pages return the right books.
- Invalid numeric path values return `400`.
- Search/list code checks row iteration errors.
- KOPDS uses fewer repeated queries for paginated book lists.

## Phase 7: KOSYNC Correctness And Performance

Goal: make KOSYNC sync behavior more observable and avoid unnecessary maintenance work.

- [x] **UR2-7.1 Report stale progress updates** (Commit: df6fdac)
  - **Repos:** kosync
  - **Read:** none
  - **Edit:** kosync/internal/database/sqlite.go, kosync/internal/api/handlers.go, kosync/internal/database/sqlite_test.go, kosync/internal/api/handlers_test.go
  - **Instructions:** Change `UpsertProgress` to return `(bool, error)` to indicate if row changed. In handlers, if unchanged (stale or equal timestamp), return 200 OK but log that it was ignored. Add tests.
  - **Verify:** `cd kosync && GOCACHE=/tmp/kosync-gocache go test ./internal/database ./internal/api`
  - **Done when:** tests pass; stale updates handled

- [x] **UR2-7.2 Skip storage cap checks when progress did not change** (Commit: 7b9d8e5)
  - **Repos:** kosync
  - **Read:** none
  - **Edit:** kosync/internal/api/handlers.go, kosync/internal/api/handlers_test.go
  - **Instructions:** Use the changed boolean from `UpsertProgress`. Call `EnforceStorageCap` only if progress actually changed. Add tests.
  - **Verify:** `cd kosync && GOCACHE=/tmp/kosync-gocache go test ./internal/api`
  - **Done when:** storage check is conditional and tests pass

- [x] **UR2-7.3 Add progress timestamp index** (Commit: c81be0f)
  - **Repos:** kosync
  - **Read:** none
  - **Edit:** kosync/internal/database/sqlite.go, kosync/internal/database/sqlite_test.go
  - **Instructions:** Add `CREATE INDEX IF NOT EXISTS idx_progress_timestamp ON progress(timestamp);` to `Migrate`. Add test checking index presence.
  - **Verify:** `cd kosync && GOCACHE=/tmp/kosync-gocache go test ./internal/database`
  - **Done when:** index created and tests pass

Acceptance criteria for Phase 7:
- Stale progress updates are visible in logs and do not pretend to update storage.
- Storage cap work only runs when progress changes.
- Progress pruning has an index to support timestamp ordering.

## Phase 8: Module, Docs, Deployment, And Integration

Goal: make the public project shape and documentation match the final behavior.

- [x] **UR2-8.1 Change KOSYNC module path** (Commit: 65a2530)
  - **Repos:** kosync
  - **Read:** none
  - **Edit:** kosync/go.mod, kosync/**/*.go
  - **Instructions:** Change module name to `github.com/nlafevers/kosync`. Update all internal imports.
  - **Verify:** `cd kosync && GOCACHE=/tmp/kosync-gocache go test ./...`
  - **Done when:** module renamed and tests pass

- [x] **UR2-8.2 Update app documentation** (Commits: kopds:efac3fb, kosync:2fd5718)
  - **Repos:** kopds, kosync
  - **Read:** none
  - **Edit:** kopds/README.md, kosync/README.md, kosync/config/config.yaml, kosync/deploy/docker-compose.yml
  - **Instructions:** Update Go versions, rate-limit configs, KOSYNC registration defaults, and `KOSYNC_DATABASE_PATH` env vars. Commit separately.
  - **Verify:** visual inspection
  - **Done when:** docs are up to date

- [x] **UR2-8.3 Update uniformity inventories** (Commits: kopds:873ef8f, kosync:ad7e18d)
  - **Repos:** kopds, kosync
  - **Read:** none
  - **Edit:** kopds/UNIFORMITY.md, kosync/UNIFORMITY.md
  - **Instructions:** Add newly uniform items: CLI storage helpers, SQLite wrappers, UpdatePassword, request IDs, rate-limits, cap checks. Keep files identical.
  - **Verify:** `diff kopds/UNIFORMITY.md kosync/UNIFORMITY.md`
  - **Done when:** inventories updated and identical

- [x] **UR2-8.4 Update root uniformity plan** (Commit: 7d021d7)
  - **Repos:** koserver
  - **Read:** none
  - **Edit:** uniformity-plan.md
  - **Instructions:** Keep previous completed round marked. Add pointer to this roadmap.
  - **Verify:** visual inspection
  - **Done when:** plan updated

- [x] **UR2-8.5 Strengthen integration tests** (Commits: kopds:e50ac3c, kosync:d17ac49)
  - **Repos:** kopds, kosync
  - **Read:** none
  - **Edit:** kopds/test/integration_test.sh, kosync/test/integration_test.sh
  - **Instructions:** Assert HTTP status codes. KOSYNC asserts KOReader content type. KOPDS asserts auth success/failures. Scripts should be fully automated.
  - **Verify:** `cd kopds && ./test/integration_test.sh && cd ../kosync && ./test/integration_test.sh`
  - **Done when:** scripts run and pass

Acceptance criteria for Phase 8:
- KOSYNC has a canonical GitHub module path.
- Docs match the new default security posture.
- Uniformity inventories are current.
- Integration tests check status codes and MIME behavior.

## Phase 9: Final Verification Checklist

Run only after all phases are done.

- [x] **FINAL-9.1 Final KOPDS verification** (Commit: none — verification only)
  - **Repos:** kopds
  - **Read:** none
  - **Edit:** none
  - **Instructions:** Run full test suite, vet, formatting, and integration checks on KOPDS.
  - **Verify:** `cd kopds && gofmt -l . && GOCACHE=/tmp/kopds-gocache go vet ./... && GOCACHE=/tmp/kopds-gocache go test ./... && GOCACHE=/tmp/kopds-gocache go run golang.org/x/vuln/cmd/govulncheck@latest ./... && ./test/integration_test.sh`
  - **Done when:** all commands pass

- [x] **FINAL-9.2 Final KOSYNC verification** (Commit: none — verification only)
  - **Repos:** kosync
  - **Read:** none
  - **Edit:** none
  - **Instructions:** Run full test suite, vet, formatting, and integration checks on KOSYNC.
  - **Verify:** `cd kosync && gofmt -l . && GOCACHE=/tmp/kosync-gocache go vet ./... && GOCACHE=/tmp/kosync-gocache go test ./... && GOCACHE=/tmp/kosync-gocache go run golang.org/x/vuln/cmd/govulncheck@latest ./... && ./test/integration_test.sh`
  - **Done when:** all commands pass

- [x] **FINAL-9.3 Final root documentation verification** (Commit: none — verification only)
  - **Repos:** koserver
  - **Read:** none
  - **Edit:** none
  - **Instructions:** Check git status in the workspace root.
  - **Verify:** `cd .. && git status --short`
  - **Done when:** only intentional root docs are changed

## Notes For Future Implementers

- Copy-paste similarity is preferred for equivalent behavior. If a helper does the same job in both apps, use the same name, same argument order, same local variable names, and same control flow wherever possible.
- Do not force domain-specific code into artificial sameness. KOPDS owns OPDS catalog, Calibre scanning, images, and book files. KOSYNC owns KOReader sync protocol, progress rows, registration, and header auth.
- Keep compatibility aliases only when they protect current users. Prefer the uniform new names in docs and examples.
- When a test fails, fix the cause before moving to the next checkbox. Do not stack several unrelated changes into one commit.
