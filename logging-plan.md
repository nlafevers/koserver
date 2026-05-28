# Logging Refactor Plan: KOPDS and KOSYNC

This plan refactors the logging system in both KOPDS and KOSYNC to achieve comprehensive coverage of all operations at appropriate log levels, with complete uniformity across both projects. This builds on Phase 2 of the [uniformity-plan.md](uniformity-plan.md), which already standardized the logger API and behavior.

## Current State

Both projects have identical `logger.New(level string, json bool, logPath string)` implementations that output to stderr and optionally to a file. However, logging **coverage** is incomplete and inconsistent:

- **KOPDS**: Minimal logging; no HTTP request/response logging in middleware; CLI operations lack success-level logging
- **KOSYNC**: Better coverage of handler operations, but missing successful GET request logging; incomplete CLI operation logging

## Goals

1. **Comprehensive Coverage**: Every operation (HTTP request, CLI command, internal action) logs success or failure at appropriate levels
2. **Uniform Format**: Both projects use identical logging patterns for equivalent operations
3. **Clarity**: DEBUG provides detailed diagnostic information; INFO provides business-level event summaries
4. **Deployability**: Logs are useful for observability in Docker containers and cloud environments

## Logging Matrix

Define expected log levels for all major event types:

| Operation Class | Event | Severity | Level | Required Fields |
|---|---|---|---|---|
| **HTTP Requests** | Request completed (2xx/3xx) | Business | INFO | method, path, status_code, duration, user (if auth'd) |
| **HTTP Requests** | Detailed response info | Diagnostic | DEBUG | method, path, status_code, duration, response_size, user (if auth'd) |
| **HTTP Requests** | Client error (4xx) | Problem | WARN | method, path, status_code, duration, error_reason |
| **HTTP Requests** | Server error (5xx) | Problem | ERROR | method, path, status_code, duration, error_detail |
| **Authentication** | Successful auth | Business | DEBUG | username, auth_method |
| **Authentication** | Auth failure | Problem | WARN | username, reason, remote_addr |
| **User Management (CLI)** | User created | Success | INFO | username, source (CLI) |
| **User Management (CLI)** | User creation failed | Failure | WARN | username, reason, source (CLI) |
| **User Management (CLI)** | User deleted | Success | INFO | username, source (CLI) |
| **User Management (CLI)** | User deletion failed | Failure | WARN | username, reason, source (CLI) |
| **User Management (CLI)** | Password changed | Success | INFO | username, source (CLI) |
| **User Management (CLI)** | Password change failed | Failure | WARN | username, reason, source (CLI) |
| **User Management (API)** | Registration attempt | Business | DEBUG | username, source (API) |
| **User Management (API)** | Duplicate registration | Problem | WARN | username, source (API) |
| **Data Operations (KOPDS)** | Book scan started | Business | INFO | library_path, scan_type |
| **Data Operations (KOPDS)** | Book scan completed | Success | INFO | library_path, books_found, duration |
| **Data Operations (KOPDS)** | Book scan failed | Failure | ERROR | library_path, error_detail |
| **Data Operations (KOSYNC)** | Progress retrieved | Business | DEBUG | username, document, timestamp |
| **Data Operations (KOSYNC)** | Progress updated | Business | INFO | username, document, percentage, timestamp |
| **Data Operations (KOSYNC)** | Progress update skipped (older) | Business | DEBUG | username, document, reason |
| **Storage Management** | Storage cap checked | Business | DEBUG | database_path, current_size_mb, cap_mb |
| **Storage Management** | Storage pruning executed | Problem | WARN | database_path, rows_deleted, freed_mb |
| **Startup/Shutdown** | Application starting | Business | INFO | app_name, version, port, config_summary |
| **Startup/Shutdown** | Database initialized | Success | INFO | database_path, tables_count |
| **Startup/Shutdown** | Application exiting | Business | INFO | reason (normal/error), uptime |
| **Startup/Shutdown** | Shutdown signal received | Business | INFO | signal_name |
| **Configuration** | Config loaded | Business | INFO | config_source, key_settings_summary |
| **Configuration** | Config validation error | Failure | ERROR | config_key, error_detail |

## Phases

At the beginning of work on each step, prior to making any changes to any code, place a mark in the box next to that step (eg. [-]) so that it is clear what work was being done if there is a sudden interruption. After the completion of each step, git commit the changes with a descriptive message, then update this document to show the current state of progress by checking the box next to the step (eg. [x]).

### Phase 1: Add HTTP Request/Response Logging Middleware

**Objective**: Standardize HTTP request/response logging across both projects with identical middleware.

- [x] **1.0 Rebase KOSYNC Middleware**: Move KOSYNC middleware from `internal/middleware/middleware.go` to `internal/api/middleware.go`, remove or retire the old `internal/middleware` package, and keep the package layout identical to KOPDS.
- [x] **1.1 Design Logging Middleware**: Create `LoggingMiddleware` for stdlib `net/http.ServeMux` that:
  - generates a unique `request_id` for each request,
  - creates a request-scoped logger with `method`, `path`, and `request_id`,
  - stores the request-scoped logger and authenticated username in context,
  - wraps `http.ResponseWriter` to capture status code, response size, and error body for 5xx responses,
  - logs completed requests at INFO level (2xx/3xx) with method, path, status_code, duration, and user (if auth'd),
  - logs client errors at WARN and server errors at ERROR, with full response/error context,
  - emits DEBUG-level logs for detailed response diagnostics,
  - is applied as the outermost middleware around the final router so it captures the full request lifecycle, including auth/rate-limiting.
- [x] **1.2 Standardize Request Context**: Update both KOPDS and KOSYNC auth middleware to store authenticated username in the request context using shared context key constants.
- [x] **1.3 Implement Logging Middleware in KOPDS**: Extend `internal/api/middleware.go` to add `LoggingMiddleware`; apply middleware as the outermost wrapper around the final mux in `main.go`.
- [x] **1.4 Implement Logging Middleware in KOSYNC**: Create `internal/api/middleware.go` with identical `LoggingMiddleware`; apply middleware as the outermost wrapper around the final mux in `main.go`.
- [x] **1.5 Test Logging Middleware**: Add unit tests verifying correct log output for various HTTP status codes and request types, request_id correlation across logs, request-scoped logger propagation, and 5xx error capture.

### Phase 2: Standardize CLI Operation Logging

**Objective**: Ensure all CLI operations (create-user, delete-user, change-password) log at INFO level on success and WARN level on failure.

- [x] **2.1 Create CLI Logging Helpers**: Add `internal/logger/cli.go` in both projects with standardized functions: `LogCLISuccess(logger *slog.Logger, operation, username string)`, `LogCLIFailure(logger *slog.Logger, operation, username, reason string)`. Helpers include `source="CLI"` field in all logs.
- [x] **2.2 Implement KOPDS CLI Logging**: Refactor `cmd/kopds/main.go` user-management functions to call logging helpers; ensure all success and failure paths log at appropriate levels.
- [x] **2.3 Implement KOSYNC CLI Logging**: Refactor `cmd/kosync/main.go` user-management functions to call identical logging helpers; ensure all success and failure paths log at appropriate levels.
- [x] **2.4 Test CLI Logging**: Add tests verifying exact log output (level and message) for success/failure of create, delete, and change-password operations; verify source="CLI" tag is present.

### Phase 3: Standardize API Operation Logging (Handlers)

**Objective**: Ensure all handler operations log success at INFO level and failures at appropriate error/warn levels with complete diagnostic information.

- [x] **3.1 Review Current Handler Logging**: Audit all handlers in both projects; document current gaps:
  - **KOPDS**: Missing success-level logging for all feed handlers; `CoverHandler` lacks cache hit/miss diagnostic logging; error paths in handlers don't log structured errors with context; handlers don't consistently use request-scoped logger.
  - **KOSYNC**: `HandleGetProgress` missing success logging; `HandleUpdateProgress` lacks detailed "skip vs update" diagnostics; handlers don't consistently use request-scoped logger.
- [x] **3.2 Add Uniform Handler Logging in KOPDS**: Add INFO logging for successful OPDS catalog responses, book catalog retrieval, image cache hits/misses, and basic auth successes; add DEBUG logging for detailed diagnostic info. Use `GetLogger(r.Context())`.
- [x] **3.3 Add Uniform Handler Logging in KOSYNC**: Re-implement the missing KOSYNC handler path and restore uniform handler logging across the API package.
  - [x] **3.3.1 Restore `HandleAuth`**: Re-introduce `HandleAuth` in `kosync/internal/api/handlers.go` so the route referenced from `cmd/kosync/main.go` compiles and returns the expected authenticated success response. Use the shared request-scoped logger and request context fields.
  - [x] **3.3.2 Restore Uniform Success Logging**: Use `GetLogger(r.Context())` in `HandleAuth`, `HandleGetProgress`, and `HandleUpdateProgress` so successful operations log at INFO level with structured context (`username`, `document`, `percentage`, `status_code` via middleware where applicable).
  - [x] **3.3.3 Restore Uniform Diagnostic Logging**: Add DEBUG-level logs for progress lookups, updates, auth success, and storage-cap enforcement details; preserve the existing request-scoped logger and context keys.
  - [x] **3.3.4 Restore Handler Coverage**: Re-run or add handler-focused tests proving auth, progress GET, and progress PUT paths complete with the expected log output and handler behavior.
- [x] **3.4 Standardize Error Context in Handlers**: Ensure all handler error paths include username (if auth'd), request path, and error detail; use consistent field names across both projects. Use `GetLogger(r.Context())`.
- [x] **3.5 Test Handler Logging**: Add integration tests for each handler verifying correct log output for success, client errors, and server errors.

### Phase 4: Standardize Startup/Shutdown Logging

**Objective**: Ensure consistent and informative logging for application lifecycle events.

- [x] **4.1 Unify Startup Logging**: Add structured startup logs in both servers and keep them at INFO level.
  - In `kopds/cmd/kopds/main.go`, update `main()` to initialize the logger before CLI branching, then emit an application startup log with `app_name`, `port`, `database_path`, `log_level`, `json_log`, and `log_path` once config is loaded. In `runServer()`, keep the existing database startup log and add a dedicated server-listening log before `ListenAndServe`.
  - In `kosync/cmd/kosync/main.go`, move startup logging to the top of `main()` after `logger.New(...)`, log `app_name`, `port`, `database_path`, `log_level`, `json_log`, and `log_path`, and ensure CLI commands do not emit server startup logs.
  - Use identical field names across projects so startup lines can be compared directly in text and JSON modes.
- [x] **4.2 Unify Shutdown Logging**: Log shutdown events consistently on signal receipt, graceful stop, and shutdown failures.
  - In `kopds/cmd/kopds/main.go`, log `shutdown signal received` with the signal name, log the start of shutdown before `srv.Shutdown`, and log `server exited cleanly` only after shutdown finishes. On shutdown failure, log `server shutdown failed` with `error`.
  - In `kosync/cmd/kosync/main.go`, add the same signal receipt and shutdown completion logs around `server.Shutdown(context.Background())`, and keep `server failed` logs scoped to actual listener failures.
  - Include `reason` and `uptime` fields when available so operators can tell the difference between normal exit and failure.
- [x] **4.3 Unify Database Initialization Logging**: Log database initialization and migration milestones with stable field names.
  - In `kopds/cmd/kopds/main.go`, log `database initialized` after `database.NewSQLite` and `database.Migrate`, including `database_path`, `migration_status`, and `storage_cap_mb` when available.
  - In `kosync/cmd/kosync/main.go`, log `database initialized` after `database.InitDB(...)` and include `database_path` plus the storage cap setting from config.
  - If startup storage-cap enforcement is added later, log that separately so initialization logs stay distinct from runtime maintenance logs.
- [x] **4.4 Test Startup/Shutdown Logging**: Add lifecycle log tests and manual verification steps.
  - Add targeted tests in `kopds/cmd/kopds/main_test.go` and `kosync/cmd/kosync/main_test.go` (or helper-based test wrappers) to assert startup, shutdown, and invalid-config logs.
  - Verify the startup path logs the same keys in both projects, including `app_name`, `port`, and `database_path`.
  - Re-run startup/shutdown scenarios with `LOG_LEVEL=INFO` and `LOG_LEVEL=DEBUG` to ensure the lifecycle events remain visible.

### Phase 5: Standardize Service/Repository Layer Logging

**Objective**: Ensure database operations and service-layer actions log appropriately without duplicating HTTP-level logging.

- [x] **5.1 Review Service Logging Strategy**: Decide on a single pattern for lower layers before coding.
  - Lower layers should log `DEBUG` for query-level diagnostics and `ERROR` for failures, while handlers retain `INFO`-level business success logs.
  - Do not duplicate request-complete logs from middleware; instead, service logs should show query context such as `table`, `operation`, and result counts.
- [x] **5.2 Implement Service Logging in KOPDS**: Add query-level logging in the repository/service stack.
  - In `kopds/internal/database/book_repository.go` and `kopds/internal/database/user_repository.go`, add `DEBUG` logs for lookup, insert, update, and delete operations, and `ERROR` logs when SQL execution fails.
  - In `kopds/internal/service/book_service.go`, log fetch boundaries (for example, author/series/tag lookups) with `DEBUG` so operators can see why handler responses were slow, but keep handler-level success logs out of the service layer.
  - Extend the storage wrapper in `kopds/internal/database/sqlite.go` to carry a logger so repository methods can emit structured, project-scoped logs.
- [x] **5.3 Implement Service Logging in KOSYNC**: Add query-level logging in `internal/database`.
  - In `kosync/internal/database/sqlite.go`, add a logger field to `Storage`, initialize it in `InitDB`, and emit `DEBUG` entries around `GetProgress`, `UpsertProgress`, `CreateUserIfNotExists`, `GetUserHash`, `DeleteUser`, and `UpdateUserPassword`.
  - Log `ERROR` with the SQL operation name and the concrete failure reason on any database error.
  - Preserve handler-level `INFO` success logs in `kosync/internal/api/handlers.go`; service/database logs should remain diagnostic, not duplicate business events.
- [x] **5.4 Test Service Logging**: Verify the lower layers log without duplicating HTTP-level logs.
  - Add unit tests for repository/database methods that assert `DEBUG` and `ERROR` messages are present, while handler-focused tests continue to assert business-level messages.
  - Check that service/database logs include stable fields such as `operation`, `username`, `document`, and `error`.

### Phase 6: Storage Cap and Maintenance Logging

**Objective**: Ensure storage cap enforcement and database maintenance operations log appropriately for operational visibility.

- [x] **6.1 Add Storage Cap Logging**: Add visibility around database size checks, pruning, and vacuuming.
  - In `kopds/internal/database/sqlite.go` and `kosync/internal/database/sqlite.go`, log `DEBUG` before each cap check with `database_path`, `current_size_mb`, and `cap_mb`.
  - When the cap is exceeded, log `WARN` before pruning with the number of records targeted, `WARN` when pruning begins, and `INFO` when pruning/recovery completes.
  - Change the prune helpers so they return deleted row counts or another structured summary that can be logged, instead of only returning success/failure.
- [x] **6.2 Add Database Maintenance Logging**: Log SQLite maintenance operations with timing and outcome.
  - Wrap `VACUUM` execution in both database packages with `DEBUG` start/end logs and `ERROR` on failure.
  - If checkpoint or WAL maintenance is added, log `DEBUG` around those calls with `duration` measured with `time.Since(...)`.
  - Keep maintenance logs distinct from request-handler logs so operators can separate runtime traffic from database housekeeping.
- [x] **6.3 Test Storage Cap Logging**: Add unit tests for pruning and vacuum logging.
  - In `kopds/internal/database/storage_cap_test.go` and `kosync/internal/database/storage_cap_test.go`, verify `DEBUG` cap-check logs, `WARN` prune-trigger logs, and completion logs after pruning.
  - Confirm that identical fields are emitted across both projects for equivalent cap-pruning scenarios.

### Phase 7: Integration Testing and Documentation

**Objective**: Verify logging works correctly in real scenarios and document for operators.

- [x] **7.1 Create Logging Test Matrix**: Add an explicit matrix of logging scenarios and expected outputs.
  - Update `logging-plan.md` with a table covering startup, CLI, authentication, progress operations, storage-cap enforcement, and shutdown for both text and JSON log formats.
  - Use the existing `internal/*_test.go` files and integration scripts as the execution targets for those matrix entries.

#### 7.1 Logging Test Matrix

| Scenario | Project | Command / Test | Expected Log Level | Key Fields | Notes |
|---|---|---|---|---|---|
| App startup | KOPDS | `go test ./cmd/kopds` / manual startup | INFO | `app_name`, `port`, `database_path`, `log_level`, `json_log` | Should appear before any request or CLI operation logs |
| App startup | KOSYNC | `go test ./cmd/kosync` / manual startup | INFO | `app_name`, `port`, `database_path`, `log_level`, `json_log` | Same format as KOPDS |
| CLI create user success | KOPDS | `cmd/kopds/main_test.go` | INFO | `username`, `operation`, `source`, `status` | `source=CLI` |
| CLI create user failure | KOPDS | `cmd/kopds/main_test.go` | WARN | `username`, `operation`, `source`, `reason` | duplicate user or invalid input |
| CLI change password success | KOPDS | `cmd/kopds/main_test.go` | INFO | `username`, `operation`, `source` | |
| CLI change password failure | KOPDS | manual test (invalid user) | WARN | `username`, `operation`, `source`, `reason` | |
| CLI delete user success | KOPDS | `cmd/kopds/main_test.go` | INFO | `username`, `operation`, `source` | |
| CLI delete user failure | KOPDS | manual test (invalid user) | WARN | `username`, `operation`, `source`, `reason` | |
| CLI create user success | KOSYNC | `cmd/kosync/main_test.go` | INFO | `username`, `operation`, `source` | |
| CLI create user failure | KOSYNC | `cmd/kosync/main_test.go` | WARN | `username`, `operation`, `source`, `reason` | |
| CLI change password success | KOSYNC | `cmd/kosync/main_test.go` | INFO | `username`, `operation`, `source` | |
| CLI delete user success | KOSYNC | `cmd/kosync/main_test.go` | INFO | `username`, `operation`, `source` | |
| Handler request success | KOPDS | `internal/api` tests or integration | INFO | `method`, `path`, `status_code`, `duration`, `request_id` | 2xx/3xx response |
| Handler request client error | KOPDS | `internal/api` tests | WARN | `method`, `path`, `status_code`, `duration`, `error_reason` | 4xx response |
| Handler request server error | KOPDS | `internal/api` tests | ERROR | `method`, `path`, `status_code`, `duration`, `error_detail` | 5xx response |
| Handler request success | KOSYNC | `internal/api` tests or integration | INFO | `method`, `path`, `status_code`, `duration`, `request_id` | Same structure as KOPDS |
| Storage cap prune | KOPDS | `internal/database/storage_cap_test.go` | WARN / INFO | `database_path`, `rows_deleted`, `freed_mb`, `cap_mb` | Prune summary logged |
| Storage cap prune | KOSYNC | `internal/database/storage_cap_test.go` | WARN / INFO | `database_path`, `rows_deleted`, `freed_mb`, `cap_mb` | Same fields as KOPDS |
| Shutdown signal | KOPDS | manual signal test | INFO | `signal_name`, `reason`, `uptime` | graceful shutdown |
| Shutdown signal | KOSYNC | manual signal test | INFO | `signal_name`, `reason`, `uptime` | graceful shutdown |
| Auth failure | KOSYNC | `internal/api` auth tests | WARN | `username`, `remote_addr`, `reason` | |
| Progress update success | KOSYNC | `internal/api` tests | INFO | `username`, `document`, `percentage`, `status` | |
| Progress update skipped | KOSYNC | `internal/database` tests | DEBUG | `username`, `document`, `reason` | Older timestamp |
| Book scan started | KOPDS | `internal/scanner` tests | INFO | `library_path`, `scan_type` | |
| Book scan completed | KOPDS | `internal/scanner` tests | INFO | `library_path`, `books_found`, `duration` | |
| Book scan failed | KOPDS | `internal/scanner` tests | ERROR | `library_path`, `error_detail` | |
| Config loaded | KOPDS | `go test ./cmd/kopds` | INFO | `config_source`, `database_path` | |
| Config loaded | KOSYNC | `go test ./cmd/kosync` | INFO | `config_source`, `database_path` | |

- [ ] **7.2 Run Full Logging Integration Tests**: Execute real logging scenarios in both projects.
  - Run `kopds/test/integration_test.sh` and `kosync/test/integration_test.sh` with `LOG_LEVEL=INFO` and `LOG_LEVEL=DEBUG` to capture operational logs.
  - Validate that request middleware logs, handler logs, CLI logs, and maintenance logs appear at the expected level and field set.
- [ ] **7.3 Test Docker Logging**: Exercise containerized logging paths.
  - Use `build/Dockerfile`, `deploy/docker-compose.yml`, and the existing container entrypoints to confirm `docker logs` shows startup, request, and failure logs with the configured log level.
  - Capture both plain text and JSON modes if the container configuration allows them.
- [ ] **7.4 Update READMEs**: Document operator-facing logging behavior.
  - Update `kopds/README.md` and `kosync/README.md` with examples of healthy vs. unhealthy log patterns, common `LOG_LEVEL` usage, and how to interpret `WARN`/`ERROR` entries.
  - Add troubleshooting notes for missing logs, storage-cap pruning, and shutdown failures.
- [ ] **7.5 Update GEMINI Files**: Document the logging strategy in both project guides.
  - Update `kopds/GEMINI.md` and `kosync/GEMINI.md` so they describe the `log/slog` strategy, expected log levels, and operator guidance.

### Phase 8: Final Verification

**Objective**: Confirm logging is comprehensive, uniform, and production-ready.

- [ ] **8.1 Logging Coverage Audit**: Re-run the logging matrix and ensure every operation class has coverage.
  - Compare event coverage from middleware, CLI, handlers, and maintenance paths across both projects.
  - Confirm there are no missing business events such as startup, shutdown, auth failure, progress update, cache miss, or storage-cap enforcement.
- [ ] **8.2 Uniformity Audit**: Compare equivalent log output across both projects.
  - Check that shared operations emit the same field names and severity levels in both `kopds` and `kosync`.
  - Record any intentional differences, especially where KOPDS has book/image operations that KOSYNC does not.
- [ ] **8.3 Run All Tests**: Execute `go test ./...` in both projects and confirm logging-related tests pass.
  - Run `go test ./...` in `kopds/` and `kosync/` from the repository root or each project directory.
  - Treat logging regressions as blocking failures.
- [ ] **8.4 Performance Check**: Verify logging overhead remains acceptable.
  - Measure request timings with middleware enabled and compare them against baseline runs to ensure added structured logging does not introduce significant latency.
  - Focus on the request completion path in `LoggingMiddleware` and repeated JSON/text handler logs.
- [ ] **8.5 Update AGENTS.md Files**: Update agent guidance to reflect `log/slog`.
  - Change the logging references in `kopds/AGENTS.md` and `kosync/AGENTS.md` from `rs/zerolog` to `log/slog`.
  - Add a short note that startup, request, CLI, and maintenance logs should follow the shared `log/slog` field conventions.
- [ ] **8.6 Final Documentation**: Update the broader planning docs.
  - Update [uniformity-plan.md](uniformity-plan.md) to reflect logging standardization completion.
  - Update [todo.md](todo.md) to mark all logging-related phases complete once the work is finished.

## Logging Implementation Guidelines

### Consistent Field Names

Use identical field names across both projects:
- HTTP request: `method`, `path`, `remote_addr`, `user` (if auth'd), `status_code`, `duration` (as `slog.Duration`)
- User operations: `username`, `operation`, `reason`, `source` ("CLI" or "API")
- Data operations: `document` (KOSYNC), `book` (KOPDS), `percentage` (KOSYNC), etc.
- Errors: `error` (error message), `error_detail` (full error stack or context)
- Timing: Use `slog.Duration` type with field name `duration` (slog handles formatting for both text and JSON output)
- Context keys: define explicit context key constants such as `ContextKeyRequestID`, `ContextKeyUser`, and `ContextKeyLogger` in a shared internal package or identical file in both projects to avoid magic string bugs.


### Log Message Style

- **INFO**: Business event, typically success or significant milestone (e.g., "User 'alice' created successfully", "Progress updated for document 'book.pdf'")
- **DEBUG**: Detailed diagnostic information (e.g., "Progress comparison: existing_ts=1000, incoming_ts=1000, skipping update")
- **WARN**: Problem detected but handled gracefully (e.g., "Registration attempt while disabled", "Storage pruning triggered")
- **ERROR**: Operational failure requiring investigation (e.g., "Failed to write progress: database locked", "Invalid config file")

### Logging with Context

Use structured logging with named fields for maximum clarity in both text and JSON output:

```go
// Good: structured with field names
slog.Info("user created successfully", "username", "alice", "source", "CLI")

// Good: includes diagnostic context
slog.Debug("progress comparison", "username", "bob", "document", "book1", 
	"existing_timestamp", 1000, "incoming_timestamp", 1000, "action", "skip")

// Avoid: vague or unstructured
slog.Info("operation completed")
```

## Success Criteria

1. Every HTTP request/response completion logs at INFO level (2xx/3xx) with status code and duration; 4xx logs at WARN, 5xx at ERROR; DEBUG level provides detailed diagnostic information
2. Every CLI operation logs success at INFO level and failure at WARN level
3. Every data operation (user create, progress update, book scan) logs success at INFO level and failure at ERROR level
4. Logs are identical in format and content for equivalent operations across KOPDS and KOSYNC
5. DEBUG level provides enough detail for troubleshooting without being verbose
6. No operation is logged multiple times at different layers (e.g., both handler and middleware)
7. All unit and integration tests pass
8. Documentation clearly explains what logs indicate healthy vs. problematic operation
9. Operators can use `LOG_LEVEL=INFO` in Docker and see all significant operational events
10. Operators can use `LOG_LEVEL=DEBUG` and get diagnostic details for troubleshooting

