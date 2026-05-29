# To Do List

The next round of work on KOSERVER, KOPDS, and KOSYNC is currently spelled out in:
@./ur2-roadmap.md

- [x] **TDL-001 Storage Lifecycle Wrappers**: Supersedes UR2-3.1. Standardized on KOPDS separation of concerns: `OpenSQLite(path, allowCreate)`, `Migrate(db)`, then inject dependencies (`NewStorage`, repositories). Removed KOSYNC `InitDB` and `Storage.Close()`. Both apps use `NewSQLite(path, allowCreate)` as a thin wrapper. KOSYNC server and CLI wired to explicit open/migrate/inject; KOPDS gained `allowCreate` on `NewSQLite`.