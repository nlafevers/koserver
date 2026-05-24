# To Do List

At the beginning of work on each step, prior to making any changes to any code, place a mark in the box next to that step (eg. [-]) so that it is clear what work was being done if there is a sudden interruption. After the completion of each step, git commit the changes with a descriptive message, then update this document to show the current state of progress by checking the box next to the step (eg. [x]).



## Testing

- [ ] Develop a procedure for testing the storage cap for both KOPDS and KOSYNC.
- [ ] Develop a procedure for testing that progress updates (`PUT`) in KOSYNC with newer timestamps override, but older timestamps do not.


## HTTP Routing

Refactor KOPDS to use `net/http.ServeMux` instead of `go-chi/chi/v5` to align with KOSYNC's approach and minimize external dependencies.

### Phase 1: Preparation and Dependency Removal

- [x] **9.1 Remove chi dependency from KOPDS go.mod**: Delete the `github.com/go-chi/chi/v5` import and run `go mod tidy` in the `kopds` directory.
- [x] **9.2 Remove chi imports from KOPDS code**: Remove all `import` statements for `github.com/go-chi/chi/v5` and `github.com/go-chi/chi/v5/middleware` across the codebase.

### Phase 2: Router and Middleware Setup

- [x] **9.3 Replace chi.NewRouter() with http.NewServeMux()**: In `kopds/cmd/kopds/main.go`, replace the chi router setup with `mux := http.NewServeMux()` and remove all chi-specific middleware registration (RequestID, RealIP, GetHead, Logger, Recoverer, Timeout).
- [x] **9.4 Decide on chi middleware replacement strategy**: Determine which chi middleware to replicate or remove:
  - `RequestID`: Optional; can be removed (not critical for KOPDS)
  - `Logger`: Optional; chi's automatic logging can be removed (KOPDS has explicit slog logging in handlers)
  - `Recoverer`: Optional; can be removed (Go's default panic handling is acceptable)
  - `Timeout`: Replace with per-handler `context.WithTimeout` if needed
  - Recommendation: Remove all for simplicity; KOPDS handlers already use explicit error handling and logging.
- [x] **9.5 Refactor BasicAuth middleware**: Extract chi-specific BasicAuth middleware from `kopds/internal/api/middleware.go` into a standalone middleware wrapper function that returns an `http.Handler`, similar to how KOSYNC's middleware functions work.

### Phase 3: Route Pattern Replacement

- [x] **9.6 Replace chi.URLParam() with r.PathValue()**: In `kopds/internal/api/handlers.go`, replace all instances of `chi.URLParam(r, "paramName")` with `r.PathValue("paramName")` (native to Go 1.22+).
- [x] **9.7 Refactor chi.Route() nested routing**: Replace the `r.Route("/opds/v1.2", func(r chi.Router) {...})` pattern with manual route registration:
  - Manually register each `/opds/v1.2/*` route with the mux
  - Wrap the protected routes handler with the BasicAuth middleware, similar to how KOSYNC wraps protected routes in `middleware.AuthMiddleware()`

### Phase 4: Route Registration Update

- [x] **9.8 Update route registration in runServer()**: In `kopds/cmd/kopds/main.go`, replace chi-style route registration with `mux.HandleFunc()` calls for both public and protected routes.  Example patterns:
  - `mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {...})`
  - `protected := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {...})` for grouped protected routes
  - Wrap `protected` with BasicAuth middleware before registering with mux

### Phase 5: Testing and Verification

- [x] **9.9 Run KOPDS unit tests**: Execute `cd kopds && go test ./...` to verify all tests pass without chi dependency.
- [x] **9.10 Run KOPDS integration tests**: Execute `kopds/test/integration_test.sh` to verify the server starts and responds to requests correctly.
- [x] **9.11 Manual smoke test**: Start KOPDS server and test a few key endpoints manually (e.g., `/health`, `/opds/v1.2/catalog`, `/opds/v1.2/authors`).

### Phase 6: Documentation

- [x] **9.12 Update KOPDS README**: Document that KOPDS now uses `net/http.ServeMux` and remove any chi-specific setup instructions.
- [x] **9.13 Update KOSYNC README**: Document that KOSYNC also uses `net/http.ServeMux`, making the HTTP routing consistent across both projects.
- [x] **9.14 Update uniformity notes**: Add a note in `kopds/UNIFORMITY.md` or `kosync/UNIFORMITY.md` documenting HTTP routing uniformity as complete.



## User Management 

- [ ] Current behavior: `create-user` overwrites/updates an existing user for both KOPDS and KOSYNC.  Desired behavior: `create-user` fails if the user already exists for both KOPDS and KOSYNC.  Additional tasks: update usage documentation in the `README.md` for both KOPDS and KOSYNC if, and only if, the overwrite/update functionality is currently mentioned.



## Logging

- [ ] Develop a matrix or table showing all possible commands/requests/inputs and what log level is expected for it for both KOPDS and KOSYNC
- [ ] Current behavior: when deployed as a Docker container with `LOG_LEVEL=INFO`, `docker logs -f <app>` does not show entries for CLI user management events.  Desired behavior: when deployed as a Docker container, `docker logs -f <app>` does show entries for CLI user management events.
- [ ] Current behavior: when deployed as a Docker container with `LOG_LEVEL=INFO` or `LOG_LEVEL=DEBUG`, KOSYNC does display successful progress updates (`PUT`) but does not display successful get progress requests (`GET`)(unsuccessful updates and requests are both logged).  Desired behavior: when deployed as a Docker container with `LOG_LEVEL=INFO` of `LOG_LEVEL=DEBUG`, KOSYNC does display both successful and unsuccessful progress updates (`PUT`) and get progress requests (`GET`).
- [ ] Current behavior: when deployed as a Docker container with `LOG_LEVEL=INFO`, KOSYNC does not display every successful HTTP request, but KOPDS does (in a different format than the other log entries).  What is KOPDS doing differently, and why do the entries of HTTP requests appear in a different format than entries for starting up the container or creating a new user?
