# To Do List

At the beginning of work on each step, prior to making any changes to any code, place a mark in the box next to that step (eg. [-]) so that it is clear what work was being done if there is a sudden interruption. After the completion of each step, git commit the changes with a descriptive message, then update this document to show the current state of progress by checking the box next to the step (eg. [x]).



## Testing

- [ ] Develop a procedure for testing the storage cap for both KOPDS and KOSYNC.
- [ ] Develop a procedure for testing that progress updates (`PUT`) in KOSYNC with newer timestamps override, but older timestamps do not.


## HTTP Routing

- [x] **Refactor KOPDS HTTP routing to use `net/http.ServeMux`**: Removed chi dependency from KOPDS and refactored all routing to use stdlib `net/http.ServeMux` to align with KOSYNC. Changes include: removed `github.com/go-chi/chi/v5` from go.mod; replaced `chi.NewRouter()` with `http.NewServeMux()`; replaced all `chi.URLParam(r, "param")` calls with `r.PathValue("param")`; refactored nested routing with manual route registration and middleware wrapping; removed chi-specific middleware (RequestID, RealIP, GetHead, Logger, Recoverer, Timeout); refactored BasicAuth middleware to work with stdlib http handlers; updated both KOPDS and KOSYNC READMEs to document uniform use of `net/http.ServeMux`. All unit tests and integration tests pass.



## User Management

- [x] **Refactor `create-user` to prevent overwriting existing users**: Standardized `create-user` behavior across KOPDS and KOSYNC to fail if a user already exists. Changes include: implemented `CreateUserIfNotExists` in both storage layers; updated CLI entrypoints to handle the "user already exists" error with standardized log and stdout messages; standardized success messages to "User 'username' created successfully."; updated CLI tests in both projects to verify failure and non-zero exit codes for duplicate creation; updated READMEs to reflect the new behavior and refer users to `change-password` for updates; and updated uniformity documentation. All tests pass.



## Logging

- [x] **Standardize logging across KOPDS and KOSYNC**: Refactored the logging system in both projects to use the standard library `log/slog` with a maximum-uniformity goal. Implemented uniform `LoggingMiddleware` for HTTP request/response tracking with unique `request_id` correlation; standardized CLI user-management logging (success/failure); added structured logging to handlers, services, and repositories (INFO for business events, DEBUG for diagnostics); and unified storage cap/maintenance logging. Updated all READMEs and `AGENTS.md` files with the new logging strategy and verified comprehensive coverage through expanded unit and integration tests.
