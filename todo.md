# To Do List

At the beginning of work on each step, prior to making any changes to any code, place a mark in the box next to that step (eg. [-]) so that it is clear what work was being done if there is a sudden interruption. After the completion of each step, git commit the changes with a descriptive message, then update this document to show the current state of progress by checking the box next to the step (eg. [x]).



## Testing

- [ ] Develop a procedure for testing the storage cap for both KOPDS and KOSYNC.
- [ ] Develop a procedure for testing that progress updates (`PUT`) in KOSYNC with newer timestamps override, but older timestamps do not.


## HTTP Routing

- [x] **Refactor KOPDS HTTP routing to use `net/http.ServeMux`**: Removed chi dependency from KOPDS and refactored all routing to use stdlib `net/http.ServeMux` to align with KOSYNC. Changes include: removed `github.com/go-chi/chi/v5` from go.mod; replaced `chi.NewRouter()` with `http.NewServeMux()`; replaced all `chi.URLParam(r, "param")` calls with `r.PathValue("param")`; refactored nested routing with manual route registration and middleware wrapping; removed chi-specific middleware (RequestID, RealIP, GetHead, Logger, Recoverer, Timeout); refactored BasicAuth middleware to work with stdlib http handlers; updated both KOPDS and KOSYNC READMEs to document uniform use of `net/http.ServeMux`. All unit tests and integration tests pass.



## User Management

Current behavior: `create-user` overwrites/updates an existing user for both KOPDS and KOSYNC.  Desired behavior: `create-user` fails if the user already exists for both KOPDS and KOSYNC.  Additional tasks: update usage documentation in the `README.md` for both KOPDS and KOSYNC if, and only if, the overwrite/update functionality is currently mentioned.

### Phase 1: Prepository/Storage Layer Updates

- [ ] **1. Add CreateUserIfNotExists() to KOPDS UserRepository**: In `kopds/internal/database/user_repository.go`, add a new method `CreateUserIfNotExists(ctx context.Context, user *domain.User) error` that checks for existing user before inserting, returning an error (e.g., "user already exists") if the user is found. Keep the existing `Save()` method unchanged for backwards compatibility.
- [ ] **2. Add CreateUserIfNotExists() to KOSYNC Storage**: In `kosync/internal/database/sqlite.go`, add a new method `CreateUserIfNotExists(username, password string) error` that checks for existing user before inserting, returning an error (e.g., "user already exists") if the user is found. Keep the existing `SaveUser()` method unchanged for backwards compatibility.

### Phase 2: CLI Updates

- [ ] **3. Update KOPDS createUser() CLI function**: Modify `kopds/cmd/kopds/main.go` `createUser()` to call the new `CreateUserIfNotExists()` method. Handle the "user already exists" error with appropriate log message and stdout message (e.g., "Error: User 'username' already exists").
- [ ] **4. Update KOSYNC createUser() CLI function**: Modify `kosync/cmd/kosync/main.go` `createUser()` to call the new `CreateUserIfNotExists()` method. Handle the "user already exists" error with appropriate log message and stdout message (e.g., "Error: User 'username' already exists").
- [ ] **5. Standardize success message**: Ensure both KOPDS and KOSYNC print identical success messages for `create-user` (e.g., "User 'username' created successfully." without "/updated").

### Phase 3: Test Updates

- [ ] **6. Add KOPDS test for duplicate user creation**: In `kopds/cmd/kopds/main_test.go`, add a test case that attempts to create a user twice and verifies the second attempt fails with appropriate error message and non-zero exit code.
- [ ] **7. Add KOSYNC test for duplicate user creation**: In `kosync/cmd/kosync/main_test.go`, add a test case that attempts to create a user twice and verifies the second attempt fails with appropriate error message and non-zero exit code.
- [ ] **8. Run KOPDS tests**: Execute `cd kopds && go test ./...` to verify all tests pass including the new user creation tests.
- [ ] **9. Run KOSYNC tests**: Execute `cd kosync && go test ./...` to verify all tests pass including the new user creation tests.

### Phase 4: Documentation

- [ ] **10. Check KOPDS README for create-user documentation**: Review `kopds/README.md` and update the `create-user` command documentation to clarify that the command fails if the user already exists and only succeeds if the user is new. Remove any mention of "overwrite" or "update" behavior for `create-user`.
- [ ] **11. Check KOSYNC README for create-user documentation**: Review `kosync/README.md` and update the `create-user` command documentation to clarify that the command fails if the user already exists and only succeeds if the user is new. Remove any mention of "overwrite" or "update" behavior for `create-user`.
- [ ] **12. Update uniformity notes**: Add a note documenting that both KOPDS and KOSYNC now have identical `create-user` behavior: fail if user exists, succeed only for new users.



## Logging

- [ ] Develop a matrix or table showing all possible commands/requests/inputs and what log level is expected for it for both KOPDS and KOSYNC
- [ ] Current behavior: when deployed as a Docker container with `LOG_LEVEL=INFO`, `docker logs -f <app>` does not show entries for CLI user management events.  Desired behavior: when deployed as a Docker container, `docker logs -f <app>` does show entries for CLI user management events.
- [ ] Current behavior: when deployed as a Docker container with `LOG_LEVEL=INFO` or `LOG_LEVEL=DEBUG`, KOSYNC does display successful progress updates (`PUT`) but does not display successful get progress requests (`GET`)(unsuccessful updates and requests are both logged).  Desired behavior: when deployed as a Docker container with `LOG_LEVEL=INFO` of `LOG_LEVEL=DEBUG`, KOSYNC does display both successful and unsuccessful progress updates (`PUT`) and get progress requests (`GET`).
- [ ] Current behavior: when deployed as a Docker container with `LOG_LEVEL=INFO`, KOSYNC does not display every successful HTTP request, but KOPDS does (in a different format than the other log entries).  What is KOPDS doing differently, and why do the entries of HTTP requests appear in a different format than entries for starting up the container or creating a new user?
