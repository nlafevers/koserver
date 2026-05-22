# Comprehensive Testing: KOSYNC & KOPDS



## User/Admin Inputs

### CLI
From a shell, execute the following commands.  They are the same for both KOPDS and KOSYNC.

- **User Creation (interactive password)**: `./<app> create-user <user>`
- **User Creation (stdin password)**: `echo "<password>" | ./<app> create-user <user> --password-stdin`
- **User Deletion**: `./<app> delete-user <user>`
- **Password Change (interactive)**: `./<app> change-password <user>`
- **Password Change (stdin)**: `echo "<password>" | ./<app> change-password <user> --password-stdin`

### HTTP
From a shell, use these `curl` commands to simulate network traffic with the servers.

#### KOPDS
- **Server Health Check**: `curl -v http://localhost:8080/health`
- **User Authentication**: `curl -v -u <user>:<pass> http://localhost:8080/opds/v1.2/catalog`
- **Navigation (Authors)**: `curl -v -u <user>:<pass> http://localhost:8080/opds/v1.2/authors`
- **Navigation (Newest)**: `curl -v -u <user>:<pass> http://localhost:8080/opds/v1.2/newest`
- **Search**: `curl -v -u <user>:<pass> "http://localhost:8080/opds/v1.2/search?q=Programming"`
- **Book Detail**: `curl -v -u <user>:<pass> http://localhost:8080/opds/v1.2/books/1`
- **Cover Image (Resized)**: `curl -v -u <user>:<pass> "http://localhost:8080/opds/v1.2/cover/1?w=200&h=300"`
- **File Download**: `curl -v -u <user>:<pass> http://localhost:8080/opds/v1.2/download/1/epub --output book.epub`

#### KOSYNC
- **Server Health Check**: `curl -v http://localhost:8081/health`
- **User Registration**: `curl -v -X POST http://localhost:8081/users/create -H "Content-Type: application/json" -d '{"username":"<user>", "password":"<md5_pass>"}'`
- **User Authentication**: `curl -v -H "X-AUTH-USER: <user>" -H "X-AUTH-KEY: <md5_pass>" -H "Accept: application/vnd.koreader.v1+json" http://localhost:8081/users/auth`
- **Get Progress**: `curl -v -H "X-AUTH-USER: <user>" -H "X-AUTH-KEY: <md5_pass>" -H "Accept: application/vnd.koreader.v1+json" http://localhost:8081/syncs/progress/<doc_id>`
- **Update Progress**: `curl -v -X PUT http://localhost:8081/syncs/progress -H "Content-Type: application/json" -H "X-AUTH-USER: <user>" -H "X-AUTH-KEY: <md5_pass>" -H "Accept: application/vnd.koreader.v1+json" -d '{"document":"<doc_id>", "percentage":0.5, "progress":"page 10", "device_id":"123", "device":"kindle"}'`

### Docker
When the server is running in a docker container, use these commands from the host shell to check logs or pass through the CLI user management commands.

- **View Logs (Follow)**: `docker logs -f <app>`
- **CLI (Interactive Password)**: `docker exec -it <container> ./<app> create-user <user>`
- **CLI (Automation)**: `echo "<password>" | docker exec -i <container> ./<app> create-user <user> --password-stdin`
- **Restart Container**: `docker compose restart <app>`
- **Container Stats**: `docker stats`

### SQLite
From a shell, use these commands to verify the contents of the database after running any of the CLI user management commands, or any HTTP requests that causes changes in the database.

- **List All Users**: `sqlite3 data/<app>.db "SELECT username FROM users;"`
- **Check Progress (KOSYNC)**: `sqlite3 data/kosync.db "SELECT * FROM progress ORDER BY timestamp DESC LIMIT 10;"`
- **Check Book Count (KOPDS)**: `sqlite3 data/kopds.db "SELECT count(*) FROM books;"`
- **Check Sync Status (KOPDS)**: `sqlite3 data/kopds.db "SELECT * FROM sync_state;"`
- **Verify FTS5 Search (KOPDS)**: `sqlite3 data/kopds.db "SELECT title FROM books_search WHERE books_search MATCH 'Programming';"`



## Verification Checklist
This checklist ensures that both servers are fully operational, harmonized, and secure when running locally.

### Work Done So Far
- [x] **Database creation**: both kopds and kosync create the .db and the dir if needed on first run
- [x] **Database permissions**: both kopds and kosync create a .db with `-rw-------` permissions
- [x] **CLI user creation**: both kopds and kosync report success the same with creating new users with both interactive and stdin password modes
- [x] **CLI user deletion**: both kopds and kosync report success the same deleting existing users and failure deleting non-existing users
- [x] **CLI password changes**: both kopds and kosync report success the same changing passwords with both interactive and stdin password modes
- [x] **Log creation**: both kopds and kosync create a log if one is specified in `config.yaml`
- [x] **Logging of events**: both kopds and kosync log the above CLI actions the same, as well as startup and shutdown



## 1. Shared Infrastructure & CLI
Verify these core features work identically on both projects.

- [x] **Database Permissions**:
    - **Verify**: Delete the `.db` file and restart the server. Verify the new file is created with `0600` permissions.
    - **Command**: `rm data/<app>.db && go build -o <app> cmd/<app>/main.go && ./<app>`
    - **Check**: `ls -l data/<app>.db` (should show `-rw-------`)
- [x] **CLI User Creation (Interactive)**:
    - **Verify**: Password masking works and user is created.
    - **Command**: `./<app> create-user testuser`
    - **Expect**: "Password: " prompt, confirm prompt.
- [x] **CLI User Creation (Automation)**:
    - **Verify**: Success via stdin.
    - **Command**: `echo "password" | ./<app> create-user bot --password-stdin`
- [x] **CLI Password Change**:
    - **Verify**: New password works for authentication.
    - **Command**: `./<app> change-password testuser`
- [x] **CLI User Deletion**:
    - **Verify**: User can no longer authenticate.
    - **Command**: `./<app> delete-user testuser`
- [ ] **Storage Cap Enforcement**:
    - **Verify**: Oldest records are pruned and `VACUUM` is triggered in the logs.
    - **Command**: Start server with `STORAGE_CAP_MB=1` and simulate activity.
    - **Check**: Look for `storage cap enforced: oldest records pruned` in logs.
- [x] **Docker Deployment**:
    - **Verify**: Containers start and respond.
    - **Command**: `docker compose up -d` (in `deploy/` dir).
    - **Check**: `curl -v http://localhost:8080/health`
- [x] **Unified Logging**:
    - **Verify**: Logs show `source` and go to stdout/log file.
    - **Command**: `tail -f <log-file>`

## 2. KOSYNC Protocol Verification
- [x] **Strict Headers**:
    - **Command**: `curl -v http://localhost:8081/syncs/progress/doc1`
    - **Expect**: `406 Not Acceptable`.
- [x] **Registration Endpoint**:
    - **Command**: `curl -v -X POST http://localhost:8081/users/create -H "Accept: application/vnd.koreader.v1+json" -d '{"username":"u", "password":"p"}'`
    - **Expect**: `201 Created`.
- [x] **Authentication**:
    - **Command**: `curl -v http://localhost:8081/users/auth -H "X-AUTH-USER: u" -H "X-AUTH-KEY: p"`
    - **Expect**: `200 OK`.
- [ ] **Progress Sync**:
    - **Verify**: `PUT` with newer timestamp overrides; `PUT` with older does not.

## 3. KOPDS Protocol & Features
- [x] **Calibre Scanner**:
    - **Verify**: "Synchronization batch completed" in logs.
- [x] **OPDS Navigation**:
    - **Verify**: `/opds/v1.2/catalog` works and Atom links are valid.
- [x] **Search (FTS5)**:
    - **Command**: `curl "http://localhost:8080/opds/v1.2/search?q=Programming"`
- [x] **Book Delivery**:
    - **Verify**: EPUB `Content-Type` and image resizing.
- [x] **Security (Basic Auth)**:
    - **Verify**: `401 Unauthorized` without credentials.

## 4. Final System Integrity
- [x] **Graceful Shutdown**: `SIGINT` (Ctrl+C). Check for "server exited cleanly".
- [x] **Environment Variable Override**: `PORT=9090 ./<app>`
- [x] **Config File Support**: Change `port` in `config/config.yaml`.
- [x] **Dependency Check**: `go.mod` consistency check.
