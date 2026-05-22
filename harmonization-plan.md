# Harmonization Plan: KOSYNC and KOPDS

This plan outlines the steps to harmonize the `kosync` and `kopds` projects, bringing both to the same level of feature parity, security, and architectural maturity.  But also to impose uniformity of style and replicate code snippets exactly where the same function exists in both projects.

## 1. Project Overview & Shared Features

Both projects are Go-based servers designed to support KOReader clients, using SQLite for persistence and providing CLI tools for administration.

### Shared Features:
- **CLI User Management**: An admin tool to create and delete users, as well as change passwords.
- **Database**: SQLite-based storage using the pure Go `modernc.org/sqlite` driver.
- **Authentication**: Secure password hashing with Bcrypt.
- **Logging**: Structured logging for observability.
- **Deployment**: Support for binary and Docker-based deployment.

## 2. Comparative Analysis

### KOSYNC (Advanced Areas):
- **CLI Capabilities**: Comprehensive management (`create-user`, `delete-user`, `change-password`) with interactive password masking and automation support.
- **Logging Features**: Unified logging to files; CLI actions are logged to the same destination as the server.
- **Security**: Strict file permission enforcement (`0600`) for the database file.
- **Reliability**: Automated storage cap enforcement to prevent disk exhaustion.
- **Documentation**: Extremely detailed, novice-friendly documentation with security guides.

### KOPDS (Advanced Areas):
- **Project Architecture**: Standard Go layout (`cmd/`, `internal/`) providing better separation of concerns and scalability.
- **Configuration Management**: Uses `viper`, allowing configuration via YAML files and environment variables.
- **Domain Modeling**: Formalized entities and repository patterns.
- **Rich Middleware**: Standard middleware stack (Request ID, Real IP, Recoverer) via the `chi` router.

## 3. Harmonization Roadmap

At the beginning of work on each step, prior to making any changes to any code, place a mark in the box next to that step (eg. [-]) so that it is clear what work was being done if there is a sudden interruption. After the completion of each step, git commit the changes with a descriptive message, then update this document to show the current state of progress by checking the box next to the step (eg. [x]).

### Phase 1: KOPDS CLI & Logging Upgrades
*Focus: Bringing KOPDS operational capabilities up to the KOSYNC standard.*

- [x] **1.1 Standardize KOPDS CLI Commands**: Refactor `createUser` in `kopds/cmd/kopds/main.go` to support the `./kopds <command> <username>` format and add help usage.
- [x] **1.2 Add delete-user to KOPDS**: Implement the `delete-user` command in `kopds` CLI and the corresponding repository method.
- [x] **1.3 Add change-password to KOPDS**: Implement the `change-password` command in `kopds` CLI and the corresponding repository method.
- [x] **1.4 Migrate KOPDS to slog**: Replace `zerolog` with the standard library `slog` for consistency with `kosync`.
- [x] **1.5 Implement Unified File Logging in KOPDS**: Update `KOPDS` to support logging to a file and ensure CLI actions are logged with the `source: CLI` attribute.

### Phase 2: KOSYNC Architectural Restructuring
*Focus: Bringing KOSYNC up to the KOPDS architectural standard.*

- [x] **2.1 Initialize KOSYNC Project Layout**: Create `cmd/kosync` and `internal/` directory structures in the `kosync` project.
- [x] **2.2 Move KOSYNC Config to viper**: Refactor `kosync/config.go` to use `viper`, supporting `config/config.yaml` and environment variables.
- [x] **2.3 Extract KOSYNC Storage Layer**: Move database logic from `kosync/storage.go` to `kosync/internal/database/`.
- [x] **2.4 Extract KOSYNC Handlers & Middleware**: Move web logic to `kosync/internal/api/` and `kosync/internal/middleware/`.
- [x] **2.5 Finalize KOSYNC Main Entry Point**: Update `kosync/cmd/kosync/main.go` to use the new internal packages.

### Phase 3: Storage, Security & Reliability Parity
*Focus: Synchronizing advanced safeguards across both projects.*

- [x] **3.1 Enforce DB Permissions in KOPDS**: Port the `0600` file permission check and enforcement from `kosync` to `kopds`.
- [x] **3.2 Port Storage Cap Enforcement to KOPDS**: Implement the `EnforceStorageCap` logic in `kopds` to prevent disk exhaustion.
- [x] **3.3 Synchronize SQLite PRAGMAs**: Verify and ensure both projects use identical SQLite settings (WAL mode, `SetMaxOpenConns(1)`).
- [x] **3.4 Middleware Alignment**: Port `KOSYNC`'s strict `Accept` and `Content-Type` headers to `KOPDS` to ensure identical client compatibility.

### Phase 4: Documentation & Final Cleanup
*Focus: Ensuring both projects are equally documented and maintained.*

- [x] **4.1 Comprehensive KOPDS Documentation Update**: Update `kopds/README.md` and `GEMINI.md` with the security and proxy guides from `kosync`.
- [x] **4.2 Harmonize Dependencies**: Update `go.mod` files in both projects to use consistent versions of shared libraries (e.g., `slog`, `modernc.org/sqlite`).
- [x] **4.3 Global Verification**: Run all tests for both projects and perform a final manual verification of all CLI commands.

### Phase 5: Deployment & Integration Testing Parity
*Focus: Synchronizing deployment artifacts and black-box testing across both projects.*

- [x] **5.1 Create KOPDS Integration Test**: Implement `kopds/test/integration_test.sh` to mirror `kosync`'s black-box testing.
- [x] **5.2 Standardize KOSYNC Data Directory**: Create `kosync/data/` and update `config.go` to use it by default for the database.
- [x] **5.3 Add KOSYNC Configuration**: Create `kosync/config/config.yaml` with default settings.
- [x] **5.4 Containerize KOSYNC**: Create `kosync/build/Dockerfile` using the same multi-stage pattern as `kopds/build/Dockerfile`.
- [x] **5.5 Implement KOSYNC Compose**: Create `kosync/deploy/docker-compose.yml` matching the structure of `kopds/deploy/docker-compose.yml`.

### Phase 6
*Cleaning up persistent inconsistencies discovered in testing.*

- [ ] **6.1 CLI Command Logging**: After building KOSYNC, but before running it, an attempt to run the `create-user` command gives an error that the database does not exist.  For kopds, the same procedure succeeds.  So clearly the way they create or initialize the databases is still not harmonized.
- [ ] **6.2 CLI Command Ouput**: KOSYNC outputs after every command a lines with `Using database: ...` and on a new line `Using log: ...`, then the message.  KOPDS does not have the the first two lines, only the message (eg. time=2026-05-18T06:36:31.272-04:00 level=INFO msg="user created successfully" username=user1 source=CLI
User 'user1' created successfully).  But again they should be the same.

## 4. Verification & Testing
- **Cross-Project Testing**: Ensure CLI commands work identically across both projects.
- **Integration Tests**: Run the existing integration tests for both projects after restructuring and harmonization.
- **Manual Verification**: Verify database permissions and storage cap enforcement manually.
