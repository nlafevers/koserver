---
name: roadmap-writer
description: Authors implementation roadmaps as self-contained steps. Use when creating, auditing, or rewriting a phased checklist roadmap.
---

# Roadmap Writer

Writes or rewrites implementation roadmaps as sequential, self-contained steps so they can be implemented with minimal context per step.

## When to Use This Skill

- When asked to generate a new implementation roadmap for a project.
- When asked to rewrite an existing roadmap to improve clarity, structure, task granularity, or context efficiency.
- When asked to migrate legacy numbered substeps to **Read / Edit / Verify / Done when** format.
- When asked to audit a codebase once and capture findings in a structured roadmap.

## When Not to Use This Skill

- When asked to **implement** changes specified in a roadmap — use **stepwise-implementor**.
- When the user only wants a one-off code change without a planning document.

## Relationship to stepwise-implementor

| Role            | Skill                                                                  |
| :-------------: | :--------------------------------------------------------------------- |
| **Planner**     | roadmap-writer — produces or updates `*.md` roadmaps                   |
| **Implementor** | stepwise-implementor — implements changes specified in `*.md` roadmaps |

Rules:

- Roadmaps must be implementable without chat carryover (aligned with implementor context principles).
- Author new steps as `[ ]` only. Do not set `[-]` or `[x]` unless **preserving** completed work during a rewrite.
- After delivering a roadmap, suggest: `Use stepwise-implementor on @<roadmap> — implement the next incomplete step only.`

## Philosophy

Heavyweight models may perform a broad audit or planning pass once. The roadmap file becomes the durable artifact; lightweight implementors then execute steps with limited reads per step. Never author steps that require “remember what we discussed” or “see previous step.” Split work until each step fits the file budget in `references/roadmap-step-template.md`.

## Principles

- **Roadmap-only edits** — planning changes to the codebase is strictly separated from implementation; edit only the roadmap file.
- **Durable plans** — a full codebase audit may require some back and forth with the user, but always keep the roadmap markdown updated at each turn.
- **Phase-scoped rewrites** — when updating one phase, read that phase’s context plus files needed to fill **Read** / **Edit** lists; avoid whole-repo re-audit unless requested.
- **Targeted context/token usage** — the infrequent audit or planning pass can be relatively unconstrained in reads, but each step must be implementable with a contrained number of reads and edits.

## Workflow

### 1. Intake

Clarify (or infer from the user message):

- Goal and target file path (e.g. `ur2-roadmap.md`)
- Scope: full roadmap, single phase, or append new phase
- Whether to preserve completed `[x]` steps and commit hashes in a rewrite

### 2. Gather Constraints

Read as needed:

- Project `AGENTS.md` at the workspace root
- Existing roadmaps, `todo.md`, or user decisions (e.g. superseded steps)
- `../stepwise-implementor/references/annotation-format.md` — checkbox and audit-trail format

### 3. Audit (if needed)

When building from findings rather than a supplied task list:

- Perform one structured audit pass; record bullets under **Audit findings to address**
- Cap exploratory reads where possible; prefer targeted files over whole-repo grep unless necessary

### 4. Outline Phases

- Assign phase numbers and one-sentence **goals**
- Define **acceptance criteria** per phase (testable bullets)
- Assign step ID prefixes (e.g. `UR2-3.1`) unique across the document

### 5. Author Steps

For each atomic step, apply [`references/roadmap-step-template.md`](references/roadmap-step-template.md):

- **Repos**, **Read** (≤5), **Edit** (≤5), **Verify** (exact commands), **Done when**
- Split steps that exceed the file budget or mix unrelated concerns
- Convert legacy lists (e.g. `1. Open ... 2. Run ...`) into the template fields

### 6. Assemble Document

Follow [`references/roadmap-document-template.md`](references/roadmap-document-template.md):

- Title, git rules, how-to-use, optional audit findings, phases, optional final verification, notes for implementers

### 7. Conform Checkboxes

Use [`../stepwise-implementor/references/annotation-format.md`](../stepwise-implementor/references/annotation-format.md):

- New work: `[ ]` with `**ID**` in the line
- Preserved completed work: keep `[x]` and `(Commit: hash)` as-is

### 8. Validate

Run the checklists in both reference files:

- Every step stands alone (no prohibited carryover phrasing)
- Every step has **Verify** and **Done when**
- File lists within budget or step is split

### 9. Output

- Write or update the roadmap markdown only
- Report path, phase/step counts, and any deferred legacy formatting
- Provide the suggested stepwise-implementor resume prompt

## Output Format

Deliver to the user:

1. **Path** — e.g. `ur2-roadmap.md`
2. **Summary** — phases and step counts; new vs rewritten sections
3. **Legacy notes** — any steps intentionally left in old format (should be none after a full rewrite)
4. **Next action** — `Use stepwise-implementor on @<roadmap> — implement the next incomplete step only.`

## References

- [`references/roadmap-step-template.md`](references/roadmap-step-template.md) — per-step Repos, Read, Edit, Verify, Done when
- [`references/roadmap-document-template.md`](references/roadmap-document-template.md) — top-level roadmap sections
- [`../stepwise-implementor/references/annotation-format.md`](../stepwise-implementor/references/annotation-format.md) — `[ ]` / `[-]` / `[x]` and commit hashes
