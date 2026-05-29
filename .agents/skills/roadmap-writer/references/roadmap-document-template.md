# Roadmap Document Template

Use this template for the **top-level structure** of an implementation roadmap. Combine with `roadmap-step-template.md` for individual steps.

## Required Sections

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

Bullets from a planning or audit pass. Helps implementors understand *why* phases exist.

```markdown
## Audit Findings To Address

- Finding one
- Finding two
```

Omit this section for greenfield roadmaps with no prior audit.

### 5. Phases

Each phase is a `## Phase N: <Title>` section containing:

- **Goal** — one sentence
- **Steps** — checklist items per `roadmap-step-template.md`
- **Acceptance criteria for Phase N** — bullets testable at phase end

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

## Legacy format migration

When rewriting an existing roadmap that uses numbered substeps:

```markdown
1. Open `/path/to/file.go`.
2. Run `go test ./...`.
```

**Convert** each step to the self-contained shape:

- Numbered “open/read” lines → **Read** and **Edit** file lists
- “Run” lines → **Verify** (exact command)
- Implicit success conditions → **Done when**

Do not leave “see previous step” or “as above” — repeat paths and commands in every step.

## ID naming

Use a stable prefix per roadmap (e.g. `UR2-3.2`, `INIT-01`). IDs must be unique and grep-safe for checkbox updates.

## Checkbox states

Author new work as `[ ]` only. Use annotation rules from `../stepwise-implementor/references/annotation-format.md`:

- `[ ]` uncompleted
- `[-]` work in progress (set by implementor, not author)
- `[x]` completed with optional `(Commit: hash)` (set by implementor)

When rewriting, preserve existing `[x]` and commit hashes; do not reset completed steps unless the user asks.

## Author checklist

- [ ] Document sections 1–3 present (git rules, how-to-use)
- [ ] Every phase has a goal and acceptance criteria
- [ ] Every step follows `roadmap-step-template.md`
- [ ] No legacy numbered substeps remain (unless explicitly deferred)
- [ ] Implementor can run with: `Use stepwise-implementor on @<this-file> — implement the next incomplete step only.`
