# KOSERVER To Do List

This file tracks cross-project tasks and short roadmaps that do not warrant a dedicated roadmap file. Completed items are kept for history.

## Git Repository Structure

KOSERVER is documentation only. KOPDS and KOSYNC are separate Git repositories inside this workspace. Code changes inside `kopds/` must be committed in the KOPDS repository. Code changes inside `kosync/` must be committed in the KOSYNC repository. Changes to this file must be committed in the KOSERVER repository. See `AGENTS.md` for full policy.

## How To Use This File

Implement one incomplete step at a time, in order within each task group. When running Go commands, prefer a writable cache under `/tmp`:

```bash
GOCACHE=/tmp/kopds-gocache go test ./...
GOCACHE=/tmp/kosync-gocache go test ./...
```

---

## Completed

- [x] **TDL-001 Storage Lifecycle Wrappers**: Supersedes UR2-3.1. Standardized on KOPDS separation of concerns: `OpenSQLite(path, allowCreate)`, `Migrate(db)`, then inject dependencies (`NewStorage`, repositories). Removed KOSYNC `InitDB` and `Storage.Close()`. Both apps use `NewSQLite(path, allowCreate)` as a thin wrapper. KOSYNC server and CLI wired to explicit open/migrate/inject; KOPDS gained `allowCreate` on `NewSQLite`.

---

## TDL-002/003: Quiet CLI Stdout Logging

**Goal:** CLI user-management commands (`create-user`, `delete-user`, `change-password`) currently print structured `slog` entries to stderr even when a log file is configured, cluttering the terminal. Only the final human-readable summary line (e.g. `"Failed to delete user: user not found."`) should appear on stdout. Structured entries should go to the log file only, or be silently discarded when no log file is configured. Server-mode logging is unchanged.

**Root cause:** `logger.New()` always constructs `io.MultiWriter(os.Stderr, file)` when a log path is given, so every `slog` entry goes to both destinations. The CLI helpers call `slog.Default()`, which points at that multi-writer.

**Fix:** Add `logger.NewCLI()` — identical level parsing and handler construction as `New`, but writes to the log file only (or `io.Discard` when no path is set). Switch `main()` in each app to call `NewCLI` before `runCLI` and `New` before `runServer`.

- [ ] **TDL-002** Add `logger.NewCLI` to both apps
  - **Repos:** kopds, kosync
  - **Read:** kopds/internal/logger/logger.go, kosync/internal/logger/logger.go
  - **Edit:** kopds/internal/logger/logger.go, kopds/internal/logger/logger_test.go, kosync/internal/logger/logger.go, kosync/internal/logger/logger_test.go
  - **Instructions:** In `internal/logger/logger.go` in **both** apps, add the following function immediately after `New`. The implementation must be identical in both:

    ```go
    // NewCLI initializes a logger for CLI mode. Structured entries go to the
    // log file only; if no log path is configured they are discarded. This
    // keeps CLI stdout clean: only fmt.Fprintf print statements reach the
    // terminal.
    func NewCLI(level string, json bool, logPath string) *slog.Logger {
        var slogLevel slog.Level
        switch strings.ToLower(level) {
        case "debug":
            slogLevel = slog.LevelDebug
        case "info":
            slogLevel = slog.LevelInfo
        case "warn":
            slogLevel = slog.LevelWarn
        case "error":
            slogLevel = slog.LevelError
        default:
            slogLevel = slog.LevelInfo
        }

        var output io.Writer = io.Discard

        if logPath != "" {
            file, err := os.OpenFile(logPath, os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
            if err != nil {
                fmt.Fprintf(os.Stderr, "failed to open log file: %v\n", err)
                // output stays io.Discard
            } else {
                output = file
            }
        }

        var handler slog.Handler
        if json {
            handler = slog.NewJSONHandler(output, &slog.HandlerOptions{Level: slogLevel})
        } else {
            handler = slog.NewTextHandler(output, &slog.HandlerOptions{Level: slogLevel})
        }

        l := slog.New(handler)
        slog.SetDefault(l)
        return l
    }
    ```

    Do NOT modify the existing `New` function.

    In `internal/logger/logger_test.go` in **both** apps (create the file — it does not exist yet), add package-level tests for `NewCLI`:

    ```go
    package logger

    import (
        "os"
        "strings"
        "testing"
    )

    func TestNewCLI(t *testing.T) {
        t.Run("NoLogPath returns non-nil logger", func(t *testing.T) {
            l := NewCLI("info", false, "")
            if l == nil {
                t.Fatal("expected non-nil logger")
            }
            // Calling Log must not panic.
            l.Info("discarded message")
        })

        t.Run("WithLogPath writes to file and not to stderr", func(t *testing.T) {
            tmp := t.TempDir()
            logFile := tmp + "/test.log"

            l := NewCLI("info", false, logFile)
            if l == nil {
                t.Fatal("expected non-nil logger")
            }
            l.Info("hello from cli", "key", "val")

            data, err := os.ReadFile(logFile)
            if err != nil {
                t.Fatalf("failed to read log file: %v", err)
            }
            if !strings.Contains(string(data), "hello from cli") {
                t.Errorf("log file should contain message, got: %s", data)
            }
        })

        t.Run("WithLogPath JSON format writes JSON", func(t *testing.T) {
            tmp := t.TempDir()
            logFile := tmp + "/test.json"

            l := NewCLI("info", true, logFile)
            l.Info("json message")

            data, err := os.ReadFile(logFile)
            if err != nil {
                t.Fatalf("failed to read log file: %v", err)
            }
            if !strings.Contains(string(data), `"msg":"json message"`) {
                t.Errorf("expected JSON log entry, got: %s", data)
            }
        })
    }
    ```

  - **Verify:** `cd /home/nathan/koserver/kopds && gofmt -l internal/logger/ && GOCACHE=/tmp/kopds-gocache go build ./... && GOCACHE=/tmp/kopds-gocache go test ./internal/logger/... && cd ../kosync && gofmt -l internal/logger/ && GOCACHE=/tmp/kosync-gocache go build ./... && GOCACHE=/tmp/kosync-gocache go test ./internal/logger/...`
  - **Done when:** `NewCLI` is present and identical in both logger packages, `logger_test.go` exists and all three subtests pass in both apps, gofmt prints nothing, and builds pass; one commit per repo

- [ ] **TDL-003** Wire `logger.NewCLI` into CLI mode in both apps
  - **Repos:** kopds, kosync
  - **Read:** kopds/cmd/kopds/main.go, kosync/cmd/kosync/main.go
  - **Edit:** kopds/cmd/kopds/main.go, kosync/cmd/kosync/main.go
  - **Instructions:** In `kopds/cmd/kopds/main.go` and `kosync/cmd/kosync/main.go`, restructure `main()` so that CLI mode initialises with `NewCLI` and server mode initialises with `New`.

    Currently both files call `logger.New(...)` unconditionally then branch on `os.Args`:

    ```go
    log := logger.New(cfg.LogLevel, cfg.JSONLog, cfg.LogPath)
    if len(os.Args) > 1 {
        runCLI(cfg)
        return
    }
    runServer(cfg, log)
    ```

    Replace with:

    ```go
    if len(os.Args) > 1 {
        logger.NewCLI(cfg.LogLevel, cfg.JSONLog, cfg.LogPath)
        runCLI(cfg)
        return
    }
    log := logger.New(cfg.LogLevel, cfg.JSONLog, cfg.LogPath)
    runServer(cfg, log)
    ```

    No other files change. `runCLI` and its helpers (`createUser`, `deleteUser`, `changePassword`, `openCLIStorage`) already use `slog.Default()` via the nil-logger fallback in `LogCLISuccess`/`LogCLIFailure`. With `slog.Default()` now pointing at the file-only (or discard) handler set by `NewCLI`, those calls will automatically go to the log file only.

    Apply the identical restructuring to both `main.go` files.

  - **Verify:**
    ```bash
    cd /home/nathan/koserver/kopds && gofmt -l cmd/kopds/ && GOCACHE=/tmp/kopds-gocache go build ./... && GOCACHE=/tmp/kopds-gocache go test ./cmd/kopds/...
    cd /home/nathan/koserver/kosync && gofmt -l cmd/kosync/ && GOCACHE=/tmp/kosync-gocache go build ./... && GOCACHE=/tmp/kosync-gocache go test ./cmd/kosync/...
    ```
  - **Done when:** both apps build and tests pass; CLI commands produce only the final human-readable summary line on stdout; slog entries go to the log file when a path is configured (or are silently discarded when not); server mode logging is unchanged; gofmt prints nothing; one commit per repo

Acceptance criteria for TDL-002/003:
- `logger.NewCLI` exists and is identical in both apps.
- `logger_test.go` covers the no-path (discard) and with-path (file-only) cases in both apps.
- Running `./kopds delete-user <name>` with a log file configured prints only the one-line summary to stdout; the structured entries appear in the log file.
- Server startup and request logging are unchanged.

---

## Notes For Future Implementers

- `NewCLI` and `New` share level-parsing and handler-construction logic. If `New` is ever changed (e.g. new log levels, handler options), apply the same change to `NewCLI` to keep them in sync.
- Do not add stderr output back to `NewCLI`. The deliberate design is: CLI = terminal-friendly (file or discard only); server = tee to stderr + file.
- The `runCLI` function and its helpers pass `nil` as the logger to `LogCLISuccess`/`LogCLIFailure`, relying on `slog.Default()`. This is intentional — it means the CLI branch picks up whichever default was set in `main()` without needing a logger argument threaded through every function.
