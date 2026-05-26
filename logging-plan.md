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
| **HTTP Requests** | Request received | Business | INFO | method, path, remote_addr, user (if auth'd) |
| **HTTP Requests** | Successful response | Business | DEBUG | method, path, status_code, response_time_ms |
| **HTTP Requests** | Client error (4xx) | Problem | WARN | method, path, status_code, error_reason |
| **HTTP Requests** | Server error (5xx) | Problem | ERROR | method, path, status_code, error_detail |
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
| **Data Operations (KOPDS)** | Book scan completed | Success | INFO | library_path, books_found, scan_time_ms |
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

- [ ] **1.1 Design Logging Middleware**: Create `LoggingMiddleware` for stdlib `net/http.ServeMux` that logs all requests at DEBUG level with method, path, remote_addr, user (if present); logs responses at DEBUG (2xx/3xx), WARN (4xx), ERROR (5xx) with status code and response time.
- [ ] **1.2 Implement Logging Middleware in KOPDS**: Add `internal/api/logging.go` with `LoggingMiddleware` implementation; apply middleware to all routes in `main.go`.
- [ ] **1.3 Implement Logging Middleware in KOSYNC**: Add `internal/middleware/logging.go` (or extend existing) with identical `LoggingMiddleware`; apply middleware to all routes in `main.go`.
- [ ] **1.4 Test Logging Middleware**: Add unit tests verifying correct log output for various HTTP status codes and request types; verify response times are captured accurately.

### Phase 2: Standardize CLI Operation Logging

**Objective**: Ensure all CLI operations (create-user, delete-user, change-password) log at INFO level on success and WARN level on failure.

- [ ] **2.1 Create CLI Logging Helpers**: Add `internal/api/cli_logger.go` (or similar) with standardized functions: `LogCLISuccess(operation, username string)`, `LogCLIFailure(operation, username, reason string)`, `LogCLIInput(operation string)`.
- [ ] **2.2 Implement KOPDS CLI Logging**: Refactor `cmd/kopds/main.go` user-management functions to call logging helpers; ensure all success and failure paths log at appropriate levels.
- [ ] **2.3 Implement KOSYNC CLI Logging**: Refactor `cmd/kosync/main.go` user-management functions to call identical logging helpers; ensure all success and failure paths log at appropriate levels.
- [ ] **2.4 Test CLI Logging**: Add tests verifying exact log output (level and message) for success/failure of create, delete, and change-password operations; verify source="CLI" tag is present.

### Phase 3: Standardize API Operation Logging (Handlers)

**Objective**: Ensure all handler operations log success at INFO level and failures at appropriate error/warn levels with complete diagnostic information.

- [ ] **3.1 Review Current Handler Logging**: Audit all handlers in both projects; document current gaps (e.g., KOPDS missing progress operation logging; KOSYNC missing successful GET logging).
- [ ] **3.2 Add Uniform Handler Logging in KOPDS**: Add INFO logging for successful OPDS catalog responses, book catalog retrieval, image cache hits/misses, and basic auth successes; add DEBUG logging for detailed diagnostic info.
- [ ] **3.3 Add Uniform Handler Logging in KOSYNC**: Add INFO logging for successful progress GET and PUT operations; add DEBUG logging for detailed diagnostic info about updates, skips, and comparisons.
- [ ] **3.4 Standardize Error Context in Handlers**: Ensure all handler error paths include username (if auth'd), request path, and error detail; use consistent field names across both projects.
- [ ] **3.5 Test Handler Logging**: Add integration tests for each handler verifying correct log output for success, client errors, and server errors.

### Phase 4: Standardize Startup/Shutdown Logging

**Objective**: Ensure consistent and informative logging for application lifecycle events.

- [ ] **4.1 Unify Startup Logging**: Both projects should log at INFO level: app name, version, selected port, database path, log level, and notable config settings; ensure output is identical in format.
- [ ] **4.2 Unify Shutdown Logging**: Both projects should log graceful shutdown and errors at appropriate levels; include uptime and shutdown reason.
- [ ] **4.3 Unify Database Initialization Logging**: Both projects should log successful database initialization, schema migration status, and storage cap enforcement at INFO level.
- [ ] **4.4 Test Startup/Shutdown Logging**: Verify expected log entries appear when starting normally, starting with invalid config, shutting down cleanly, and encountering errors.

### Phase 5: Standardize Service/Repository Layer Logging

**Objective**: Ensure database operations and service-layer actions log appropriately without duplicating HTTP-level logging.

- [ ] **5.1 Review Service Logging Strategy**: Decide whether repository/service layer operations log individually (detailed) or only at API handler level (cleaner); document the choice.
- [ ] **5.2 Implement Service Logging in KOPDS**: Add DEBUG-level logging for database queries (book lookups, user queries); add ERROR logging for query failures.
- [ ] **5.3 Implement Service Logging in KOSYNC**: Add DEBUG-level logging for database queries (progress lookups, user queries); add ERROR logging for query failures.
- [ ] **5.4 Test Service Logging**: Verify service-layer logging complements (not duplicates) handler-level logging; verify DEBUG logs show query details and ERROR logs show failure reasons.

### Phase 6: Storage Cap and Maintenance Logging

**Objective**: Ensure storage cap enforcement and database maintenance operations log appropriately for operational visibility.

- [ ] **6.1 Add Storage Cap Logging**: Both projects should log at DEBUG level when checking cap, at WARN level when pruning is triggered, and at INFO level when pruning completes; include freed space and deleted row counts.
- [ ] **6.2 Add Database Maintenance Logging**: Both projects should log VACUUM operations, WAL checkpoints, and maintenance timing at DEBUG level.
- [ ] **6.3 Test Storage Cap Logging**: Verify expected log entries for cap checks, pruning triggers, and completions; verify log content is identical between projects.

### Phase 7: Integration Testing and Documentation

**Objective**: Verify logging works correctly in real scenarios and document for operators.

- [ ] **7.1 Create Logging Test Matrix**: Document test cases covering all operation classes and log levels; ensure tests run with both text and JSON logging.
- [ ] **7.2 Run Full Logging Integration Tests**: Execute integration scripts and manual testing with LOG_LEVEL=DEBUG and LOG_LEVEL=INFO; verify all expected logs appear with correct level, content, and format.
- [ ] **7.3 Test Docker Logging**: Start both applications in Docker containers with various LOG_LEVEL settings; verify logs appear in `docker logs` output correctly.
- [ ] **7.4 Update READMEs**: Document expected log output for common operations; add troubleshooting section explaining what logs indicate healthy vs. problematic operation.
- [ ] **7.5 Update GEMINI Files**: Document the logging strategy, expected log levels for all operations, and interpretation guidance.

### Phase 8: Final Verification

**Objective**: Confirm logging is comprehensive, uniform, and production-ready.

- [ ] **8.1 Logging Coverage Audit**: Re-run the logging matrix; verify every operation class has at least one test case; verify no significant operations are missing logs.
- [ ] **8.2 Uniformity Audit**: Compare log output from equivalent operations in both projects; document any intentional differences and justify them.
- [ ] **8.3 Run All Tests**: Execute `go test ./...` in both projects; ensure all logging tests pass.
- [ ] **8.4 Performance Check**: Verify logging does not introduce measurable performance degradation; check logging middleware response times.
- [ ] **8.5 Final Documentation**: Update [uniformity-plan.md](uniformity-plan.md) to reflect logging standardization completion; update [todo.md](todo.md) to mark all logging TODOs as complete.

## Logging Implementation Guidelines

### Consistent Field Names

Use identical field names across both projects:
- HTTP request: `method`, `path`, `remote_addr`, `user` (if auth'd), `status_code`, `response_time_ms`
- User operations: `username`, `operation`, `reason`, `source` ("CLI" or "API")
- Data operations: `document` (KOSYNC), `book` (KOPDS), `percentage` (KOSYNC), etc.
- Errors: `error` (error message), `error_detail` (full error stack or context)

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

1. Every HTTP request/response logs at appropriate level (DEBUG for success, WARN/ERROR for failures)
2. Every CLI operation logs success at INFO level and failure at WARN level
3. Every data operation (user create, progress update, book scan) logs success at INFO level and failure at ERROR level
4. Logs are identical in format and content for equivalent operations across KOPDS and KOSYNC
5. DEBUG level provides enough detail for troubleshooting without being verbose
6. No operation is logged multiple times at different layers (e.g., both handler and middleware)
7. All unit and integration tests pass
8. Documentation clearly explains what logs indicate healthy vs. problematic operation
9. Operators can use `LOG_LEVEL=INFO` in Docker and see all significant operational events
10. Operators can use `LOG_LEVEL=DEBUG` and get diagnostic details for troubleshooting

