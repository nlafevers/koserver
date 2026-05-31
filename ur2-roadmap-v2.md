# Uniformity Round 2 Implementation Roadmap

This roadmap records the next round of work for bringing KOPDS and KOSYNC closer together while also fixing security, edge-case, test, bloat, and performance issues found during the fresh audit.

KOSERVER is documentation only. KOPDS and KOSYNC are separate Git repositories inside this workspace. Code changes inside `kopds/` must be committed in the KOPDS repository. Code changes inside `kosync/` must be committed in the KOSYNC repository. Changes to this roadmap or other root documentation must be committed in the KOSERVER repository.

## How To Use This Roadmap

When running Go commands in this workspace, prefer a writable cache under `/tmp`:

```bash
GOCACHE=/tmp/kopds-gocache go test ./...
GOCACHE=/tmp/kosync-gocache go test ./...
```

Do not copy commits across repositories. If you edit a KOPDS file, commit from `/home/nathan/koserver/kopds`. If you edit a KOSYNC file, commit from `/home/nathan/koserver/kosync`. If you edit this roadmap, commit from `/home/nathan/koserver`.

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

- [x] **UR2-1.1** Format KOPDS (Commit: 235c79a)
  - **Repos:** kopds
  - **Read:** kopds/
  - **Edit:** kopds/
  - **Verify:** `cd kopds && gofmt -l .` (should be empty)
  - **Done when:** `gofmt -w .` and `GOCACHE=/tmp/kopds-gocache go test ./...` passed.

- [x] **UR2-1.2** Format KOSYNC (Commit: e837d82)
  - **Repos:** kosync
  - **Read:** kosync/
  - **Edit:** kosync/
  - **Verify:** `cd kosync && gofmt -l .` (should be empty)
  - **Done when:** `gofmt -w .` and `GOCACHE=/tmp/kosync-gocache go test ./...` passed.

- [x] **UR2-1.3** Tidy KOPDS dependencies (Commit: 28c5561)
  - **Repos:** kopds
  - **Read:** kopds/go.mod
  - **Edit:** kopds/go.mod, kopds/go.sum
  - **Verify:** `grep chi kopds/go.mod` returns nothing
  - **Done when:** `go mod tidy` removed `chi` and `go test ./...` passed.

- [x] **UR2-1.4** Confirm KOSYNC dependencies are tidy (Commit: none)
  - **Repos:** kosync
  - **Read:** kosync/go.mod
  - **Edit:** kosync/go.mod, kosync/go.sum
  - **Verify:** `cd kosync && go mod tidy && git diff --exit-code go.mod go.sum`
  - **Done when:** `go mod tidy` causes no changes or tests pass after changes.

Acceptance criteria for Phase 1:

- `gofmt -l .` prints nothing in both app repos.
- `go test ./...` passes in both app repos.
- KOPDS no longer depends on `github.com/go-chi/chi/v5`.

## Phase 2: Toolchain And Release Security

Goal: fix the Go standard library vulnerability and make release automation match both projects.

- [x] **UR2-2.1** Update Go versions in KOPDS (Commit: 3a2d10f)
  - **Repos:** kopds
  - **Read:** kopds/.github/workflows/publish-binaries.yml
  - **Edit:** kopds/.github/workflows/publish-binaries.yml
  - **Verify:** `grep "go-version: '1.26.x'" kopds/.github/workflows/publish-binaries.yml`
  - **Done when:** workflow updated to Go 1.26.x and tests pass.

- [x] **UR2-2.2** Update Go versions in KOSYNC (Commit: 9b1b210)
  - **Repos:** kosync
  - **Read:** kosync/.github/workflows/publish-binaries.yml
  - **Edit:** kosync/.github/workflows/publish-binaries.yml
  - **Verify:** `grep "go-version: '1.26.x'" kosync/.github/workflows/publish-binaries.yml`
  - **Done when:** workflow updated to Go 1.26.x and tests pass.

- [x] **UR2-2.3** Fix Docker publish workflows (Commit kopds: eaa84e4, commit kosync: c5e7764)
  - **Repos:** kopds, kosync
  - **Read:** kopds/.github/workflows/docker-publish.yml, kosync/.github/workflows/docker-publish.yml
  - **Edit:** kopds/.github/workflows/docker-publish.yml, kosync/.github/workflows/docker-publish.yml
  - **Verify:** `grep "file: build/Dockerfile" kopds/.github/workflows/docker-publish.yml kosync/.github/workflows/docker-publish.yml`
  - **Done when:** both workflows explicitly point to `build/Dockerfile`.

- [x] **UR2-2.4** Fix KOSYNC binary release workflow (Commit: 9a6a7ad)
  - **Repos:** kosync
  - **Read:** kosync/.github/workflows/publish-binaries.yml
  - **Edit:** kosync/.github/workflows/publish-binaries.yml
  - **Verify:** `grep BINARY_NAME kosync/.github/workflows/publish-binaries.yml` shows `kosync`
  - **Done when:** workflow builds `kosync` from `./cmd/kosync`.

Acceptance criteria for Phase 2:

- Both Dockerfiles use a fixed, non-vulnerable Go patch version (superseded).
- Both binary release workflows use the correct app name and package path.

## Phase 3: Uniform CLI And Database Lifecycle

Goal: make the command-line user management code as identical as practical.

- [x] **UR2-3.1** Storage lifecycle wrappers (Commit: none)
  - **Repos:** kopds, kosync
  - **Read:** kopds/internal/database/sqlite.go, kosync/internal/database/sqlite.go
  - **Edit:** kopds/internal/database/sqlite.go, kosync/internal/database/sqlite.go, kosync/cmd/kosync/main.go
  - **Verify:** both apps use `NewSQLite(path, allowCreate)` and `Migrate(db)`
  - **Done when:** KOSYNC `InitDB` removed; both apps share the same open/migrate/inject flow (Superseded by TDL-001).

- [ ] **UR2-3.2** Add KOPDS storage user methods
  - **Repos:** kopds
  - **Read:** kopds/internal/database/sqlite.go, kopds/internal/database/user_repository.go
  - **Edit:** kopds/internal/database/sqlite.go, kopds/internal/database/user_repository.go
  - **Verify:** `GOCACHE=/tmp/kopds-gocache go test ./internal/database ./cmd/kopds`
  - **Done when:** `CreateUserIfNotExists`, `SaveUser`, `GetUserHash`, `UpdatePassword`, and `DeleteUser` added to KOPDS storage.

- [ ] **UR2-3.3** Rename or wrap KOSYNC password update
  - **Repos:** kosync
  - **Read:** kosync/internal/database/sqlite.go
  - **Edit:** kosync/internal/database/sqlite.go, kosync/cmd/kosync/main.go
  - **Verify:** `GOCACHE=/tmp/kosync-gocache go test ./internal/database ./cmd/kosync`
  - **Done when:** KOSYNC has `UpdatePassword(username, passwordHash)` called by CLI.

- [ ] **UR2-3.4** Make CLI functions copy-paste similar
  - **Repos:** kopds, kosync
  - **Read:** kopds/cmd/kopds/main.go, kosync/cmd/kosync/main.go
  - **Edit:** kopds/cmd/kopds/main.go, kosync/cmd/kosync/main.go
  - **Verify:** `GOCACHE=/tmp/kopds-gocache go test ./cmd/kopds`, `GOCACHE=/tmp/kosync-gocache go test ./cmd/kosync`
  - **Done when:** CLI functions like `runCLI`, `createUser`, `deleteUser` are identical except for app-specific types.

Acceptance criteria for Phase 3:

- CLI user-management functions are copy-paste similar across KOPDS and KOSYNC.
- Both apps still create missing databases from CLI commands.
- Duplicate `create-user` still fails from the CLI in both apps.

## Phase 4: Uniform Server Lifecycle And Shared Middleware

Goal: make server startup, shutdown, logging, and shared middleware behavior easier to compare.

- [ ] **UR2-4.1** Extract KOSYNC `runServer`
  - **Repos:** kosync
  - **Read:** kosync/cmd/kosync/main.go
  - **Edit:** kosync/cmd/kosync/main.go
  - **Verify:** `GOCACHE=/tmp/kosync-gocache go test ./cmd/kosync ./internal/api`
  - **Done when:** `main()` calls `runServer(cfg, log)`; shutdown uses timeout context.

- [ ] **UR2-4.2** Add KOSYNC config validation
  - **Repos:** kosync
  - **Read:** kosync/internal/config/config.go
  - **Edit:** kosync/internal/config/config.go, kosync/cmd/kosync/main.go
  - **Verify:** `GOCACHE=/tmp/kosync-gocache go test ./internal/config ./cmd/kosync`
  - **Done when:** `cfg.Validate()` checks Port, DatabasePath, LogLevel, StorageCapMB.

- [ ] **UR2-4.3** Standardize request ID generation
  - **Repos:** kopds, kosync
  - **Read:** kopds/internal/api/middleware.go, kosync/internal/api/middleware.go
  - **Edit:** kopds/internal/api/middleware.go, kosync/internal/api/middleware.go
  - **Verify:** `GOCACHE=/tmp/kopds-gocache go test ./internal/api`, `GOCACHE=/tmp/kosync-gocache go test ./internal/api`
  - **Done when:** both use `rand.Read` (16 bytes) with hex encoding and timestamp fallback.

- [ ] **UR2-4.4** Enable SQLite foreign keys uniformly
  - **Repos:** kopds, kosync
  - **Read:** kopds/internal/database/sqlite.go, kosync/internal/database/sqlite.go
  - **Edit:** kopds/internal/database/sqlite.go, kosync/internal/database/sqlite.go
  - **Verify:** tests prove foreign-key violations fail in both repos.
  - **Done when:** `PRAGMA foreign_keys=ON;` added to initialization.

- [ ] **UR2-4.5** Move disabled storage-cap check to the top
  - **Repos:** kopds, kosync
  - **Read:** kopds/internal/database/sqlite.go, kosync/internal/database/sqlite.go
  - **Edit:** kopds/internal/database/sqlite.go, kosync/internal/database/sqlite.go
  - **Verify:** tests prove disabled caps do not inspect missing files.
  - **Done when:** `if capMB <= 0 { return false, nil }` is the first check in `EnforceStorageCap`.

Acceptance criteria for Phase 4:

- KOSYNC has the same `main` to `runServer` shape as KOPDS.
- Middleware helper names remain aligned.
- Storage-cap disabled behavior is identical and cheap.
- SQLite foreign keys are active in both apps.

## Phase 5: Security Behavior

Goal: reduce public attack surface and make KOSYNC protocol responses consistent.

- [ ] **UR2-5.1** Disable KOSYNC public registration by default
  - **Repos:** kosync
  - **Read:** kosync/internal/config/config.go, kosync/config/config.yaml, kosync/deploy/docker-compose.yml
  - **Edit:** kosync/internal/config/config.go, kosync/config/config.yaml, kosync/deploy/docker-compose.yml
  - **Verify:** `GOCACHE=/tmp/kosync-gocache go test ./internal/config ./internal/api`
  - **Done when:** registration is disabled by default in code, config, and compose.

- [ ] **UR2-5.2** Wire KOSYNC rate limiting
  - **Repos:** kosync
  - **Read:** kosync/internal/config/config.go, kosync/internal/api/middleware.go, kosync/cmd/kosync/main.go
  - **Edit:** kosync/internal/config/config.go, kosync/internal/api/middleware.go, kosync/cmd/kosync/main.go
  - **Verify:** tests for 429 status, IP-based limiting, and X-Forwarded-For trust.
  - **Done when:** rate limiting covers `/users/create` and protected auth routes.

- [ ] **UR2-5.3** Add equivalent KOPDS failed-auth rate limiting
  - **Repos:** kopds
  - **Read:** kopds/internal/config/config.go, kopds/internal/api/middleware.go
  - **Edit:** kopds/internal/config/config.go, kopds/internal/api/middleware.go
  - **Verify:** tests for failed Basic Auth attempts rate limiting.
  - **Done when:** KOPDS has comparable failed-auth protection.

- [ ] **UR2-5.4** Fix KOSYNC response content type
  - **Repos:** kosync
  - **Read:** kosync/internal/api/handlers.go, kosync/cmd/kosync/main.go
  - **Edit:** kosync/internal/api/handlers.go, kosync/cmd/kosync/main.go
  - **Verify:** `/users/auth` and `/users/create` return `application/vnd.koreader.v1+json`.
  - **Done when:** `ContentTypeMiddleware` wraps all relevant routes and `HandleAuth` doesn't override it.

- [ ] **UR2-5.5** Make KOSYNC registration timing less distinguishable
  - **Repos:** kosync
  - **Read:** kosync/internal/api/handlers.go
  - **Edit:** kosync/internal/api/handlers.go
  - **Verify:** tests confirm 201 for duplicate registration and unchanged password.
  - **Done when:** `randomDelay` used for both new and duplicate registrations.

Acceptance criteria for Phase 5:

- KOSYNC public registration is disabled unless explicitly enabled.
- KOSYNC rate limiting is wired into real routes.
- KOPDS has comparable failed-auth protection.
- KOSYNC response content type matches the KOReader protocol.

## Phase 6: KOPDS Correctness And Performance

Goal: fix confirmed KOPDS bugs and reduce unnecessary query work.

- [ ] **UR2-6.1** Fix KOPDS author listing query
  - **Repos:** kopds
  - **Read:** kopds/internal/database/book_repository.go
  - **Edit:** kopds/internal/database/book_repository.go
  - **Verify:** regression test with books/authors where IDs != book IDs.
  - **Done when:** join uses `bal.book_id` instead of `bal.author_id`.

- [ ] **UR2-6.2** Validate KOPDS path IDs
  - **Repos:** kopds
  - **Read:** kopds/internal/api/handlers.go
  - **Edit:** kopds/internal/api/handlers.go
  - **Verify:** `GOCACHE=/tmp/kopds-gocache go test ./internal/api`
  - **Done when:** `parsePositiveID` returns 400 for invalid/non-positive numeric path IDs.

- [ ] **UR2-6.3** Check row iteration errors
  - **Repos:** kopds
  - **Read:** kopds/internal/database/book_repository.go
  - **Edit:** kopds/internal/database/book_repository.go
  - **Verify:** `GOCACHE=/tmp/kopds-gocache go test ./internal/database`
  - **Done when:** `Search` and `listBooks` check `rows.Err()` after iteration.

- [ ] **UR2-6.4** Remove duplicate KOPDS reindex helper
  - **Repos:** kopds
  - **Read:** kopds/internal/database/book_repository.go
  - **Edit:** kopds/internal/database/book_repository.go
  - **Verify:** `GOCACHE=/tmp/kopds-gocache go test ./internal/database ./internal/scanner`
  - **Done when:** `ReindexBook` logic consolidated to a single helper.

- [ ] **UR2-6.5** Escape Calibre SQLite read-only DSN
  - **Repos:** kopds
  - **Read:** kopds/internal/scanner/calibre_reader.go
  - **Edit:** kopds/internal/scanner/calibre_reader.go
  - **Verify:** tests with spaces and special characters in DB path pass.
  - **Done when:** DSN built using safe SQLite URI construction.

- [ ] **UR2-6.6** Batch hydrate listed books
  - **Repos:** kopds
  - **Read:** kopds/internal/database/book_repository.go
  - **Edit:** kopds/internal/database/book_repository.go
  - **Verify:** `GOCACHE=/tmp/kopds-gocache go test ./internal/database ./internal/api`
  - **Done when:** book lists use batch queries for authors, tags, formats, series.

Acceptance criteria for Phase 6:

- Author pages return the right books.
- Invalid numeric path values return `400`.
- Search/list code checks row iteration errors.
- KOPDS uses fewer repeated queries for paginated book lists.

## Phase 7: KOSYNC Correctness And Performance

Goal: make KOSYNC sync behavior more observable and avoid unnecessary maintenance work.

- [ ] **UR2-7.1** Report stale progress updates
  - **Repos:** kosync
  - **Read:** kosync/internal/database/sqlite.go, kosync/internal/api/handlers.go
  - **Edit:** kosync/internal/database/sqlite.go, kosync/internal/api/handlers.go
  - **Verify:** `GOCACHE=/tmp/kosync-gocache go test ./internal/database ./internal/api`
  - **Done when:** stale updates (older/equal timestamp) logged as ignored.

- [ ] **UR2-7.2** Skip storage cap checks when progress did not change
  - **Repos:** kosync
  - **Read:** kosync/internal/api/handlers.go
  - **Edit:** kosync/internal/api/handlers.go
  - **Verify:** tests proving stale updates do not trigger storage-cap enforcement.
  - **Done when:** `EnforceStorageCap` only called if `UpsertProgress` returned changed.

- [ ] **UR2-7.3** Add progress timestamp index
  - **Repos:** kosync
  - **Read:** kosync/internal/database/sqlite.go
  - **Edit:** kosync/internal/database/sqlite.go
  - **Verify:** `sqlite_master` query confirms `idx_progress_timestamp` exists.
  - **Done when:** index created in `Migrate`.

Acceptance criteria for Phase 7:

- Stale progress updates are visible in logs and do not pretend to update storage.
- Storage cap work only runs when progress changes.
- Progress pruning has an index to support timestamp ordering.

## Phase 8: Module, Docs, Deployment, And Integration

Goal: make the public project shape and documentation match the final behavior.

- [ ] **UR2-8.1** Change KOSYNC module path
  - **Repos:** kosync
  - **Read:** kosync/go.mod
  - **Edit:** kosync/go.mod, (all .go files in kosync/)
  - **Verify:** `GOCACHE=/tmp/kosync-gocache go test ./...`
  - **Done when:** module and all internal imports use `github.com/nlafevers/kosync`.

- [ ] **UR2-8.2** Update app documentation
  - **Repos:** kopds, kosync
  - **Read:** kopds/README.md, kosync/README.md, kosync/config/config.yaml, kosync/deploy/docker-compose.yml
  - **Edit:** kopds/README.md, kosync/README.md, kosync/config/config.yaml, kosync/deploy/docker-compose.yml
  - **Verify:** (visual) docs reflect Go version, rate-limit, and KOSYNC registration changes.
  - **Done when:** both READMEs are current with Phase 1-7 changes.

- [ ] **UR2-8.3** Update uniformity inventories
  - **Repos:** kopds, kosync
  - **Read:** kopds/UNIFORMITY.md, kosync/UNIFORMITY.md
  - **Edit:** kopds/UNIFORMITY.md, kosync/UNIFORMITY.md
  - **Verify:** (visual) new identical items listed; audit notes for intentional differences current.
  - **Done when:** inventory files match the final project state.

- [ ] **UR2-8.4** Update root uniformity plan
  - **Repos:** koserver
  - **Read:** uniformity-plan.md
  - **Edit:** uniformity-plan.md
  - **Verify:** (visual) pointer to this roadmap added.
  - **Done when:** root plan updated.

- [ ] **UR2-8.5** Strengthen integration tests
  - **Repos:** kopds, kosync
  - **Read:** kopds/test/integration_test.sh, kosync/test/integration_test.sh
  - **Edit:** kopds/test/integration_test.sh, kosync/test/integration_test.sh
  - **Verify:** `./test/integration_test.sh` passes in both repos.
  - **Done when:** tests assert HTTP status codes and KOSYNC asserts KOReader content type.

Acceptance criteria for Phase 8:

- KOSYNC has a canonical GitHub module path.
- Docs match the new default security posture.
- Uniformity inventories are current.
- Integration tests check status codes and MIME behavior.

## Phase 9: Final Verification Checklist

Run this only after all phases are done.

- [ ] **UR2-9.1** Final KOPDS verification
  - **Repos:** kopds
  - **Verify:** `gofmt -l .`, `go vet ./...`, `go test ./...`, `govulncheck ./...`, `./test/integration_test.sh`
  - **Done when:** all commands pass.

- [ ] **UR2-9.2** Final KOSYNC verification
  - **Repos:** kosync
  - **Verify:** `gofmt -l .`, `go vet ./...`, `go test ./...`, `govulncheck ./...`, `./test/integration_test.sh`
  - **Done when:** all commands pass.

- [ ] **UR2-9.3** Final root documentation verification
  - **Repos:** koserver
  - **Verify:** `git status --short` (only root docs changed)
  - **Done when:** root docs are verified.

## Notes For Future Implementers

- Copy-paste similarity is preferred for equivalent behavior. If a helper does the same job in both apps, use the same name, same argument order, same local variable names, and same control flow wherever possible.
- Do not force domain-specific code into artificial sameness. KOPDS owns OPDS catalog, Calibre scanning, images, and book files. KOSYNC owns KOReader sync protocol, progress rows, registration, and header auth.
- Keep compatibility aliases only when they protect current users. Prefer the uniform new names in docs and examples.
- When a test fails, fix the cause before moving to the next checkbox. Do not stack several unrelated changes into one commit.
