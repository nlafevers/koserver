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

- [x] **UR2-1.1 Format KOPDS** (Commit: 235c79a)
  1. Open a terminal in `/home/nathan/koserver/kopds`.
  2. Run:
     ```bash
     gofmt -w .
     ```
  3. Run:
     ```bash
     gofmt -l .
     ```
  4. Confirm the second command prints nothing.
  5. Run:
     ```bash
     GOCACHE=/tmp/kopds-gocache go test ./...
     ```

- [x] **UR2-1.2 Format KOSYNC** (Commit: e837d82)
  1. Open a terminal in `/home/nathan/koserver/kosync`.
  2. Run:
     ```bash
     gofmt -w .
     ```
  3. Run:
     ```bash
     gofmt -l .
     ```
  4. Confirm the second command prints nothing.
  5. Run:
     ```bash
     GOCACHE=/tmp/kosync-gocache go test ./...
     ```

- [x] **UR2-1.3 Tidy KOPDS dependencies** (Commit: 28c5561)
  1. Open `/home/nathan/koserver/kopds/go.mod`.
  2. Confirm `github.com/go-chi/chi/v5` is still listed.
  3. Run:
     ```bash
     GOCACHE=/tmp/kopds-gocache go mod tidy
     ```
  4. Confirm `github.com/go-chi/chi/v5` is removed from `go.mod` and `go.sum`.
  5. Run:
     ```bash
     GOCACHE=/tmp/kopds-gocache go test ./...
     ```

- [x] **UR2-1.4 Confirm KOSYNC dependencies are tidy** (Commit: none)
  1. Open a terminal in `/home/nathan/koserver/kosync`.
  2. Run:
     ```bash
     GOCACHE=/tmp/kosync-gocache go mod tidy
     ```
  3. Run:
     ```bash
     git diff -- go.mod go.sum
     ```
  4. If there are changes, run tests:
     ```bash
     GOCACHE=/tmp/kosync-gocache go test ./...
     ```

Acceptance criteria for Phase 1:

- `gofmt -l .` prints nothing in both app repos.
- `go test ./...` passes in both app repos.
- KOPDS no longer depends on `github.com/go-chi/chi/v5`.

## Phase 2: Toolchain And Release Security

Goal: fix the Go standard library vulnerability and make release automation match both projects.

- [x] **UR2-2.1 Update Go versions in KOPDS** (Commit: 3a2d10f)
  1. Open `/home/nathan/koserver/kopds/go.mod`.
  2. Keep the `go` directive at the project-required version unless the project owner decides to raise it. The immediate vulnerability fix is in the build toolchain, not necessarily the module directive.
  3. Open `/home/nathan/koserver/kopds/.github/workflows/publish-binaries.yml`.
  4. Change `go-version: '1.25.0'` to `go-version: '1.26.x'`.
  5. ~~Open `/home/nathan/koserver/kopds/build/Dockerfile`.~~
  6. ~~Change the builder image from `golang:1.26-alpine` to a fixed patch tag such as `golang:1.26.3-alpine`.~~
  7. Run:
     ```bash
     GOCACHE=/tmp/kopds-gocache go test ./...
     ```
  8. ~~Run:~~
     ```bash
     GOCACHE=/tmp/kopds-gocache go run golang.org/x/vuln/cmd/govulncheck@latest ./...
     ```
  9. ~~If `govulncheck` still reports `GO-2026-4971`, confirm the command is using Go `1.26.3` or newer by running `go version`.~~

- [x] **UR2-2.2 Update Go versions in KOSYNC** (Commit: 9b1b210)
  1. Repeat the same toolchain update in `/home/nathan/koserver/kosync`.
  2. Update `.github/workflows/publish-binaries.yml`.
  3. ~~Update `build/Dockerfile`.~~
  4. Run:
     ```bash
     GOCACHE=/tmp/kosync-gocache go test ./...
     GOCACHE=/tmp/kosync-gocache go run golang.org/x/vuln/cmd/govulncheck@latest ./...
     ```

- [x] **UR2-2.3 Fix Docker publish workflows** (Commit kopds: eaa84e4, commit kosync: c5e7764)
  1. Open `/home/nathan/koserver/kopds/.github/workflows/docker-publish.yml`.
  2. Under the `docker/build-push-action` `with:` block, add:
     ```yaml
     file: build/Dockerfile
     ```
  3. Repeat the same change in `/home/nathan/koserver/kosync/.github/workflows/docker-publish.yml`.
  4. Commit each repository separately.

- [x] **UR2-2.4 Fix KOSYNC binary release workflow** (Commit: 9a6a7ad)
  1. Open `/home/nathan/koserver/kosync/.github/workflows/publish-binaries.yml`.
  2. Find the line that sets `BINARY_NAME="kopds-${{ matrix.goos }}-${{ matrix.goarch }}"`.
  3. Change `kopds` to `kosync`.
  4. Find the `go build` line.
  5. Change `./cmd/kopds` to `./cmd/kosync`.
  6. Run KOSYNC tests:
     ```bash
     GOCACHE=/tmp/kosync-gocache go test ./...
     ```

Acceptance criteria for Phase 2:

- ~~Both Dockerfiles use a fixed, non-vulnerable Go patch version.~~
- Both binary release workflows use the correct app name and package path.
- ~~`govulncheck` no longer reports called vulnerabilities.~~

## Phase 3: Uniform CLI And Database Lifecycle

Goal: make the command-line user management code as identical as practical.

- [ ] **3.1 Add KOPDS storage lifecycle wrappers**
  1. Open `/home/nathan/koserver/kopds/internal/database/sqlite.go`.
  2. Add a function named `InitDB(path string, allowCreate bool) (*Storage, error)` that mirrors KOSYNC:
     - Call `OpenSQLite(path, allowCreate)`.
     - If opening fails, return the error.
     - Run `Migrate(db)`.
     - If migration fails, close the database and return the error.
     - Return `&Storage{db: db, log: slog.Default()}`.
  3. Add `func (s *Storage) Close() error { return s.db.Close() }`.
  4. Keep existing `NewSQLite(path)` for compatibility.
  5. Run KOPDS tests:
     ```bash
     GOCACHE=/tmp/kopds-gocache go test ./internal/database ./cmd/kopds
     ```
  6. Commit in KOPDS:
     ```bash
     git add internal/database/sqlite.go
     git commit -m "Add uniform database lifecycle wrapper"
     ```

- [ ] **3.2 Add KOPDS storage user methods**
  1. Open `/home/nathan/koserver/kopds/internal/database/sqlite.go` and `/home/nathan/koserver/kopds/internal/database/user_repository.go`.
  2. Add storage methods with these exact names where possible:
     - `CreateUserIfNotExists(username, password string) error`
     - `SaveUser(username, password string) error`
     - `GetUserHash(username string) (string, error)`
     - `UpdatePassword(username, passwordHash string) error`
     - `DeleteUser(username string) error`
  3. Implement each method by using the same SQL behavior currently used by `sqliteUserRepository`.
  4. Keep the existing repository interface intact so current KOPDS callers continue to work.
  5. Add focused tests in `internal/database/user_repository_test.go` or a new storage test.
  6. Run:
     ```bash
     GOCACHE=/tmp/kopds-gocache go test ./internal/database ./cmd/kopds
     ```
  7. Commit in KOPDS:
     ```bash
     git add internal/database
     git commit -m "Add uniform storage user operations"
     ```

- [ ] **3.3 Rename or wrap KOSYNC password update**
  1. Open `/home/nathan/koserver/kosync/internal/database/sqlite.go`.
  2. Add a method named `UpdatePassword(username, passwordHash string) error`.
  3. Move the body of `UpdateUserPassword` into `UpdatePassword`, or have `UpdateUserPassword` call `UpdatePassword`.
  4. Update KOSYNC CLI code to call `UpdatePassword`.
  5. Keep `UpdateUserPassword` temporarily if tests or callers still use it.
  6. Run:
     ```bash
     GOCACHE=/tmp/kosync-gocache go test ./internal/database ./cmd/kosync
     ```
  7. Commit in KOSYNC:
     ```bash
     git add internal/database cmd/kosync
     git commit -m "Use uniform password update method"
     ```

- [ ] **3.4 Make CLI functions copy-paste similar**
  1. Open both entrypoints side by side:
     - `/home/nathan/koserver/kopds/cmd/kopds/main.go`
     - `/home/nathan/koserver/kosync/cmd/kosync/main.go`
  2. Compare these functions:
     - `runCLI`
     - `printUsage`
     - `createUser`
     - `deleteUser`
     - `changePassword`
     - `openCLIStorage`
     - `passwordFromArgs`
     - `readPasswordInteractively`
  3. Make the functions identical except for import paths, app-specific types, and `appName`.
  4. Prefer the cleaner KOPDS CLI dispatch style after KOSYNC has been formatted.
  5. Use the same stdout and error wording in both apps.
  6. Run:
     ```bash
     cd /home/nathan/koserver/kopds
     GOCACHE=/tmp/kopds-gocache go test ./cmd/kopds ./internal/database

     cd /home/nathan/koserver/kosync
     GOCACHE=/tmp/kosync-gocache go test ./cmd/kosync ./internal/database
     ```
  7. Commit in each app repo separately with:
     ```bash
     git commit -m "Standardize CLI user management"
     ```

Acceptance criteria for Phase 3:

- CLI user-management functions are copy-paste similar across KOPDS and KOSYNC.
- Both apps still create missing databases from CLI commands.
- Duplicate `create-user` still fails from the CLI in both apps.

## Phase 4: Uniform Server Lifecycle And Shared Middleware

Goal: make server startup, shutdown, logging, and shared middleware behavior easier to compare.

- [ ] **4.1 Extract KOSYNC `runServer`**
  1. Open `/home/nathan/koserver/kosync/cmd/kosync/main.go`.
  2. Change `main()` so it matches KOPDS:
     - Load config.
     - Create logger with `logger.New`.
     - If there are CLI args, call `runCLI(cfg)` and return.
     - Otherwise call `runServer(cfg, log)`.
  3. Move all server setup currently inside `main()` into `runServer(cfg *config.Config, log *slog.Logger)`.
  4. Make shutdown use a timeout context like KOPDS does.
  5. Keep KOSYNC-specific routes unchanged.
  6. Run:
     ```bash
     GOCACHE=/tmp/kosync-gocache go test ./cmd/kosync ./internal/api
     ```
  7. Commit in KOSYNC:
     ```bash
     git add cmd/kosync/main.go cmd/kosync/*_test.go
     git commit -m "Align server lifecycle structure"
     ```

- [ ] **4.2 Add KOSYNC config validation**
  1. Open `/home/nathan/koserver/kosync/internal/config/config.go`.
  2. Add `func (c *Config) Validate() error`.
  3. Validate:
     - `Port` is between `1` and `65535`, unless tests intentionally allow `0`.
     - `DatabasePath` is not empty.
     - `LogLevel` is one of `debug`, `info`, `warn`, or `error`.
     - `StorageCapMB` is not negative.
  4. Call `cfg.Validate()` in KOSYNC `runServer` before opening the database.
  5. Add config tests for valid defaults and invalid values.
  6. Run:
     ```bash
     GOCACHE=/tmp/kosync-gocache go test ./internal/config ./cmd/kosync
     ```
  7. Commit in KOSYNC:
     ```bash
     git add internal/config cmd/kosync
     git commit -m "Validate KOSYNC configuration"
     ```

- [ ] **4.3 Standardize request ID generation**
  1. Open both middleware files:
     - `kopds/internal/api/middleware.go`
     - `kosync/internal/api/middleware.go`
  2. Find `generateRequestID`.
  3. Change both functions to:
     - Allocate 16 bytes.
     - Call `rand.Read`.
     - If `rand.Read` fails, fall back to a timestamp-based value.
     - Return the hex string.
  4. Keep the function name identical.
  5. Add or update middleware tests to confirm `request_id` exists and is non-empty.
  6. Run API tests in both repos.
  7. Commit separately in each repo with:
     ```bash
     git commit -m "Standardize request ID generation"
     ```

- [ ] **4.4 Enable SQLite foreign keys uniformly**
  1. Open both `internal/database/sqlite.go` files.
  2. Find the `PRAGMA journal_mode=WAL; PRAGMA busy_timeout=5000;` statement.
  3. Add `PRAGMA foreign_keys=ON;` to the same initialization flow.
  4. Add a test in each app that proves a foreign-key violation fails.
  5. Run database tests in both repos.
  6. Commit separately with:
     ```bash
     git commit -m "Enable SQLite foreign key enforcement"
     ```

- [ ] **4.5 Move disabled storage-cap check to the top**
  1. Open both `internal/database/sqlite.go` files.
  2. In `Storage.EnforceStorageCap`, add this as the first logic after getting the logger:
     ```go
     if capMB <= 0 {
         return false, nil
     }
     ```
  3. Keep `enforceStorageCap` helper behavior identical.
  4. Add or update tests proving disabled caps do not inspect a missing file.
  5. Run database tests in both repos.
  6. Commit separately with:
     ```bash
     git commit -m "Skip storage cap work when disabled"
     ```

Acceptance criteria for Phase 4:

- KOSYNC has the same `main` to `runServer` shape as KOPDS.
- Middleware helper names remain aligned.
- Storage-cap disabled behavior is identical and cheap.
- SQLite foreign keys are active in both apps.

## Phase 5: Security Behavior

Goal: reduce public attack surface and make KOSYNC protocol responses consistent.

- [ ] **5.1 Disable KOSYNC public registration by default**
  1. Open `/home/nathan/koserver/kosync/internal/config/config.go`.
  2. Change the default for `disable_registration` from `false` to `true`.
  3. Open `/home/nathan/koserver/kosync/config/config.yaml`.
  4. Change `disable_registration: false` to `disable_registration: true`.
  5. Open `/home/nathan/koserver/kosync/deploy/docker-compose.yml`.
  6. Change `KOSYNC_DISABLE_REGISTRATION=false` to `KOSYNC_DISABLE_REGISTRATION=true`.
  7. Update README text so users know CLI user creation is the default and public registration is opt-in.
  8. Add tests proving the default config disables registration.
  9. Run:
     ```bash
     GOCACHE=/tmp/kosync-gocache go test ./internal/config ./internal/api
     ```
  10. Commit in KOSYNC:
      ```bash
      git add internal/config config deploy README.md internal/api
      git commit -m "Disable public registration by default"
      ```

- [ ] **5.2 Wire KOSYNC rate limiting**
  1. Open `/home/nathan/koserver/kosync/internal/config/config.go`.
  2. Add config fields:
     - `RateLimitEnabled bool`
     - `RateLimitPerMinute int`
     - `RateLimitBurst int`
     - `TrustProxyHeaders bool`
  3. Add defaults:
     - `rate_limit_enabled: true`
     - `rate_limit_per_minute: 30`
     - `rate_limit_burst: 10`
     - `trust_proxy_headers: false`
  4. Open `/home/nathan/koserver/kosync/internal/api/middleware.go`.
  5. Update rate limiting so the key is a clean client IP, not the full `host:port` value.
  6. If `trust_proxy_headers` is false, use `net.SplitHostPort(r.RemoteAddr)`.
  7. If `trust_proxy_headers` is true, prefer the first IP in `X-Forwarded-For`, then fall back to `RemoteAddr`.
  8. Open `/home/nathan/koserver/kosync/cmd/kosync/main.go`.
  9. Wrap `/users/create` and protected auth routes with rate limiting when enabled.
  10. Add tests for:
      - Requests over the limit return `429`.
      - Different IPs get different limiters.
      - `X-Forwarded-For` is ignored unless `trust_proxy_headers` is true.
  11. Run:
      ```bash
      GOCACHE=/tmp/kosync-gocache go test ./internal/api ./cmd/kosync
      ```
  12. Commit in KOSYNC:
      ```bash
      git add internal/config internal/api cmd/kosync config README.md deploy
      git commit -m "Enable KOSYNC request rate limiting"
      ```

- [ ] **5.3 Add equivalent KOPDS failed-auth rate limiting**
  1. Open `/home/nathan/koserver/kopds/internal/config/config.go`.
  2. Add the same rate-limit config fields and defaults as KOSYNC.
  3. Open `/home/nathan/koserver/kopds/internal/api/middleware.go`.
  4. Add the same client IP helper and limiter types if they now exist in KOSYNC.
  5. Apply rate limiting only around Basic Auth failures or auth attempts, not around every successful catalog request.
  6. Add tests for too many failed Basic Auth attempts.
  7. Run:
     ```bash
     GOCACHE=/tmp/kopds-gocache go test ./internal/api ./internal/config ./cmd/kopds
     ```
  8. Commit in KOPDS:
     ```bash
     git add internal/config internal/api cmd/kopds config README.md deploy
     git commit -m "Add Basic Auth rate limiting"
     ```

- [ ] **5.4 Fix KOSYNC response content type**
  1. Open `/home/nathan/koserver/kosync/internal/api/handlers.go`.
  2. In `HandleAuth`, remove `w.Header().Set("Content-Type", "application/json")`.
  3. Open `/home/nathan/koserver/kosync/cmd/kosync/main.go`.
  4. Make sure `ContentTypeMiddleware` wraps both protected routes and `/users/create`.
  5. Add tests for:
     - `/users/auth` returns `application/vnd.koreader.v1+json`.
     - `/users/create` returns `application/vnd.koreader.v1+json`.
     - Error responses from KOSYNC also use the KOReader MIME type where practical.
  6. Run:
     ```bash
     GOCACHE=/tmp/kosync-gocache go test ./internal/api ./cmd/kosync
     ```
  7. Commit in KOSYNC:
     ```bash
     git add internal/api cmd/kosync
     git commit -m "Use KOReader MIME type consistently"
     ```

- [ ] **5.5 Make KOSYNC registration timing less distinguishable**
  1. Open `/home/nathan/koserver/kosync/internal/api/handlers.go`.
  2. Find `randomDelay`.
  3. Use the same delay path for new registrations and duplicate registrations.
  4. Keep duplicate registration fake success behavior so attackers cannot easily enumerate users.
  5. Add a test that duplicate registration still returns `201`.
  6. Add a test that duplicate registration does not change the stored password hash.
  7. Run:
     ```bash
     GOCACHE=/tmp/kosync-gocache go test ./internal/api ./internal/database
     ```
  8. Commit in KOSYNC:
     ```bash
     git add internal/api
     git commit -m "Harden registration timing behavior"
     ```

Acceptance criteria for Phase 5:

- KOSYNC public registration is disabled unless explicitly enabled.
- KOSYNC rate limiting is wired into real routes.
- KOPDS has comparable failed-auth protection.
- KOSYNC response content type matches the KOReader protocol.

## Phase 6: KOPDS Correctness And Performance

Goal: fix confirmed KOPDS bugs and reduce unnecessary query work.

- [ ] **6.1 Fix KOPDS author listing query**
  1. Open `/home/nathan/koserver/kopds/internal/database/book_repository.go`.
  2. Find `ListByAuthor`.
  3. In the SQL query, change the join from:
     ```sql
     JOIN books_authors_link bal ON b.id = bal.author_id
     ```
     to:
     ```sql
     JOIN books_authors_link bal ON b.id = bal.book_id
     ```
  4. Add a regression test with two books and two authors where author IDs do not equal book IDs.
  5. Confirm the selected author only returns their books.
  6. Run:
     ```bash
     GOCACHE=/tmp/kopds-gocache go test ./internal/database ./internal/api
     ```
  7. Commit in KOPDS:
     ```bash
     git add internal/database
     git commit -m "Fix author book listing query"
     ```

- [ ] **6.2 Validate KOPDS path IDs**
  1. Open `/home/nathan/koserver/kopds/internal/api/handlers.go`.
  2. Add a helper named `parsePositiveID(value string) (int64, error)`.
  3. The helper should:
     - Call `strconv.ParseInt(value, 10, 64)`.
     - Return an error if parsing fails.
     - Return an error if the ID is less than or equal to zero.
  4. Use this helper in:
     - `AuthorBooksHandler`
     - `SeriesBooksHandler`
     - `TagBooksHandler`
     - `BookDetailHandler`
     - `CoverHandler`
     - `BookFileHandler`
  5. Return `400 Bad Request` for invalid IDs.
  6. Add tests for invalid author, series, tag, book, cover, and download IDs.
  7. Run:
     ```bash
     GOCACHE=/tmp/kopds-gocache go test ./internal/api
     ```
  8. Commit in KOPDS:
     ```bash
     git add internal/api
     git commit -m "Validate OPDS path identifiers"
     ```

- [ ] **6.3 Check row iteration errors**
  1. Open `/home/nathan/koserver/kopds/internal/database/book_repository.go`.
  2. In `Search`, after iterating rows and before using the IDs, check:
     ```go
     if err := rows.Err(); err != nil {
         return nil, 0, err
     }
     ```
  3. In `listBooks`, do the same after scanning IDs.
  4. Prefer `defer rows.Close()` after checking that `QueryContext` succeeded.
  5. Run:
     ```bash
     GOCACHE=/tmp/kopds-gocache go test ./internal/database
     ```
  6. Commit in KOPDS:
     ```bash
     git add internal/database/book_repository.go
     git commit -m "Check book row iteration errors"
     ```

- [ ] **6.4 Remove duplicate KOPDS reindex helper**
  1. Open `/home/nathan/koserver/kopds/internal/database/book_repository.go`.
  2. Compare method `func (r *sqliteBookRepository) ReindexBook(...)` with package function `func ReindexBook(...)`.
  3. If the package function is unused, delete it.
  4. If it is still needed, make both call a single private helper that contains the SQL once.
  5. Run:
     ```bash
     GOCACHE=/tmp/kopds-gocache go test ./internal/database ./internal/scanner
     ```
  6. Commit in KOPDS:
     ```bash
     git add internal/database/book_repository.go
     git commit -m "Deduplicate book reindexing logic"
     ```

- [ ] **6.5 Escape Calibre SQLite read-only DSN**
  1. Open `/home/nathan/koserver/kopds/internal/scanner/calibre_reader.go`.
  2. Find:
     ```go
     dsn := fmt.Sprintf("file:%s?mode=ro", path)
     ```
  3. Replace the ad hoc string building with a safe SQLite URI construction that handles spaces, `?`, `#`, and other special characters in file paths.
  4. Add tests using a temporary database path with spaces and special characters.
  5. Run:
     ```bash
     GOCACHE=/tmp/kopds-gocache go test ./internal/scanner
     ```
  6. Commit in KOPDS:
     ```bash
     git add internal/scanner
     git commit -m "Escape Calibre database URI"
     ```

- [ ] **6.6 Batch hydrate listed books**
  1. Open `/home/nathan/koserver/kopds/internal/database/book_repository.go`.
  2. Find `listBooks` and `Search`.
  3. Notice they first fetch IDs and then call `GetByID` once per book.
  4. Add a helper that fetches all requested books and their authors, tags, formats, and series in batches.
  5. Keep output order the same as the ID order.
  6. Add tests proving list order is unchanged.
  7. Benchmark or log query count before and after if possible.
  8. Run:
     ```bash
     GOCACHE=/tmp/kopds-gocache go test ./internal/database ./internal/api
     ```
  9. Commit in KOPDS:
     ```bash
     git add internal/database
     git commit -m "Batch load listed books"
     ```

Acceptance criteria for Phase 6:

- Author pages return the right books.
- Invalid numeric path values return `400`.
- Search/list code checks row iteration errors.
- KOPDS uses fewer repeated queries for paginated book lists.

## Phase 7: KOSYNC Correctness And Performance

Goal: make KOSYNC sync behavior more observable and avoid unnecessary maintenance work.

- [ ] **7.1 Report stale progress updates**
  1. Open `/home/nathan/koserver/kosync/internal/database/sqlite.go`.
  2. Change `UpsertProgress` so callers can tell whether a row was inserted or updated.
  3. One simple approach is to return `(bool, error)` where `true` means the row changed.
  4. Update callers in `internal/api/handlers.go`.
  5. If a client sends an older or equal timestamp, return `200 OK` but log that the stale update was ignored.
  6. Add tests for:
     - New progress returns changed.
     - Newer timestamp returns changed.
     - Older timestamp returns unchanged.
     - Equal timestamp returns unchanged.
  7. Run:
     ```bash
     GOCACHE=/tmp/kosync-gocache go test ./internal/database ./internal/api
     ```
  8. Commit in KOSYNC:
     ```bash
     git add internal/database internal/api
     git commit -m "Report stale progress updates"
     ```

- [ ] **7.2 Skip storage cap checks when progress did not change**
  1. Open `/home/nathan/koserver/kosync/internal/api/handlers.go`.
  2. Use the changed/unchanged result from `UpsertProgress`.
  3. Only call `storage.EnforceStorageCap` when a row was inserted or updated.
  4. Add tests proving stale updates do not call storage-cap enforcement. If direct testing is hard, add a small interface around storage and use a fake in the handler test.
  5. Run:
     ```bash
     GOCACHE=/tmp/kosync-gocache go test ./internal/api
     ```
  6. Commit in KOSYNC:
     ```bash
     git add internal/api internal/database
     git commit -m "Skip storage cap for stale progress updates"
     ```

- [ ] **7.3 Add progress timestamp index**
  1. Open `/home/nathan/koserver/kosync/internal/database/sqlite.go`.
  2. In `Migrate`, after creating the `progress` table, add:
     ```sql
     CREATE INDEX IF NOT EXISTS idx_progress_timestamp ON progress(timestamp);
     ```
  3. Add a test that queries `sqlite_master` and confirms the index exists.
  4. Run:
     ```bash
     GOCACHE=/tmp/kosync-gocache go test ./internal/database
     ```
  5. Commit in KOSYNC:
     ```bash
     git add internal/database
     git commit -m "Index progress timestamps"
     ```

Acceptance criteria for Phase 7:

- Stale progress updates are visible in logs and do not pretend to update storage.
- Storage cap work only runs when progress changes.
- Progress pruning has an index to support timestamp ordering.

## Phase 8: Module, Docs, Deployment, And Integration

Goal: make the public project shape and documentation match the final behavior.

- [ ] **8.1 Change KOSYNC module path**
  1. Open `/home/nathan/koserver/kosync/go.mod`.
  2. Change:
     ```go
     module kosync
     ```
     to:
     ```go
     module github.com/nlafevers/kosync
     ```
  3. Replace imports that start with `kosync/` with `github.com/nlafevers/kosync/`.
  4. Run:
     ```bash
     GOCACHE=/tmp/kosync-gocache go test ./...
     ```
  5. Commit in KOSYNC:
     ```bash
     git add .
     git commit -m "Use canonical module path"
     ```

- [ ] **8.2 Update app documentation**
  1. Open both app READMEs.
  2. Update Go version requirements.
  3. Update rate-limit configuration references.
  4. Update KOSYNC registration docs to say registration is disabled by default.
  5. Update KOSYNC docs to prefer `KOSYNC_DATABASE_PATH` and mention `KOSYNC_DB_PATH` only as a legacy alias.
  6. Update examples in `config/config.yaml` and `deploy/docker-compose.yml`.
  7. Commit documentation changes separately in each app repo.

- [ ] **8.3 Update uniformity inventories**
  1. Open:
     - `/home/nathan/koserver/kopds/UNIFORMITY.md`
     - `/home/nathan/koserver/kosync/UNIFORMITY.md`
  2. Add newly identical or intentionally aligned items:
     - CLI storage helpers.
     - `InitDB`.
     - `UpdatePassword`.
     - request ID generation.
     - rate-limit helpers.
     - storage-cap disabled check.
  3. Add final audit notes for intentional differences.
  4. Keep the two files as close to identical as practical.
  5. Commit in each app repo:
     ```bash
     git add UNIFORMITY.md
     git commit -m "Update uniformity inventory"
     ```

- [ ] **8.4 Update root uniformity plan**
  1. Open `/home/nathan/koserver/uniformity-plan.md`.
  2. Keep the completed historical checklist intact or clearly mark it as the previous round.
  3. Add a short pointer to this roadmap.
  4. Do not duplicate every instruction from this file unless desired.
  5. Commit in the KOSERVER root repo:
     ```bash
     git add uniformity-plan.md implementation-roadmap.md
     git commit -m "Add fresh uniformity implementation roadmap"
     ```

- [ ] **8.5 Strengthen integration tests**
  1. Open both integration scripts:
     - `/home/nathan/koserver/kopds/test/integration_test.sh`
     - `/home/nathan/koserver/kosync/test/integration_test.sh`
  2. Make each script assert HTTP status codes, not only response bodies.
  3. Make KOSYNC assert the KOReader content type.
  4. Make KOPDS assert authenticated catalog access and unauthorized failures.
  5. Ensure each script builds the binary, creates a temp database through the CLI, starts the server, tests HTTP behavior, and cleans up.
  6. Run scripts in an environment that allows local listening:
     ```bash
     cd /home/nathan/koserver/kopds
     ./test/integration_test.sh

     cd /home/nathan/koserver/kosync
     ./test/integration_test.sh
     ```
  7. Commit integration test updates separately in each app repo.

Acceptance criteria for Phase 8:

- KOSYNC has a canonical GitHub module path.
- Docs match the new default security posture.
- Uniformity inventories are current.
- Integration tests check status codes and MIME behavior.

## Final Verification Checklist

Run this only after all phases are done.

- [ ] **Final KOPDS verification**
  1. Open `/home/nathan/koserver/kopds`.
  2. Run:
     ```bash
     gofmt -l .
     GOCACHE=/tmp/kopds-gocache go vet ./...
     GOCACHE=/tmp/kopds-gocache go test ./...
     GOCACHE=/tmp/kopds-gocache go run golang.org/x/vuln/cmd/govulncheck@latest ./...
     ./test/integration_test.sh
     ```
  3. Confirm all commands pass.

- [ ] **Final KOSYNC verification**
  1. Open `/home/nathan/koserver/kosync`.
  2. Run:
     ```bash
     gofmt -l .
     GOCACHE=/tmp/kosync-gocache go vet ./...
     GOCACHE=/tmp/kosync-gocache go test ./...
     GOCACHE=/tmp/kosync-gocache go run golang.org/x/vuln/cmd/govulncheck@latest ./...
     ./test/integration_test.sh
     ```
  3. Confirm all commands pass.

- [ ] **Final root documentation verification**
  1. Open `/home/nathan/koserver`.
  2. Run:
     ```bash
     git status --short
     ```
  3. Confirm only intentional root documentation files are changed.
  4. Commit root documentation changes in the KOSERVER repo.

## Notes For Future Implementers

- Copy-paste similarity is preferred for equivalent behavior. If a helper does the same job in both apps, use the same name, same argument order, same local variable names, and same control flow wherever possible.
- Do not force domain-specific code into artificial sameness. KOPDS owns OPDS catalog, Calibre scanning, images, and book files. KOSYNC owns KOReader sync protocol, progress rows, registration, and header auth.
- Keep compatibility aliases only when they protect current users. Prefer the uniform new names in docs and examples.
- When a test fails, fix the cause before moving to the next checkbox. Do not stack several unrelated changes into one commit.
