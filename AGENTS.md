# KOSERVER - Lightweight Dual-Purpose Server for KOReader Clients

KOSERVER itself is not an application, but rather a repository for documentation.  Within the KOSERVER root directory are directories for two applications which are tracked in their own separate git repositories: KOPDS and KOSYNC.  These apps serve KOReader e-book reader clients.  KOPDS is an OPDS (Open Publication Distribution System) server.  KOSYNC is a reading progress synchronization server.  The documentation in KOSERVER will serve two purposes.  First, document the ongoing work to bring harmonization of functionality and uniformity of code to both KOPDS and KOSYNC.  Second, document the process and tools needed to deploy these two application to a shared machine or cloud VM.

## Uniformity
See `uniformity-plan.md` for the overall plan to bring uniformity to the twin apps.  See `kopds/UNIFORMITY.md` for uniformity information specific to KOPDS.  See `kosync/UNIFORMITY.md` for uniformity information specific to KOSYNC.

## Deployment
There are no documents yet related to deployment.  This section will need to be updated as deployment documentation is developed.

## Git
Changes to documentation files in the KOSERVER project root must be commited to the KOSERVER repository.  Any KOPDS or KOSYNC code or documentation changes must be commited to their own separate repositories.

## KOPDS Project Context
@./kopds/AGENTS.md

## KOSYNC Project Context
@./kosync/AGENTS.md