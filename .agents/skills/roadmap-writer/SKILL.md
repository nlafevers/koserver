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

- When asked to **implement** changes specified in a roadmap — use **roadmap-implementer**.
- When the user only wants a one-off code change without a planning document.

## Relationship to Other Skills

| Skill                  | Role        | When to Use                                                                    |
| :--------------------: | :---------: | :----------------------------------------------------------------------------- |
| roadmap-writer         | Planner     | Writing or rewriting implementation roadmaps, but **not** implementing them    |
| roadmap-implementer    | Implementer | Implementing changes specified in roadmaps, but **not** troubleshooting issues |
| conventional-committer | Committer   | Committing changes to the repository                                           |

## Philosophy

Heavyweight models are used to perform large context audits or planning passes and output a detailed implementation roadmap as a durable artifact of the plan.  Lightweight models are then used to implement the plan in a stepwise manner requiring only minimal context.  The planning phase agents serve as senior advisors and architects, with a relatively wide latitude for exercising independent judgment, and the freedom to explore the workspace as needed.  The implementation phase agents serve as junior engineers, are expected to stop and ask for guidance if they cannot adhere strictly to the plan, and are expected to avoid broad reads or attempts to troubleshoot issues independently.  Implementation phase agents also commit changes, using committer skills if available.

## Principles

- **Roadmap-Only Edits** — planning changes to the codebase is strictly separated from implementation; edit only the roadmap file.
- **Durable Plans** — capture all audit findings, planning decisions, and other results of the planning pass in the roadmap document; do not rely on the chat history or memory.
- **Thinking Upfront** — the planning phase is the time for deep thinking with a large context; the steps of the implementation plan should be self-contained and require minimal context or thinking.
- **Self-contained Steps** — each step should be implementable in isolation; avoid carryover reads or references to other steps.
- **Progress Annotation** - author new steps as `[ ]` only; do not set `[-]` or `[x]` unless **preserving** completed work during a rewrite.  The git log and the roadmap annotations are the durable source of truth for what has been done, not the chat history or memory.

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
- `../roadmap-implementer/references/annotation-format.md` — checkbox and audit-trail format

### 3. Audit (if needed)

When building from findings rather than a supplied task list:

- Perform one structured audit pass; record bullets under **Audit findings to address**
- Cap exploratory reads where possible; prefer targeted files over whole-repo grep unless necessary

### 4. Outline Phases

- Assign phase numbers and one-sentence **goals**
- Define **acceptance criteria** per phase (testable bullets)
- Assign step ID prefixes (e.g. `UR2-3.1`) unique across the document

### 5. Author Steps

For each atomic step, apply Part 2 of [`references/roadmap-template.md`](references/roadmap-template.md):

- **Repos**, **Read** (≤5), **Edit** (≤5), **Verify** (exact commands), **Done when**
- Split steps that exceed the file budget or mix unrelated concerns
- Convert legacy lists (e.g. `1. Open ... 2. Run ...`) into the template fields

### 6. Assemble Document

Follow Part 1 of [`references/roadmap-template.md`](references/roadmap-template.md):

- Title, git rules, how-to-use, optional audit findings, phases, optional final verification, notes for implementers

### 7. Conform Checkboxes

Use [`../roadmap-implementer/references/annotation-format.md`](../roadmap-implementer/references/annotation-format.md):

- New work: `[ ]` with `**ID**` in the line
- Preserved completed work: keep `[x]` and `(Commit: hash)` as-is

### 8. Validate

Run the author checklist in [`references/roadmap-template.md`](references/roadmap-template.md):

- Every step stands alone (no prohibited carryover phrasing)
- Every step has **Verify** and **Done when**
- File lists within budget or step is split

### 9. Output

- Write or update the roadmap markdown only
- Report path, phase/step counts, and any deferred legacy formatting
- Provide the suggested roadmap-implementer resume prompt

## Output Format

Deliver to the user:

1. **Path** — e.g. `ur2-roadmap.md`
2. **Summary** — phases and step counts; new vs rewritten sections
3. **Legacy notes** — any steps intentionally left in old format (should be none after a full rewrite)
4. **Next action** — `Use roadmap-implementer on @<roadmap> — implement the next incomplete step only.`

## References

- [`references/roadmap-template.md`](references/roadmap-template.md) — full template: document structure, per-step fields, legacy migration, and author checklist
- [`../roadmap-implementer/references/annotation-format.md`](../roadmap-implementer/references/annotation-format.md) — `[ ]` / `[-]` / `[x]` and commit hashes
