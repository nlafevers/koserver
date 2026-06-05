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

- [x] **TDL-001 Storage Lifecycle Wrappers**: Standardized SQLite open/migrate/inject flow in both apps. Both use `NewSQLite(path, allowCreate)` as a thin wrapper; removed KOSYNC `InitDB` and `Storage.Close()`.

- [x] **TDL-002** Add `logger.NewCLI` to both apps (Commits: kopds:c77830e, kosync:d03a9ad). Added `NewCLI(level, json, logPath)` to `internal/logger/logger.go` in both repos — identical to `New` but writes to the log file only (or `io.Discard`), keeping the terminal clean. Added `logger_test.go` in both repos covering no-path (discard) and with-path (file-only) cases.

- [x] **TDL-003** Wire `logger.NewCLI` into CLI mode in both apps (Commits: kopds:27f63e3, kosync:1facdad). Restructured `main()` to call `logger.NewCLI` before `runCLI` and `logger.New` before `runServer`. CLI commands now print only the human-readable summary to the terminal; structured entries go to the log file or are silently discarded.

---

## TDL-004/005: Stable Config Search Paths for Native Installs

**Goal:** Both apps currently search for `config.yaml` only in the working directory (`.` and `./config`). Under systemd the working directory is `/`; for CLI invocations it is wherever the user's shell is. Neither is a predictable location. As a result `config.yaml` only works reliably in Docker (where the working directory is `/app`). Native users must pass every setting as an environment variable. Add stable, conventional search paths so a config file placed in a known location is found automatically without environment variables.

**Fix:** In `Load()` in `internal/config/config.go` in **both** apps, add three `viper.AddConfigPath` calls immediately after the existing `viper.AddConfigPath("./config")` line. The new paths are added **after** the existing ones so Docker and development working-directory configs still take precedence. `os` and `path/filepath` are already imported in both files.

- [ ] **TDL-004** Add config search paths to KOPDS
  - **Repos:** kopds
  - **Read:** kopds/internal/config/config.go, kopds/internal/config/config_test.go
  - **Edit:** kopds/internal/config/config.go, kopds/internal/config/config_test.go
  - **Instructions:** In `kopds/internal/config/config.go`, in the `Load()` function, add the following three lines immediately after `viper.AddConfigPath("./config")` (currently line 37). No new imports are needed.

    ```go
    if dir, err := os.UserConfigDir(); err == nil {
        viper.AddConfigPath(filepath.Join(dir, "kopds"))
    }
    viper.AddConfigPath("/var/lib/kopds")
    viper.AddConfigPath("/etc/kopds")
    ```

    `os.UserConfigDir()` returns `$XDG_CONFIG_HOME` when set, otherwise `$HOME/.config` on Linux. The conditional silently skips the path if the OS cannot determine a config directory.

    In `kopds/internal/config/config_test.go`, add the following test function after the existing tests. No new imports are needed (`os`, `path/filepath`, and `testing` are already imported).

    ```go
    func TestLoadConfigFromUserConfigDir(t *testing.T) {
        tmp := t.TempDir()
        // XDG_CONFIG_HOME overrides $HOME/.config; os.UserConfigDir() respects it.
        t.Setenv("XDG_CONFIG_HOME", tmp)
        os.Unsetenv("KOPDS_LOG_LEVEL")

        cfgDir := filepath.Join(tmp, "kopds")
        if err := os.MkdirAll(cfgDir, 0755); err != nil {
            t.Fatalf("MkdirAll: %v", err)
        }
        if err := os.WriteFile(
            filepath.Join(cfgDir, "config.yaml"),
            []byte("log_level: debug\n"),
            0644,
        ); err != nil {
            t.Fatalf("WriteFile: %v", err)
        }

        cfg, err := Load()
        if err != nil {
            t.Fatalf("Load: %v", err)
        }
        if cfg.LogLevel != "debug" {
            t.Errorf("want log_level=debug from config file in UserConfigDir, got %s", cfg.LogLevel)
        }
    }
    ```

  - **Verify:** `cd /home/nathan/koserver/kopds && gofmt -l internal/config/ && GOCACHE=/tmp/kopds-gocache go build ./... && GOCACHE=/tmp/kopds-gocache go test ./internal/config/...`
  - **Done when:** the three `AddConfigPath` calls are present after `./config` in `Load()`, `TestLoadConfigFromUserConfigDir` passes, all existing config tests still pass, gofmt prints nothing, and build passes; one commit in the kopds repo

- [ ] **TDL-005** Add config search paths to KOSYNC
  - **Repos:** kosync
  - **Read:** kosync/internal/config/config.go, kosync/internal/config/config_test.go
  - **Edit:** kosync/internal/config/config.go, kosync/internal/config/config_test.go
  - **Instructions:** In `kosync/internal/config/config.go`, in the `Load()` function, add the following three lines immediately after `viper.AddConfigPath("./config")` (currently line 52). No new imports are needed — `os` and `path/filepath` are already imported.

    ```go
    if dir, err := os.UserConfigDir(); err == nil {
        viper.AddConfigPath(filepath.Join(dir, "kosync"))
    }
    viper.AddConfigPath("/var/lib/kosync")
    viper.AddConfigPath("/etc/kosync")
    ```

    In `kosync/internal/config/config_test.go`, add `"path/filepath"` to the import block (it is not currently imported there). Then add the following test function after the existing tests:

    ```go
    func TestLoadConfigFromUserConfigDir(t *testing.T) {
        tmp := t.TempDir()
        // XDG_CONFIG_HOME overrides $HOME/.config; os.UserConfigDir() respects it.
        t.Setenv("XDG_CONFIG_HOME", tmp)
        os.Unsetenv("KOSYNC_LOG_LEVEL")

        cfgDir := filepath.Join(tmp, "kosync")
        if err := os.MkdirAll(cfgDir, 0755); err != nil {
            t.Fatalf("MkdirAll: %v", err)
        }
        if err := os.WriteFile(
            filepath.Join(cfgDir, "config.yaml"),
            []byte("log_level: debug\n"),
            0644,
        ); err != nil {
            t.Fatalf("WriteFile: %v", err)
        }

        cfg, err := Load()
        if err != nil {
            t.Fatalf("Load: %v", err)
        }
        if cfg.LogLevel != "debug" {
            t.Errorf("want log_level=debug from config file in UserConfigDir, got %s", cfg.LogLevel)
        }
    }
    ```

  - **Verify:** `cd /home/nathan/koserver/kosync && gofmt -l internal/config/ && GOCACHE=/tmp/kosync-gocache go build ./... && GOCACHE=/tmp/kosync-gocache go test ./internal/config/...`
  - **Done when:** the three `AddConfigPath` calls are present after `./config` in `Load()`, `"path/filepath"` is added to the import block of `config_test.go`, `TestLoadConfigFromUserConfigDir` passes, all existing config tests still pass, gofmt prints nothing, and build passes; one commit in the kosync repo

Acceptance criteria for TDL-004/005:
- Both `internal/config/config.go` files register `os.UserConfigDir()/<app>`, `/var/lib/<app>`, and `/etc/<app>` as config search paths, in that order, after `./config`.
- A `config.yaml` placed in `$XDG_CONFIG_HOME/kopds/` (or `$XDG_CONFIG_HOME/kosync/`) is found and loaded by `Load()`.
- Working-directory configs still take precedence over the new paths.
- All existing tests continue to pass in both repos.

---

## Notes For Future Implementers

- `NewCLI` and `New` share level-parsing and handler-construction logic. If `New` is ever changed (e.g. new log levels, handler options), apply the same change to `NewCLI` to keep them in sync.
- Do not add stderr output back to `NewCLI`. The deliberate design is: CLI = terminal-friendly (file or discard only); server = tee to stderr + file.
- The `runCLI` function and its helpers pass `nil` as the logger to `LogCLISuccess`/`LogCLIFailure`, relying on `slog.Default()`. This is intentional — it means the CLI branch picks up whichever default was set in `main()` without needing a logger argument threaded through every function.
