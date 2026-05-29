# Roadmap Template

Use this file when authoring implementation roadmaps consumed by **roadmap-implementer**. It covers both the **top-level document structure** and the **per-step format**.

---

## Part 1: Document Structure

A roadmap document must contain the following sections in order.

### 1. Title and purpose

```markdown
# <Project> Implementation Roadmap

One paragraph: what this roadmap achieves and why it exists.
```

### 2. Workspace and git rules

Summarize multi-repo boundaries if present. Point to project `AGENTS.md` rather than duplicating full policy.

```markdown
KOSERVER is documentation only. KOPDS and KOSYNC are separate Git repositories.
Code changes in `kopds/` commit in the KOPDS repo; `kosync/` in KOSYNC; root docs in KOSERVER.
```

### 3. How to use this roadmap

Tooling conventions implementors need on every step (copy-paste commands):

```markdown
## How To Use This Roadmap

When running Go commands, prefer a writable cache under `/tmp`:
\`\`\`bash
GOCACHE=/tmp/kopds-gocache go test ./...
GOCACHE=/tmp/kosync-gocache go test ./...
\`\`\`

Do not copy commits across repositories.
```

### 4. Audit findings to address (optional)

Bullets from a planning or audit pass. Helps implementors understand *why* phases exist. Omit for greenfield roadmaps with no prior audit.

```markdown
## Audit Findings To Address

- Finding one
- Finding two
```

### 5. Phases

Each phase is a `## Phase N: <Title>` section containing a **Goal**, **Steps**, and **Acceptance criteria**.

```markdown
## Phase 3: Example Phase

Goal: one sentence describing the phase outcome.

- [ ] **EX-3.1** Short step title
  - **Repos:** kopds
  - **Read:** path/to/file.go
  - **Edit:** path/to/file.go
  - **Verify:** `go test ./...`
  - **Done when:** tests pass

Acceptance criteria for Phase 3:

- Criterion one
- Criterion two
```

### 6. Final verification (optional)

A closing phase or section with repo-wide checks run only after earlier phases complete.

```markdown
## Phase 9: Final Verification Checklist

Run only after all phases are done.

- [ ] **FINAL-9.1** ...
```

### 7. Notes for future implementers

Document intentional differences, deferred work, and decisions that steps should not undo.

```markdown
## Notes For Future Implementers

- Intentional difference between apps: ...
- Do not reintroduce deprecated pattern X
```

---

## Part 2: Per-Step Format

Each step must stand alone. An agent bootstrapping from the roadmap and the listed files should not need prior conversation or earlier steps.

### Step Fields

| Field         | Required    | Purpose                                        |
| :-----------: | :---------: | :--------------------------------------------- |
| **ID**        | Yes         | Unique, grep-safe identifier (e.g., `UR2-3.2`) |
| **Goal**      | Yes         | One sentence describing the outcome            |
| **Repos**     | Yes         | Which git roots change                         |
| **Read**      | Recommended | Files to read for context (cap ~5)             |
| **Edit**      | Recommended | Files allowed to change (cap ~5)               |
| **Verify**    | Yes         | Exact commands to run (copy-paste ready)       |
| **Done when** | Yes         | Acceptance criteria                            |

### File Budget

- **Read:** at most ~5 files per step
- **Edit:** at most ~5 files per step
- If more files are needed, split into multiple steps with sequential IDs

### ID Naming

Use a stable prefix per roadmap (e.g., `UR2-3.2`, `INIT-01`). IDs must be unique and grep-safe for checkbox updates.

### Checkbox States

Author new work as `[ ]` only. Use annotation rules from `../roadmap-implementer/references/annotation-format.md`:

- `[ ]` uncompleted
- `[-]` work in progress (set by implementor, not author)
- `[x]` completed with optional `(Commit: hash)` (set by implementor)

When rewriting, preserve existing `[x]` and commit hashes; do not reset completed steps unless the user asks.

### Prohibited Phrasing

Do not rely on conversation carryover. Avoid:

- "See previous step"
- "As above"
- "Continue from last time"
- "Same as UR2-3.1" without repeating paths and commands

Repeat paths, repos, and verify commands in every step that needs them.

### Example Step

```markdown
- [ ] **UR2-3.2** Add KOPDS storage user methods
  - **Repos:** kopds
  - **Read:** kopds/internal/database/user_repository.go
  - **Edit:** kopds/internal/database/sqlite.go, kopds/internal/database/user_repository.go
  - **Verify:** `GOCACHE=/tmp/kopds-gocache go test ./internal/database ./cmd/kopds`
  - **Done when:** tests pass; `domain.UserRepository` interface unchanged
```

### Multi-Repository Steps

When a single logical step touches more than one repo, list **Repos** and group **Edit** paths by repo. The implementor commits separately in each repo (see project `AGENTS.md`).

```markdown
- [ ] **UR2-2.3** Fix Docker publish workflows
  - **Repos:** kopds, kosync
  - **Read:** kopds/.github/workflows/docker-publish.yml, kosync/.github/workflows/docker-publish.yml
  - **Edit:** kopds/.github/workflows/docker-publish.yml, kosync/.github/workflows/docker-publish.yml
  - **Verify:** (visual) `file: build/Dockerfile` present under `docker/build-push-action` in both workflows
  - **Done when:** both workflows specify `file: build/Dockerfile`; one commit per repo
```

---

## Part 3: Legacy Format Migration

When rewriting an existing roadmap that uses numbered substeps:

```markdown
1. Open `/path/to/file.go`.
2. Run `go test ./...`.
```

**Convert** each step to the self-contained shape:

- Numbered "open/read" lines → **Read** and **Edit** file lists
- "Run" lines → **Verify** (exact command)
- Implicit success conditions → **Done when**

Do not leave "see previous step" or "as above" — repeat paths and commands in every step.

---

## Author Checklist

- [ ] Document sections 1–3 present (title/purpose, git rules, how-to-use)
- [ ] Every phase has a goal and acceptance criteria
- [ ] Every actionable step has a unique, grep-safe ID
- [ ] Every step lists **Repos**, **Verify**, and **Done when**
- [ ] File lists stay within budget or the step is split
- [ ] No step depends on unread chat history or prior steps
- [ ] No legacy numbered substeps remain (unless explicitly deferred)
- [ ] New steps use `[ ]`; preserved completed steps keep `[x]` and commit hashes
- [ ] Implementor can run with: `Use roadmap-implementer on @<this-file> — implement the next incomplete step only.`
