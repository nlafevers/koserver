# KOSERVER - Lightweight Dual-Purpose Server for KOReader Clients

KOSERVER itself is not an application, but rather a repository for documentation.  Within the KOSERVER root directory are directories for two applications which are tracked in their own separate git repositories: KOPDS and KOSYNC.  These apps are served to KOReader e-book reader clients.  KOPDS is an OPDS (Open Publication Distribution System) server.  KOSYNC is a reading progress synchronization server.  The documentation in KOSERVER will serve two purposes.  First, document the ongoing development of both KOPDS and KOSYNC.  Second, document the process and tools needed to deploy these two application to a shared machine or cloud VM.

## KOPDS and KOSYNC Development
Next steps for the development of KOPDS and KOSYNC are found here: @./todo.md

## Deployment
The deployment guide for running KOPDS and KOSYNC together — native (systemd) and Docker, behind a Caddy reverse proxy with automatic HTTPS, including a free-tier GCP VM walkthrough — is here: @./deployment-guide.md

Copy-paste-ready sample configuration (Caddyfile, combined docker-compose, systemd units) lives in `./deploy/`. These deployment samples reference both app images/binaries, so they belong to the KOSERVER repository rather than either app's repository.

## Git
Changes to documentation files in the KOSERVER project root must be committed to the KOSERVER repository.  Any changes to code or documentation of either KOPDS or KOSYNC must be committed to their own separate repositories.

## KOPDS Project Context
@./kopds/AGENTS.md

## KOSYNC Project Context
@./kosync/AGENTS.md

## History
The history of the joint development of KOPDS and KOSYNC is found here: @./project-history.md