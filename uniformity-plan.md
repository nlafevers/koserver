# Uniformity Plan: KOSYNC and KOPDS

This plan supersedes broad "harmonization" as the primary refactoring goal. The new goal is maximum practical uniformity: functions and modules that do the same thing in both projects should have identical names and identical code wherever their behavior is truly the same. Where project domains differ, the implementation should still be shaped so the shared scaffolding is identical and only the domain-specific parts vary.

At the beginning of work on each step, prior to making any changes to any code, place a mark in the box next to that step (eg. [-]) so that it is clear what work was being done if there is a sudden interruption. After the completion of each step, git commit the changes with a descriptive message, then update this document to show the current state of progress by checking the box next to the step (eg. [x]).

## 1. Baseline Findings

### Currently Identical Functions

- `passwordFromArgs` in `kopds/cmd/kopds/main.go` and `kosync/cmd/kosync/main.go`
- `readPasswordInteractively` in `kopds/cmd/kopds/main.go` and `kosync/cmd/kosync/main.go`

### Near-Identical or Equivalent Functions to Standardize

- `printUsage` in both CLI entrypoints
- `HashPassword` and password-check helpers in both API packages
- `logger.New` in KOPDS and `logger.Init` in KOSYNC
- `config.Load` in both config packages
- `database.NewSQLite` in KOPDS and `database.InitDB` in KOSYNC
- `Storage.EnforceStorageCap` in both database packages
- CLI user-management command flow and output

### Project-Specific Code That Should Not Be Forced Into Identical Form

- KOPDS OPDS catalog, Calibre scanner, image cache, book repository, and link-generation logic
- KOSYNC KOReader sync protocol handlers, progress storage, registration endpoint, and header authentication middleware
- Database schemas where the stored domain is different: OPDS catalog/index data in KOPDS versus progress sync data in KOSYNC

## 2. Standardization Choices

- Use KOPDS as the default structural baseline because its `cmd/`, `internal/`, repository, config, and logger organization is more mature.
- Use KOSYNC's bcrypt cost `12` as the password-hashing baseline because it is the stronger explicit security choice.
- Use stricter SQLite directory permissions of `0750` and database file permissions of `0600`.
- Prefer identical duplicated code inside each repository over introducing a shared module, because the projects are tracked as separate repositories.
- Keep backwards-compatible KOSYNC support for `db_path` and `KOSYNC_DB_PATH`, but make `database_path` and `KOSYNC_DATABASE_PATH` the preferred names for uniformity.

## 3. Roadmap

At the beginning of work on each step, prior to making any changes to any code, place a mark in the box next to that step (eg. [-]) so that it is clear what work was being done if there is a sudden interruption. After the completion of each step, git commit the changes with a descriptive message, then update this document to show the current state of progress by checking the box next to the step (eg. [x]).

### Phase 1: Stabilize the Current Baseline

- [x] **1.1 Repair KOPDS Scanner Tests**: Update scanner tests for the current `NewSyncEngine(repo, libraryPath, dbPath, capMB, logger)` signature.
- [x] **1.2 Repair KOSYNC API Tests**: Update progress-handler tests to pass a `config.Config` with the required database path and storage cap fields.
- [x] **1.3 Capture Uniformity Inventory**: Add or update developer-facing notes that identify the currently identical functions and the high-confidence standardization targets listed in this document.

### Phase 2: Authentication and Logging Uniformity

- [x] **2.1 Standardize Password Hashing**: Make both projects use identical `HashPassword(password string) (string, error)` code with bcrypt cost `12`.
- [x] **2.2 Standardize Password Checking**: Make both projects expose and use identical `CheckPassword(hash, password string) bool` code; update KOPDS callers from `CheckPasswordHash`.
- [x] **2.3 Standardize Logger API**: Rename or wrap KOSYNC logging so both projects call `logger.New(level string, json bool, logPath string) *slog.Logger`.
- [x] **2.4 Standardize Logger Behavior**: Make both logger implementations identical, including output stream, text/JSON handler selection, file teeing, and default slog registration.

### Phase 3: Configuration Uniformity

- [x] **3.1 Normalize KOSYNC Config Names**: Prefer `DatabasePath`, `Port int`, `LogLevel`, `JSONLog`, `LogPath`, and `StorageCapMB` in KOSYNC to match KOPDS naming style.
- [x] **3.2 Preserve KOSYNC Compatibility Aliases**: Keep `db_path` and `KOSYNC_DB_PATH` working as deprecated aliases for `database_path` and `KOSYNC_DATABASE_PATH`.
- [x] **3.3 Extract Identical Path Resolution**: Add identical helper code in both config packages for resolving database, log, and cache paths relative to the executable.
- [x] **3.4 Align Config Examples**: Update `config/config.yaml` in both projects so shared settings use the same names and formatting where applicable.

### Phase 4: SQLite Lifecycle Uniformity

- [x] **4.1 Add Identical SQLite Open Helper**: Add `OpenSQLite(path string, allowCreate bool) (*sql.DB, error)` to both database packages with identical directory creation, file permissions, WAL, busy timeout, `SetMaxOpenConns(1)`, and `Ping` behavior.
- [x] **4.2 Refactor KOPDS Database Initialization**: Make `NewSQLite(path)` call `OpenSQLite(path, true)` and keep KOPDS schema migration in `Migrate(db)`.
- [x] **4.3 Refactor KOSYNC Database Initialization**: Make `InitDB(path, allowCreate)` call `OpenSQLite`, then run KOSYNC schema migration.
- [x] **4.4 Align Migration Naming**: Add or rename KOSYNC migration code so both projects have a `Migrate(db)` entrypoint, while keeping schemas project-specific.
- [x] **4.5 Verify Database Creation Parity**: Confirm user-management CLI commands can create and migrate the configured database in both projects.

### Phase 5: Storage-Cap Uniformity

- [x] **5.1 Extract Shared Cap Threshold Logic**: Make the cap-disabled, stat, size comparison, and VACUUM flow identical in both projects.
- [x] **5.2 Isolate Project-Specific Pruning SQL**: Keep only the DELETE target query project-specific: KOPDS prunes catalog sync state or index records as appropriate, while KOSYNC prunes progress rows.
- [x] **5.3 Add Focused Storage-Cap Tests**: Add tests proving cap-disabled behavior, below-cap no-op behavior, over-cap pruning, and error handling are equivalent at the shared-helper level.

### Phase 6: CLI Uniformity

- [x] **6.1 Standardize CLI Dispatch**: Make `runCLI`, `printUsage`, `passwordFromArgs`, and `readPasswordInteractively` identical by using an `appName` constant and command-specific action functions.
- [x] **6.2 Standardize CLI Database Behavior**: Remove KOSYNC's requirement that the database already exist before user-management commands; CLI database creation should match KOPDS.
- [x] **6.3 Standardize CLI Output**: Remove KOSYNC's `Using database:` and `Using log:` lines so both projects print the same style of command result and errors.
- [x] **6.4 Standardize CLI User Operations**: Expose identical CLI-facing `SaveUser`, `DeleteUser`, and `UpdatePassword` behavior in both projects; `create-user` should fail if the user already exists in both projects.
- [x] **6.5 Expand CLI Tests**: Test exact stdout/stderr behavior, database auto-creation, duplicate create failure behavior, delete, change-password, and missing-user failures in both projects.

### Phase 7: Dependency, Deployment, and Integration Parity

- [x] **7.1 Align Shared Dependencies**: Align shared dependency versions, especially `golang.org/x/crypto`, `golang.org/x/term`, `github.com/spf13/viper`, and `modernc.org/sqlite`.
- [x] **7.2 Align Dockerfile Patterns**: Keep project-specific binary names and ports, but make shared Dockerfile structure and comments identical where possible.
- [x] **7.3 Align Compose Patterns**: Keep project-specific volumes and environment variables, but make shared compose structure and security posture identical where possible.
- [x] **7.4 Update Integration Tests**: Make both shell integration tests build the binary, create a temp DB through the CLI, assert CLI output, start the server with explicit env paths, and clean up configured artifacts.

### Phase 8: Documentation and Final Verification

- [x] **8.1 Update READMEs**: Document the uniform CLI behavior, preferred config names, compatibility aliases, DB auto-creation behavior, and logging behavior.
- [x] **8.2 Update GEMINI Files**: Reflect the new uniformity standard and the final project-specific boundaries.
- [x] **8.3 Run Unit Tests**: Run `go test ./...` in both `kopds` and `kosync`.
- [x] **8.4 Run Integration Tests**: Run each integration script in an environment that allows local listening and HTTP calls.
- [x] **8.5 Final Uniformity Audit**: Re-run a function-level similarity inventory and record any remaining equivalent functions that are not identical, with an explanation for each accepted difference.

## 4. Acceptance Criteria

- Equivalent functions have identical names and identical code wherever their responsibilities are the same.
- Any remaining non-identical equivalent behavior is justified by a real domain difference, not style drift.
- KOSYNC and KOPDS CLI user-management commands behave the same for database creation, success messages, errors, and password input.
- Both projects pass their unit tests.
- Both integration scripts pass outside sandboxed environments that block local listeners.
- Documentation reflects the final behavior and no longer describes the old KOSYNC CLI database guard as desired behavior.
