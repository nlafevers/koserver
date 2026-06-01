# Uniformity Round 3 Implementation Roadmap

This roadmap implements the third-audit findings recorded in `uniformity-plan.md` (Round 3), `kopds/UNIFORMITY.md`, and `kosync/UNIFORMITY.md`. Round 3 is a pruning round: two prior rounds of uniformity work left behind **uniform code that was never wired in** and **legacy code that was never removed**. The goal is to delete that dead/duplicate code, standardize one redundant wrapper, collapse one duplicated storage-cap code path, and reconcile the uniformity inventories — without changing any runtime behavior or any intentional domain boundary.

The audit found **no new security vulnerabilities**: the Go standard-library `govulncheck` issue was already closed in Round 2 (Go 1.26.x), and the KOSYNC registration path was re-verified safe against account takeover (`HandleUserCreate` checks `GetUserHash` before any write — `kosync/internal/api/handlers.go:54`). Round 3 is therefore bloat removal and inventory correction only.

## Git Repository Structure

KOSERVER is documentation only. KOPDS and KOSYNC are separate Git repositories inside this workspace. Code changes inside `kopds/` must be committed in the KOPDS repository. Code changes inside `kosync/` must be committed in the KOSYNC repository. Changes to this roadmap or other root documentation (including `uniformity-plan.md`) must be committed in the KOSERVER repository. Do not copy commits across repositories. See the workspace `AGENTS.md` for full policy.

## How To Use This Roadmap

When running Go commands in this workspace, prefer a writable cache under `/tmp`:

```bash
GOCACHE=/tmp/kopds-gocache go test ./...
GOCACHE=/tmp/kosync-gocache go test ./...
```

Implement one incomplete step at a time, in order. Each step is self-contained: read only the listed files, edit only the listed files, run the listed verify commands, and confirm the "Done when" criteria before committing. Removing a symbol and removing/adapting its tests belong to the **same** step so the build never breaks between commits.

## Audit Findings To Address

- KOPDS carries a full parallel `*Storage` user-management surface (`CreateUserIfNotExists`, `SaveUser`, `GetUserHash`, `UpdatePassword`, `DeleteUser`) that was copied to mirror KOSYNC but is never wired into production; production uses `sqliteUserRepository`.
- KOPDS `NewStorage` has no non-test callers; production builds `&Storage{db: r.db}` directly.
- KOPDS `UserRepository.Save` / `sqliteUserRepository.Save` has no non-test callers.
- KOPDS `(*BookService).GetLinkGenerator` has no callers anywhere.
- KOPDS `internal/opds/atom.go` defines OPDS Atom symbols that are never populated (`AtomNamespace`, `IndirectAcquisition` + its `Link` field, `Category` + its `Entry` field, and several never-assigned scalar `Link`/`Entry`/`Feed` fields).
- KOPDS `go.mod` mislabels `golang.org/x/time` as `// indirect` though it is a direct dependency.
- KOSYNC `NewSQLite` has no callers; production calls `OpenSQLite` directly. KOPDS calls `NewSQLite` at every site, leaving `OpenSQLite` reachable only through the wrapper. The two apps disagree on which name to call.
- KOSYNC `GetRequestID` has no callers.
- KOSYNC `models.User` has no references.
- KOSYNC has both `UpdatePassword` (one-line wrapper) and the legacy `UpdateUserPassword` it delegates to.
- KOSYNC `SaveUser` is a redundant hop reached only by `CreateUser`.
- Both apps duplicate the storage-cap size gate: `(*Storage).EnforceStorageCap` stats and gates, then calls the package-level `enforceStorageCap`, which re-stats and re-gates.
- The `UNIFORMITY.md` "Currently Identical Functions" list still lists `NewStorage` and `NewSQLite`, which this round makes asymmetric or removes.

## Phase 1: KOPDS Dead-Code Prune

Goal: remove KOPDS code that has no non-test callers, updating or deleting only the tests that exercise the removed symbols.

- [ ] **UR3-1.1** Remove KOPDS dead `*Storage` user methods
  - **Repos:** kopds
  - **Read:** kopds/internal/database/sqlite.go, kopds/internal/database/user_repository_test.go
  - **Edit:** kopds/internal/database/sqlite.go, kopds/internal/database/user_repository_test.go
  - **Instructions:** In `kopds/internal/database/sqlite.go`, delete the five `*Storage` user methods that have zero production callers: `CreateUserIfNotExists` (the `func (s *Storage) CreateUserIfNotExists(...)` version, NOT the repository version in `user_repository.go`), `SaveUser`, `GetUserHash`, `UpdatePassword`, and `DeleteUser`. Keep the `Storage` struct, its `logger()` method, `EnforceStorageCap`, `pruneStorageCapRecords`, `vacuum`, and the package-level `enforceStorageCap`. In `kopds/internal/database/user_repository_test.go`, delete the entire `TestStorageUserMethods` test function (it is the only caller of these methods and the only `NewStorage` caller at line 31). Leave `TestUserRepository` in place.
  - **Verify:** `cd kopds && gofmt -l . && GOCACHE=/tmp/kopds-gocache go build ./... && GOCACHE=/tmp/kopds-gocache go test ./internal/database ./internal/api ./cmd/kopds`
  - **Done when:** the five `*Storage` user methods and `TestStorageUserMethods` are gone, gofmt prints nothing, and tests pass

- [ ] **UR3-1.2** Remove KOPDS `NewStorage` constructor
  - **Repos:** kopds
  - **Read:** kopds/internal/database/sqlite.go, kopds/internal/database/storage_cap_integration_test.go, kopds/internal/database/storage_cap_test.go
  - **Edit:** kopds/internal/database/sqlite.go, kopds/internal/database/storage_cap_integration_test.go, kopds/internal/database/storage_cap_test.go
  - **Instructions:** In `kopds/internal/database/sqlite.go`, delete the `NewStorage` function (`func NewStorage(db *sql.DB, log *slog.Logger) *Storage`). Production never calls it; the live storage-cap path builds the struct directly at `book_repository.go:811`. Update the two remaining test call sites to build the struct literal directly: in `storage_cap_integration_test.go` (line ~21) replace `storage := NewStorage(db, slog.Default())` with `storage := &Storage{db: db, log: slog.Default()}`; in `storage_cap_test.go` (line ~138) replace `storage := NewStorage(db, logger)` with `storage := &Storage{db: db, log: logger}`. Both test files are in `package database`, so they may set the unexported `db`/`log` fields directly. Leave the `Storage` struct definition and its `log` field unchanged.
  - **Verify:** `cd kopds && gofmt -l . && GOCACHE=/tmp/kopds-gocache go build ./... && GOCACHE=/tmp/kopds-gocache go test ./internal/database`
  - **Done when:** `NewStorage` is gone, both test sites build the struct literal, gofmt prints nothing, and tests pass

- [ ] **UR3-1.3** Remove KOPDS `UserRepository.Save`
  - **Repos:** kopds
  - **Read:** kopds/internal/domain/interfaces.go, kopds/internal/database/user_repository.go, kopds/internal/database/user_repository_test.go
  - **Edit:** kopds/internal/domain/interfaces.go, kopds/internal/database/user_repository.go, kopds/internal/database/user_repository_test.go
  - **Instructions:** `Save` has no production callers (create uses `CreateUserIfNotExists`, update uses `UpdatePassword`). Remove `Save(ctx context.Context, user *User) error` from the `UserRepository` interface in `kopds/internal/domain/interfaces.go`. Remove the `func (r *sqliteUserRepository) Save(...)` implementation from `kopds/internal/database/user_repository.go`. In `kopds/internal/database/user_repository_test.go`, `TestUserRepository` calls `repo.Save(ctx, user)` at two places (~line 130 to create the user, ~line 167 to change its password): replace the first with `repo.CreateUserIfNotExists(ctx, user)` and the second with `repo.UpdatePassword(ctx, user.Username, user.Password)`. Adjust any assertions that depended on `Save` populating `user.ID` so the test still exercises create-then-update behavior and passes.
  - **Verify:** `cd kopds && gofmt -l . && GOCACHE=/tmp/kopds-gocache go build ./... && GOCACHE=/tmp/kopds-gocache go test ./internal/database ./internal/api ./cmd/kopds`
  - **Done when:** `Save` is gone from both the interface and the implementation, `TestUserRepository` uses `CreateUserIfNotExists`/`UpdatePassword`, gofmt prints nothing, and tests pass

- [ ] **UR3-1.4** Remove KOPDS `BookService.GetLinkGenerator`
  - **Repos:** kopds
  - **Read:** kopds/internal/service/book_service.go
  - **Edit:** kopds/internal/service/book_service.go
  - **Instructions:** In `kopds/internal/service/book_service.go`, delete the `GetLinkGenerator` method (`func (s *BookService) GetLinkGenerator() *utils.LinkGenerator`) and its doc comment. It has zero callers anywhere (handlers receive the `LinkGenerator` directly via `api.NewHandler`). Leave the `linkGenerator` struct field and all other methods intact. If removing the method leaves the `pkg/utils` import unused, remove that import; if other code in the file still uses `utils`, keep it.
  - **Verify:** `cd kopds && gofmt -l . && GOCACHE=/tmp/kopds-gocache go build ./... && GOCACHE=/tmp/kopds-gocache go test ./...`
  - **Done when:** `GetLinkGenerator` is gone, the package compiles, gofmt prints nothing, and tests pass

- [ ] **UR3-1.5** Remove unused OPDS Atom symbols
  - **Repos:** kopds
  - **Read:** kopds/internal/opds/atom.go, kopds/internal/opds/atom_test.go
  - **Edit:** kopds/internal/opds/atom.go
  - **Instructions:** In `kopds/internal/opds/atom.go`, remove the OPDS Atom symbols that are never populated anywhere in the repo. Confirm each has no assignment before deleting by running `grep -rn "<symbol>" kopds --include='*.go'` and checking there is no composite-literal field assignment or const use. Remove: the `AtomNamespace` const; the `IndirectAcquisition` type and the `Link.IndirectAcquisitions` field that references it; the `Category` type and the `Entry.Categories` field that references it. Also remove the never-assigned scalar fields `Link.Count`, `Link.Price`, `Link.Currency`, `Entry.Published`, `Entry.Rights`, and `Feed.Icon`. Do NOT remove `OPDSNamespace` (used at `handlers.go:46`) or any field that is assigned in handlers (e.g. `Content.Summary`, `Content.Type`). `atom_test.go` does not reference any of the removed symbols, so it needs no change — but if the build reports a removed symbol is still used, restore that one symbol and note it.
  - **Verify:** `cd kopds && gofmt -l . && GOCACHE=/tmp/kopds-gocache go build ./... && GOCACHE=/tmp/kopds-gocache go test ./internal/opds ./internal/api`
  - **Done when:** the listed Atom symbols are removed, the package compiles, gofmt prints nothing, and tests pass

- [ ] **UR3-1.6** Promote `golang.org/x/time` to a direct dependency
  - **Repos:** kopds
  - **Read:** kopds/go.mod
  - **Edit:** kopds/go.mod, kopds/go.sum
  - **Instructions:** `golang.org/x/time` is used directly (`rate.Every` in `cmd/kopds/main.go`) but is marked `// indirect` in `kopds/go.mod`. Run `cd kopds && GOCACHE=/tmp/kopds-gocache go mod tidy` to remove the incorrect `// indirect` annotation and reconcile `go.sum`. Do not add or remove any other dependency manually.
  - **Verify:** `cd kopds && GOCACHE=/tmp/kopds-gocache go build ./... && GOCACHE=/tmp/kopds-gocache go test ./... && grep -n "golang.org/x/time" go.mod`
  - **Done when:** `go.mod` lists `golang.org/x/time` without `// indirect`, `go mod tidy` is a no-op on re-run, and tests pass

Acceptance criteria for Phase 1:
- KOPDS has a single user-storage implementation (`sqliteUserRepository`); the `*Storage` user methods and `NewStorage` are gone.
- `UserRepository.Save` and `BookService.GetLinkGenerator` are gone.
- Unused OPDS Atom symbols are gone.
- `go.mod` correctly classifies `golang.org/x/time`.
- `gofmt -l .` prints nothing and `go test ./...` passes in KOPDS.

## Phase 2: KOSYNC Dead-Code Prune

Goal: remove KOSYNC code that has no non-test callers and collapse redundant indirection, without changing API or CLI behavior.

- [ ] **UR3-2.1** Remove KOSYNC `GetRequestID`
  - **Repos:** kosync
  - **Read:** kosync/internal/api/context.go
  - **Edit:** kosync/internal/api/context.go
  - **Instructions:** In `kosync/internal/api/context.go`, delete the `GetRequestID` function (`func GetRequestID(ctx context.Context) string`) and its doc comment. It has zero callers (the request ID is stored in context by the middleware and emitted through the bound logger, never read back out). Keep `GetLogger` and `GetUser` and the context-key definitions they use. If deleting `GetRequestID` leaves a context key or import used only by it, remove that too; otherwise leave keys/imports untouched.
  - **Verify:** `cd kosync && gofmt -l . && GOCACHE=/tmp/kosync-gocache go build ./... && GOCACHE=/tmp/kosync-gocache go test ./internal/api ./cmd/kosync`
  - **Done when:** `GetRequestID` is gone, the package compiles, gofmt prints nothing, and tests pass

- [ ] **UR3-2.2** Remove KOSYNC `models.User`
  - **Repos:** kosync
  - **Read:** kosync/internal/models/models.go
  - **Edit:** kosync/internal/models/models.go
  - **Instructions:** In `kosync/internal/models/models.go`, delete the `User` struct (`type User struct { ... }`) and its doc comment. It is never referenced anywhere (auth and registration work with raw `username` and hash strings). Keep the `Progress` struct and all its fields and tags unchanged — `Progress` is used in `sqlite.go` and `handlers.go`. Confirm with `grep -rn "models.User" kosync --include='*.go'` returning nothing before deleting.
  - **Verify:** `cd kosync && gofmt -l . && GOCACHE=/tmp/kosync-gocache go build ./... && GOCACHE=/tmp/kosync-gocache go test ./...`
  - **Done when:** `models.User` is gone, `models.Progress` is unchanged, the module compiles, gofmt prints nothing, and tests pass

- [ ] **UR3-2.3** Collapse KOSYNC `UpdateUserPassword` into `UpdatePassword`
  - **Repos:** kosync
  - **Read:** kosync/internal/database/sqlite.go, kosync/internal/database/sqlite_test.go
  - **Edit:** kosync/internal/database/sqlite.go
  - **Instructions:** In `kosync/internal/database/sqlite.go`, `UpdatePassword(username, passwordHash string) error` (added in UR2-3.3) is a one-line wrapper whose only job is `return s.UpdateUserPassword(username, passwordHash)`. Move the body of `UpdateUserPassword` directly into `UpdatePassword` (the UPDATE statement, `RowsAffected` check, "user not found" handling, and logging), then delete `UpdateUserPassword`. Keep the public method name `UpdatePassword` and its signature exactly, since the CLI calls it (`cmd/kosync/main.go`). If the inlined log message read "failed to update user password", change it to "failed to update password" to match KOPDS wording.
  - **Verify:** `cd kosync && gofmt -l . && GOCACHE=/tmp/kosync-gocache go build ./... && GOCACHE=/tmp/kosync-gocache go test ./internal/database ./cmd/kosync`
  - **Done when:** `UpdateUserPassword` is gone, `UpdatePassword` contains the full implementation, gofmt prints nothing, and tests pass

- [ ] **UR3-2.4** Inline KOSYNC `SaveUser` into `CreateUser`
  - **Repos:** kosync
  - **Read:** kosync/internal/database/sqlite.go, kosync/internal/database/sqlite_test.go
  - **Edit:** kosync/internal/database/sqlite.go
  - **Instructions:** In `kosync/internal/database/sqlite.go`, `SaveUser(username, password string) error` is reached only by `CreateUser` (`return s.SaveUser(username, password)`); it has no other callers, including tests. Move the body of `SaveUser` (the `INSERT ... ON CONFLICT(username) DO UPDATE SET password_hash=excluded.password_hash` upsert plus its logging and error handling) directly into `CreateUser`, then delete `SaveUser`. Keep `CreateUser`'s name and signature exactly (it is called from `internal/api/handlers.go` and tests). Do not change `CreateUserIfNotExists` — it remains the CLI's duplicate-guarded create path. Behavior must be identical: the API registration handler already checks existence before calling `CreateUser`, so the upsert is only ever reached for new users.
  - **Verify:** `cd kosync && gofmt -l . && GOCACHE=/tmp/kosync-gocache go build ./... && GOCACHE=/tmp/kosync-gocache go test ./internal/database ./internal/api ./cmd/kosync`
  - **Done when:** `SaveUser` is gone, `CreateUser` performs the upsert directly, gofmt prints nothing, and tests pass

Acceptance criteria for Phase 2:
- KOSYNC `GetRequestID` and `models.User` are gone.
- KOSYNC has a single `UpdatePassword` (no `UpdateUserPassword`) and a single `CreateUser` (no `SaveUser` hop).
- `gofmt -l .` prints nothing and `go test ./...` passes in KOSYNC.

## Phase 3: Shared Wrapper Standardization And Storage-Cap Dedup

Goal: standardize the SQLite-open call name across both apps and remove the redundant wrapper, and eliminate the duplicated storage-cap size gate in both apps while preserving the package-level test seam.

- [ ] **UR3-3.1** Standardize on `OpenSQLite` and remove `NewSQLite` in both apps
  - **Repos:** kopds, kosync
  - **Read:** kopds/cmd/kopds/main.go, kopds/internal/database/sqlite.go, kosync/internal/database/sqlite.go
  - **Edit:** kopds/cmd/kopds/main.go, kopds/internal/database/sqlite.go, kosync/internal/database/sqlite.go
  - **Instructions:** `NewSQLite(path, allowCreate)` is a one-line wrapper that just calls `OpenSQLite(path, allowCreate)`. KOSYNC already calls `OpenSQLite` directly and never calls `NewSQLite`; KOPDS calls `NewSQLite`. Standardize both apps on `OpenSQLite`: (1) In `kopds/cmd/kopds/main.go`, replace every `database.NewSQLite(` call with `database.OpenSQLite(` (confirm the set with `grep -n "NewSQLite" kopds/cmd/kopds/main.go`; there are call sites in `openCLIStorage` and `runServer`). (2) Delete the `NewSQLite` function from `kopds/internal/database/sqlite.go`. (3) Delete the `NewSQLite` function from `kosync/internal/database/sqlite.go` (it is already dead there). Do not change `OpenSQLite` itself. Commit KOPDS and KOSYNC separately.
  - **Verify:** `cd kopds && gofmt -l . && GOCACHE=/tmp/kopds-gocache go build ./... && GOCACHE=/tmp/kopds-gocache go test ./... && cd ../kosync && gofmt -l . && GOCACHE=/tmp/kosync-gocache go build ./... && GOCACHE=/tmp/kosync-gocache go test ./...`
  - **Done when:** neither repo defines `NewSQLite`, all SQLite opens call `OpenSQLite`, gofmt prints nothing in both, and tests pass in both; one commit per repo

- [ ] **UR3-3.2** Remove the duplicated storage-cap size gate in both apps
  - **Repos:** kopds, kosync
  - **Read:** kopds/internal/database/sqlite.go, kopds/internal/database/storage_cap_test.go, kosync/internal/database/sqlite.go, kosync/internal/database/storage_cap_test.go
  - **Edit:** kopds/internal/database/sqlite.go, kosync/internal/database/sqlite.go
  - **Instructions:** In both apps, `(*Storage).EnforceStorageCap` does `os.Stat` + size-vs-cap comparison and only then calls the package-level `enforceStorageCap`, which performs the `capMB <= 0` guard, a second `os.Stat`, and the same size comparison again before pruning. Keep the package-level `enforceStorageCap(path, capMB, prune, vacuum)` function and its signature unchanged — `storage_cap_test.go` in both apps calls it directly as a unit and must keep passing. Eliminate the duplication by slimming the **method**: keep the early `if capMB <= 0 { ...; return false, nil }` debug-log guard, then `return enforceStorageCap(path, capMB, s.pruneStorageCapRecords, s.vacuum)` without the method doing its own `os.Stat` / size comparison. So the over-cap warning and `current_size_mb` logging can move into the package-level `enforceStorageCap` (have it compute size and log the "storage cap exceeded" warning + the post-prune "storage cap enforced" info using `s`'s logger is not available there, so pass logging through the existing method, or keep minimal logging in the method around the single delegated call). Net requirement: the file is stat-ed once per `EnforceStorageCap` invocation, the package-level function remains the tested gate, and the existing `storage_cap_test.go` and `storage_cap_integration_test.go` still pass unchanged. Implement identically in both apps.
  - **Verify:** `cd kopds && gofmt -l . && GOCACHE=/tmp/kopds-gocache go test ./internal/database && cd ../kosync && gofmt -l . && GOCACHE=/tmp/kosync-gocache go test ./internal/database`
  - **Done when:** `EnforceStorageCap` no longer performs its own redundant `os.Stat`/size comparison, the package-level `enforceStorageCap` signature is unchanged, both apps are implemented identically, gofmt prints nothing, and all storage-cap tests pass; one commit per repo

Acceptance criteria for Phase 3:
- Neither app defines `NewSQLite`; all SQLite opens go through `OpenSQLite`.
- The storage-cap file size is stat-ed once per enforcement; the package-level `enforceStorageCap` test seam is intact.
- Storage-cap unit and integration tests pass in both apps.

## Phase 4: Inventory And Plan Reconciliation

Goal: update the uniformity inventories and the round plan to reflect the completed prune, keeping the two `UNIFORMITY.md` files byte-identical.

- [ ] **UR3-4.1** Reconcile the `UNIFORMITY.md` inventories
  - **Repos:** kopds, kosync
  - **Read:** kopds/UNIFORMITY.md, kosync/UNIFORMITY.md
  - **Edit:** kopds/UNIFORMITY.md, kosync/UNIFORMITY.md
  - **Instructions:** Update both `UNIFORMITY.md` files (they must remain byte-identical). In the "Currently Identical Functions" list, remove `NewSQLite` (no longer defined in either app) and remove `NewStorage` (now KOSYNC-only). Under "Intentional Project Boundaries", add an entry: KOSYNC owns the `NewStorage` constructor because its `Storage` is the primary storage object; KOPDS builds `&Storage{...}` inline as a thin storage-cap adapter and has no `NewStorage`. Add a note that both apps open SQLite via `OpenSQLite` directly (no `NewSQLite` wrapper). In the "Round 3 Audit" section, convert the pruning-candidate bullets into a short "Round 3 completed" summary stating that the dead `*Storage` user methods, `NewStorage` (KOPDS), `UserRepository.Save`, `GetLinkGenerator`, unused Atom symbols, `GetRequestID`, `models.User`, `UpdateUserPassword`, `SaveUser`, and `NewSQLite` were removed, and the storage-cap gate de-duplicated. Remove the stale "Inventory corrections … entry was removed pending the prune" wording for `UpdatePassword` and instead state plainly that `UpdatePassword` is intentionally NOT cross-identical (KOPDS: `sqliteUserRepository.UpdatePassword`, context-based; KOSYNC: `(*Storage).UpdatePassword`). After editing, confirm the files are identical.
  - **Verify:** `diff kopds/UNIFORMITY.md kosync/UNIFORMITY.md && echo IDENTICAL`
  - **Done when:** `NewSQLite`/`NewStorage` are removed from the identical list, the new boundaries and Round 3 summary are present, `diff` reports the files identical, and the command prints `IDENTICAL`; one commit per repo

- [ ] **UR3-4.2** Mark Round 3 complete in the root plan
  - **Repos:** koserver
  - **Read:** uniformity-plan.md, ur3-roadmap.md
  - **Edit:** uniformity-plan.md
  - **Instructions:** In `uniformity-plan.md`, change the "Uniformity Round 3 — Audit Findings (June 2026)" heading to "Uniformity Round 3 — Completed" and add a sentence pointing to `ur3-roadmap.md` for the per-step implementation and commits (mirroring how the Round 2 section points to `ur2-roadmap.md`). Keep the existing findings prose but add a leading note that the findings were implemented in `ur3-roadmap.md`. Do not delete the findings detail.
  - **Verify:** visual inspection that the Round 3 section is marked completed and references `ur3-roadmap.md`
  - **Done when:** the Round 3 section is marked completed and points to `ur3-roadmap.md`

Acceptance criteria for Phase 4:
- Both `UNIFORMITY.md` files are byte-identical and reflect the post-prune reality.
- `uniformity-plan.md` marks Round 3 complete and references this roadmap.

## Phase 5: Final Verification Checklist

Run only after all earlier phases are complete.

- [ ] **FINAL-3.1** Final KOPDS verification
  - **Repos:** kopds
  - **Read:** none
  - **Edit:** none
  - **Instructions:** Run the full KOPDS quality gate: formatting, vet, tests, vulnerability scan, and the integration script. Fix nothing here — if a check fails, return to the relevant phase step and fix the cause there.
  - **Verify:** `cd kopds && gofmt -l . && GOCACHE=/tmp/kopds-gocache go vet ./... && GOCACHE=/tmp/kopds-gocache go test ./... && GOCACHE=/tmp/kopds-gocache go run golang.org/x/vuln/cmd/govulncheck@latest ./... && ./test/integration_test.sh`
  - **Done when:** all commands pass

- [ ] **FINAL-3.2** Final KOSYNC verification
  - **Repos:** kosync
  - **Read:** none
  - **Edit:** none
  - **Instructions:** Run the full KOSYNC quality gate: formatting, vet, tests, vulnerability scan, and the integration script. Fix nothing here — if a check fails, return to the relevant phase step and fix the cause there.
  - **Verify:** `cd kosync && gofmt -l . && GOCACHE=/tmp/kosync-gocache go vet ./... && GOCACHE=/tmp/kosync-gocache go test ./... && GOCACHE=/tmp/kosync-gocache go run golang.org/x/vuln/cmd/govulncheck@latest ./... && ./test/integration_test.sh`
  - **Done when:** all commands pass

- [ ] **FINAL-3.3** Final root documentation verification
  - **Repos:** koserver
  - **Read:** none
  - **Edit:** none
  - **Instructions:** Confirm the workspace root has only the intended documentation changes and that the two inventories are identical.
  - **Verify:** `git status --short && diff kopds/UNIFORMITY.md kosync/UNIFORMITY.md && echo IDENTICAL`
  - **Done when:** only intended root docs changed and the inventories are byte-identical

Acceptance criteria for Phase 5:
- KOPDS and KOSYNC pass gofmt, vet, tests, govulncheck, and integration tests.
- Root documentation is clean and the inventories are identical.

## Notes For Future Implementers

- This round only removes code with no non-test callers, collapses redundant indirection, and de-duplicates one code path. No runtime behavior, API response, CLI output, or schema changes are intended. If a step seems to require a behavior change to make tests pass, stop and re-read — the test, not the behavior, is what should change.
- Removing a symbol and updating/removing its tests is a single step. Never leave a commit where the build is broken.
- **`NewSQLite` is removed from both apps by design** (Phase 3). It was an UR2/TDL-001 uniformity alias for `OpenSQLite`, but KOSYNC never actually called it. The uniform end state is "both apps call `OpenSQLite` directly," which supersedes the earlier "both expose `NewSQLite`" decision.
- **`NewStorage` is intentionally KOSYNC-only after this round.** KOSYNC's `Storage` is the primary storage object constructed via `NewStorage`; KOPDS's `Storage` is a thin storage-cap adapter built inline (`&Storage{db: r.db}` at `book_repository.go:811`). Do not re-add `NewStorage` to KOPDS for the sake of symmetry — this is a documented boundary, not drift.
- **`UpdatePassword` is intentionally NOT cross-identical.** KOPDS's live update is `sqliteUserRepository.UpdatePassword` (context-based, clean-architecture interface); KOSYNC's is `(*Storage).UpdatePassword`. This follows the standing rule "do not force domain-specific code into artificial sameness." Do not try to unify them by reintroducing a `*Storage` user surface in KOPDS.
- **KOSYNC keeps both `CreateUser` and `CreateUserIfNotExists`** with different semantics on purpose: the API registration handler guards existence itself and then calls `CreateUser` (an upsert reached only for new users); the CLI calls `CreateUserIfNotExists` (duplicate-guarded). This naming difference is acceptable and was reviewed during the Round 3 audit; do not collapse them.
- The package-level `enforceStorageCap(path, capMB, prune, vacuum)` function in both apps is a deliberate **test seam** exercised directly by `storage_cap_test.go`. Keep its signature stable; de-duplication happens by slimming the `(*Storage).EnforceStorageCap` method, not by deleting the free function.
- Keep the two `UNIFORMITY.md` files byte-identical; verify with `diff` after every edit.
