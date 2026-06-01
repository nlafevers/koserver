# Commands

## CLI
From a shell, execute the following commands.  They are the same for both KOPDS and KOSYNC.

- **User Creation (interactive password)**: `./<app> create-user <user>`
- **User Creation (stdin password)**: `echo "<password>" | ./<app> create-user <user> --password-stdin`
- **User Deletion**: `./<app> delete-user <user>`
- **Password Change (interactive)**: `./<app> change-password <user>`
- **Password Change (stdin)**: `echo "<password>" | ./<app> change-password <user> --password-stdin`

## HTTP
From a shell, use these `curl` commands to simulate network traffic with the servers.

### KOPDS
- **Server Health Check**: `curl -v http://localhost:8080/health`
- **User Authentication**: `curl -v -u <user>:<pass> http://localhost:8080/opds/v1.2/catalog`
- **Navigation (Authors)**: `curl -v -u <user>:<pass> http://localhost:8080/opds/v1.2/authors`
- **Navigation (Newest)**: `curl -v -u <user>:<pass> http://localhost:8080/opds/v1.2/newest`
- **Search**: `curl -v -u <user>:<pass> "http://localhost:8080/opds/v1.2/search?q=Programming"`
- **Book Detail**: `curl -v -u <user>:<pass> http://localhost:8080/opds/v1.2/books/1`
- **Cover Image (Resized)**: `curl -v -u <user>:<pass> "http://localhost:8080/opds/v1.2/cover/1?w=200&h=300"`
- **File Download**: `curl -v -u <user>:<pass> http://localhost:8080/opds/v1.2/download/1/epub --output book.epub`

### KOSYNC
- **Server Health Check**: `curl -v http://localhost:8081/health`
- **User Registration**: `curl -v -X POST http://localhost:8081/users/create -H "Content-Type: application/json" -d '{"username":"<user>", "password":"<md5_pass>"}'`
- **User Authentication**: `curl -v -H "X-AUTH-USER: <user>" -H "X-AUTH-KEY: <md5_pass>" -H "Accept: application/vnd.koreader.v1+json" http://localhost:8081/users/auth`
- **Get Progress**: `curl -v -H "X-AUTH-USER: <user>" -H "X-AUTH-KEY: <md5_pass>" -H "Accept: application/vnd.koreader.v1+json" http://localhost:8081/syncs/progress/<doc_id>`
- **Update Progress**: `curl -v -X PUT http://localhost:8081/syncs/progress -H "Content-Type: application/json" -H "X-AUTH-USER: <user>" -H "X-AUTH-KEY: <md5_pass>" -H "Accept: application/vnd.koreader.v1+json" -d '{"document":"<doc_id>", "percentage":0.5, "progress":"page 10", "device_id":"123", "device":"kindle"}'`

## Docker
When the server is running in a docker container, use these commands from the host shell to check logs or pass through the CLI user management commands.

- **View Logs (Follow)**: `docker logs -f <app>`
- **CLI (Interactive Password)**: `docker exec -it <container> ./<app> create-user <user>`
- **CLI (Automation)**: `echo "<password>" | docker exec -i <container> ./<app> create-user <user> --password-stdin`
- **Restart Container**: `docker compose restart <app>`
- **Container Stats**: `docker stats`

## SQLite
From a shell, use these commands to verify the contents of the database after running any of the CLI user management commands, or any HTTP requests that causes changes in the database.

- **List All Users**: `sqlite3 data/<app>.db "SELECT username FROM users;"`
- **Check Progress (KOSYNC)**: `sqlite3 data/kosync.db "SELECT * FROM progress ORDER BY timestamp DESC LIMIT 10;"`
- **Check Book Count (KOPDS)**: `sqlite3 data/kopds.db "SELECT count(*) FROM books;"`
- **Check Sync Status (KOPDS)**: `sqlite3 data/kopds.db "SELECT * FROM sync_state;"`
- **Verify FTS5 Search (KOPDS)**: `sqlite3 data/kopds.db "SELECT title FROM books_search WHERE books_search MATCH 'Programming';"`
