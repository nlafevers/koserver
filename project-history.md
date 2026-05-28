# KOSERVER Project History

Status: May 28, 2026

This document condenses the work recorded in `final-verification-checklist.md`, `harmonization-plan.md`, `logging-plan.md`, `uniformity-plan.md`, and `todo.md`, with additional context from the Git histories of KOSERVER, KOPDS, and KOSYNC. It is a human-readable history, not a new implementation plan.

## Purpose

KOSERVER is the documentation root for two related Go applications that serve KOReader clients:

- KOPDS: an OPDS server for browsing, searching, and downloading books from a Calibre-style library.
- KOSYNC: a KOReader reading-progress synchronization server.

The central project goal became to make the two applications feel like twins operationally. Where they do the same job, their code should use the same names, the same flow, and ideally copy-paste-identical implementations. Where their domains differ, the surrounding scaffolding should still look the same, with only the domain-specific pieces changing.

## Starting Point

The two projects began with complementary strengths.

KOPDS already had the stronger application structure. It used the standard Go `cmd/` and `internal/` layout, Viper-based configuration, clear domain and repository boundaries, OPDS-specific services, Docker support, and richer catalog behavior.

KOSYNC was more advanced in several operational areas. It had more complete CLI user-management commands, password-masked interactive input, automation-friendly stdin password support, strict database file permissions, storage-cap enforcement, and detailed deployment and security documentation.

The early work therefore had two tracks: bring KOPDS up to KOSYNC's operational standard, and bring KOSYNC up to KOPDS's architectural standard.

## Early Project Buildout

KOSYNC's history shows the app being built from the ground up: Go module initialization, environment configuration, structured logging, user and progress data models, secure SQLite initialization, bcrypt password hashing, timestamp-based progress conflict handling, authentication middleware, KOReader-compatible API handlers, integration tests, deployment documentation, and CLI user management.

KOPDS's history shows a longer-running OPDS application being hardened and matured: catalog navigation, OPDS search, cover image handling, file delivery safeguards, Docker publishing, deployment recommendations, admin hardening, database deadlock fixes, SQLite PRAGMA alignment, storage-cap enforcement, database permission enforcement, and black-box integration testing.

These histories explain the first harmonization strategy: each project had already solved different problems well, so the best path was to copy mature behavior across rather than invent new patterns.

## Harmonization Phase

The first major cross-project plan was broad harmonization. Its goal was feature parity, security parity, and architectural maturity.

KOPDS adopted KOSYNC-style operational capabilities. Its CLI was standardized around commands such as `create-user`, `delete-user`, and `change-password`. It gained interactive and stdin password flows, unified file logging, migration from `zerolog` to the standard library `log/slog`, stricter SQLite permissions, storage-cap enforcement, aligned SQLite PRAGMAs, and an integration test script.

KOSYNC adopted KOPDS-style structure. It gained `cmd/kosync`, `internal/` packages, Viper configuration, an extracted database layer, extracted API handlers and middleware, a cleaner main entrypoint, default config files, a default data directory, Dockerfile support, and docker-compose deployment.

This phase also exposed lingering inconsistencies. The most visible ones were KOSYNC requiring an existing database before CLI user commands could run, and KOSYNC printing extra `Using database:` and `Using log:` lines that KOPDS did not print. These were later resolved during the stricter uniformity phase.

## Uniformity Phase

After broad harmonization, the goal tightened from "similar enough" to maximum practical uniformity. The rule became: if a function or module does the same job in both projects, it should have the same name and the same implementation wherever possible.

Several standardization choices guided the work:

- KOPDS became the default structural baseline because its package layout was more mature.
- KOSYNC's bcrypt cost of `12` became the password-hashing baseline.
- SQLite directories should use `0750` permissions and database files should use `0600`.
- Identical duplicated code inside each repository was preferred over a shared module, because KOPDS and KOSYNC are separate Git repositories.
- KOSYNC kept backwards-compatible aliases such as `db_path` and `KOSYNC_DB_PATH`, while preferring the uniform `database_path` and `KOSYNC_DATABASE_PATH` names.

The uniformity work touched most shared infrastructure. Password hashing and checking were standardized as `HashPassword` and `CheckPassword`. Logger construction became `logger.New(level string, json bool, logPath string)` in both projects. Configuration names, path resolution, config examples, and environment variable behavior were aligned. Both projects gained identical SQLite open helpers with directory creation, file permission enforcement, WAL mode, busy timeout, `SetMaxOpenConns(1)`, and ping behavior.

Storage-cap enforcement was reshaped so the common control flow was identical. Cap-disabled behavior, file-size checks, over-cap pruning, and `VACUUM` handling were shared. Only the pruning SQL remained project-specific: KOPDS prunes catalog-related data, while KOSYNC prunes progress rows.

The CLI became one of the most important uniformity wins. Dispatch, usage text, password reading, stdin password handling, database creation behavior, command output, and user-management semantics were aligned. Both projects now create and migrate their configured database when needed for CLI commands. Both omit noisy config-path status lines. Both treat `create-user` as a create-only operation that fails if the user already exists.

Deployment and tests were also aligned. Shared dependencies were brought closer together, Dockerfile and compose patterns were made similar, and both integration scripts were updated to build binaries, create temporary databases through the CLI, assert CLI output, run servers with explicit environment paths, and clean up their artifacts.

## Routing and HTTP Shape

A later cleanup brought KOPDS routing in line with KOSYNC by replacing chi routing with the standard library `net/http.ServeMux`. This removed a framework-level difference from the shared HTTP surface and made middleware composition easier to compare between projects.

The route behavior still remains intentionally domain-specific. KOPDS serves OPDS catalog, author, series, tag, search, cover, and download endpoints. KOSYNC serves KOReader sync endpoints for registration, authentication, progress retrieval, and progress update. The shared goal is not identical routes, but identical structure around routing, middleware, logging, and error handling where the responsibilities match.

## User Management

User management became a major shared contract.

Both apps now support the same CLI command family: `create-user`, `delete-user`, and `change-password`. Passwords can be entered interactively with masking or supplied through stdin for automation. The output style and error behavior were standardized.

The most important semantic change was preventing accidental overwrites. Both storage layers gained `CreateUserIfNotExists`, and both CLIs now fail with a non-zero exit code if `create-user` is run for an existing user. Documentation now points operators to `change-password` when they intend to update credentials.

Tests were expanded in both projects to verify success paths, duplicate-create failures, delete behavior, password-change behavior, missing-user failures, stdout and stderr content, and database auto-creation.

## Logging Refactor

Once structure and CLI behavior were aligned, the next major gap was logging coverage. Both apps had the same logger constructor, but they did not log the same kinds of events at the same levels.

The logging refactor made logging comprehensive and uniform. Both projects now use `log/slog` with the same text/JSON behavior, optional file teeing, and default logger registration. The shared logging strategy defines INFO for business events, DEBUG for diagnostics, WARN for handled problems, and ERROR for operational failures.

Both projects gained matching HTTP logging middleware. The middleware assigns a request ID, stores a request-scoped logger in context, records the authenticated user when available, captures status code and response size, measures request duration, and logs request completion at the appropriate level. Successful responses log at INFO, client errors at WARN, and server errors at ERROR, with DEBUG logs available for extra diagnostics.

CLI logging was standardized through helper functions in `internal/logger/cli.go`. User-management success and failure events now include stable fields such as `username`, `operation`, `reason`, and `source="CLI"`.

Handler, service, and repository logging were expanded. KOPDS logs OPDS feed responses, catalog operations, cover handling, scanner activity, image cache behavior, authentication results, repository queries, and database errors. KOSYNC logs authentication, registration, progress reads, progress updates, skipped stale updates, repository queries, and database errors. Both projects use request-scoped loggers where appropriate, so request IDs and users can be followed through the stack.

Startup, shutdown, and database initialization logs were also aligned. Both applications log startup with fields such as `app_name`, `port`, `database_path`, `log_level`, `json_log`, and `log_path`. Both log signal receipt, graceful shutdown, shutdown failures, and database initialization with stable field names.

Storage maintenance logging was added for cap checks, pruning, `VACUUM`, and maintenance failures. This made disk-protection behavior visible to operators instead of silently deleting records.

The logging work ended with a logging matrix, unit tests, integration tests, Docker logging checks, README updates, and AGENTS/GEMINI guidance. The result is that operators can run with `LOG_LEVEL=INFO` for normal operations or `LOG_LEVEL=DEBUG` for troubleshooting and see comparable output from both apps.

## Final Verification

The final verification work closed the remaining testing gaps.

Both projects were verified to create databases and parent directories on first run, create database files with `0600` permissions, support all CLI user-management commands in interactive and stdin modes, create log files when configured, and log CLI, startup, and shutdown events consistently.

KOPDS verification covered the Calibre scanner, OPDS navigation, FTS5 search, book detail responses, cover resizing, EPUB delivery, Basic Auth rejection without credentials, graceful shutdown, config-file support, and environment overrides.

KOSYNC verification covered strict KOReader headers, user registration, authentication, progress retrieval, progress update, and timestamp conflict behavior. A later fix made `HandleUpdateProgress` respect a client-supplied timestamp when non-zero, using server time only when the timestamp is missing. Tests now verify that newer progress wins and equal or older progress is ignored.

Storage-cap testing was made concrete in both projects. KOPDS now has an integration test that bloats a real SQLite `sync_state` table, enforces a 1 MB cap, verifies rows were deleted, and confirms the physical database file shrank after `VACUUM`. KOSYNC has the matching test against its `progress` table.

The KOSERVER documentation repository tracked these waves of work through planning documents, progress updates, condensed todo entries, and final completion checkboxes. Its commit history shows the planning arc clearly: initial documentation, testing tasks, routing completion, user-management completion, logging-plan expansion and completion, timestamp completion, and final testing completion.

## Current Shared Baseline

As of this history, KOPDS and KOSYNC share a common operational baseline:

- Go applications with `cmd/` and `internal/` structure.
- SQLite persistence using `modernc.org/sqlite`.
- Viper-based configuration from config files and environment variables.
- `database_path`, logging, and storage-cap settings with aligned naming.
- Database directory creation and strict database file permissions.
- Bcrypt password hashing with aligned helper names and cost.
- Standard-library `net/http.ServeMux` routing.
- Structured `log/slog` logging with matching constructor behavior.
- Request logging middleware with request IDs and stable fields.
- CLI user management with matching commands, messages, stdin support, and duplicate-user behavior.
- Storage-cap enforcement with shared control flow and visible maintenance logs.
- Docker and compose deployment patterns.
- Unit and integration tests covering shared behavior.

## Intentional Differences

Some differences are correct and should remain.

KOPDS owns OPDS catalog behavior, Calibre scanning, FTS5 book search, image cache and cover resizing, download links, book repositories, and OPDS feed generation. KOSYNC owns KOReader sync behavior, header authentication, registration, progress storage, and timestamp conflict resolution.

The schemas and pruning SQL differ because the stored domains differ. KOPDS stores catalog and sync-state data for books. KOSYNC stores user and reading-progress records. Ports, binary names, environment-variable prefixes, and a few config defaults also remain app-specific.

The uniformity standard is therefore not "make the apps identical." It is "make the shared scaffolding identical, and make domain differences explicit."

## Maintenance Lesson

The project moved through three useful stages:

1. Harmonization: copy mature capabilities from one project to the other.
2. Uniformity: make equivalent code names and implementations identical wherever practical.
3. Verification: prove the shared behavior with tests, integration scripts, Docker checks, and operator-facing documentation.

Future work should preserve that discipline. When adding a shared capability to one project, check the other project immediately. When equivalent code starts to drift, prefer exact copy-paste alignment unless there is a real domain reason. When a difference is intentional, document it so future maintenance does not mistake it for accidental inconsistency.
