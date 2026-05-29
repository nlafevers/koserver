# Roadmap Step Template

Use this template when authoring implementation roadmaps consumed by **stepwise-implementor**. Each step must stand alone: an agent bootstrapping from the roadmap and listed files should not need prior conversation or earlier steps.

## Per-Step Fields

| Field         | Required    | Purpose                                        |
| :-----------: | :---------: | :--------------------------------------------- |
| **ID**        | Yes         | Unique, grep-safe identifier (e.g., `UR2-3.2`) |
| **Goal**      | Yes         | One sentence describing the outcome            |
| **Repos**     | Yes         | Which git roots change                         |
| **Read**      | Recommended | Files to read for context (cap ~5)             |
| **Edit**      | Recommended | Files allowed to change (cap ~5)               |
| **Verify**    | Yes         | Exact commands to run (copy-paste ready)       |
| **Done when** | Yes         | Acceptance criteria                            |

## File Budget

- **Read:** at most ~5 files per step
- **Edit:** at most ~5 files per step
- If more files are needed, split into multiple steps with sequential IDs

## Prohibited Phrasing

Do not rely on conversation carryover:

- "See previous step"
- "As above"
- "Continue from last time"
- "Same as UR2-3.1" without repeating paths and commands

Repeat paths, repos, and verify commands in every step that needs them.

## Example Step

```markdown
- [ ] **UR2-3.2** Add KOPDS storage user methods
  - **Repos:** kopds
  - **Read:** kopds/internal/database/user_repository.go
  - **Edit:** kopds/internal/database/sqlite.go, kopds/internal/database/user_repository.go
  - **Verify:** `GOCACHE=/tmp/kopds-gocache go test ./internal/database ./cmd/kopds`
  - **Done when:** tests pass; `domain.UserRepository` interface unchanged
```

## Example Phase Header

```markdown
## Phase 3: Uniform CLI And Database Lifecycle

Goal: make CLI user-management code as identical as practical across KOPDS and KOSYNC.

Acceptance criteria for Phase 3:
- CLI user-management functions are copy-paste similar
- Both apps still create missing databases from CLI commands
- Duplicate `create-user` still fails from the CLI in both apps
```

## Multi-Repository Steps

When a single logical step touches more than one repo, list **Repos** and group **Edit** paths by repo. The implementor commits separately in each repo (see project `AGENTS.md`).

```markdown
- [ ] **UR2-2.3** Fix Docker publish workflows
  - **Repos:** kopds, kosync
  - **Read:** kopds/.github/workflows/docker-publish.yml, kosync/.github/workflows/docker-publish.yml
  - **Edit:** kopds/.github/workflows/docker-publish.yml, kosync/.github/workflows/docker-publish.yml
  - **Verify:** (visual) `file: build/Dockerfile` present under `docker/build-push-action` in both workflows
  - **Done when:** both workflows specify `file: build/Dockerfile`; one commit per repo
```

## Checklist for Roadmap Authors

- [ ] Every actionable step has a unique ID
- [ ] Every step lists **Verify** commands
- [ ] File lists stay within budget or the step is split
- [ ] No step depends on unread chat history
- [ ] Interrupted work uses `[-]`; resume is via git state + this step's lists, not prior messages
