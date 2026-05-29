---
name: stepwise-implementor
description: Implements roadmaps one step at a time with minimal context. Use for stepwise, phased, or checkpointed roadmap execution when a detailed planning document is the guide.
---

# Stepwise Implementor

Uses cost-efficient, lightweight AI models/agents to implement changes when a detailed planning document is available as a guide.

## When to Use This Skill

- When asked to implement changes specified in a detailed roadmap or checklist.
- When asked to implement changes in a stepwise manner.
- When asked to execute one roadmap step at a time with minimal context carryover.

## When Not to Use This Skill

- When detailed instructions have already been given regarding how to implement changes, those instructions take precedence.

## Context Management Principles

Agents cannot programmatically clear conversation context mid-session. Use these rules instead:

- **Compress** — summarize the finished step into a short **Step Handoff** block (see below).
- **Clear** — recommend a **new Agent session** and bootstrap from the roadmap only.

Core rules:

- **One step per invocation** — complete workflow steps 1–9 for a single step ID, then stop unless the user explicitly authorized unattended multi-step execution of a phase.
- **Roadmap + git are source of truth** — never depend on conversation memory for what was done; read `[x]`/`[-]` and commit hashes from the roadmap file.
- **No carryover reads** — do not re-read files from a previous step unless the *current* step lists them under **Read** or **Edit**.
- **No speculative exploration** — no repo-wide search or broad reads beyond the step's file list and required project docs (`AGENTS.md`, repo-root rules).
- **Durable > conversational** — if a fact must survive to the next step, it belongs in the roadmap (checkbox, commit hash) or the codebase, not in chat.

Roadmap steps must be self-contained. See `../roadmap-writer/references/roadmap-step-template.md` (authored by **roadmap-writer**).

## Workflow

### 0. Bootstrap (start of every step — cold start)

1. Treat the user instruction to use this skill on a roadmap as the standing goal.
2. Read **only**:
   - The roadmap file (or the target phase section + the single target step)
   - `references/annotation-format.md` if checkbox format is unclear
   - Project `AGENTS.md` / git-repo rules (paths, multi-repo commit rules)
   - Files listed under the current step's **Read** / **Edit** lists (if present)
   - If the current step lacks **Read** / **Edit** lists, ask the user to rewrite it with **roadmap-writer** before implementing
3. **Do not** consult prior assistant messages, prior step plans, or earlier tool results — except the **Step Handoff** block in the *immediately previous* assistant turn (if continuing in the same thread).
4. If resuming `[-]`: run the step's verification commands, inspect `git status` and `git diff`, then continue implementation without re-deriving context from old chat.

### 1. Locate & Parse

Ensure a detailed planning document with unique step identifiers (e.g., `UR2-3.2`) is present. If absent, ask the user to provide or generate one before proceeding.

### 2. Assess Progress

Review the roadmap for completed `[x]`, in-progress `[-]`, and incomplete `[ ]` steps.

- *If an in-progress `[-]` step is found*, assume a previous execution was interrupted. Use bootstrap rules above (git + step file list), not chat history.
- *Otherwise*, identify the very first incomplete `[ ]` step.

### 3. Plan & Mark WIP

- Formulate a specific implementation plan for this single step.
- If instructions are ambiguous or you foresee architectural blockers, stop and ask the user for clarification.
- Once ready to proceed, update the roadmap file immediately by changing that specific step's checkbox to `[-]`.

### 4. Implement

Execute the code changes required *only* for this specific step. Do not modify code relevant to future steps. Touch only files listed under the step's **Edit** list (or implied by the step if the roadmap predates the template).

### 5. Verify (Definition of Done)

Run relevant linters, compilers, or test suites listed in the step's **Verify** / **Commands** section. **NEVER** mark a step complete if the build is broken or tests fail.

### 6. Mark Complete

Update the roadmap file a second time, changing the specific step identifier from `[-]` to `[x]`. Append the commit hash as an audit trail; see `references/annotation-format.md` for details.

### 7. Commit Changes

Invoke the `git-committer` skill to commit the implementation if available, otherwise:

- Identify the repository root for each file modified: `git -C "$(dirname "/path/to/modified/file")" rev-parse --show-toplevel`
- If modified files span multiple repositories, repeat commit workflow for each repository.
- Navigate to each repository root explicitly for git commands (e.g., `cd <path-to-repo-root> && git <command>`) to avoid directory drift.
- Stage only files modified in the current implementation step.
- Create a commit using Conventional Commits: `type(scope): subject`. Include a body for complex changes, referencing the unique step identifier.

### 8. Teardown (end of every step — context boundary)

1. Emit a **Step Handoff** block (required schema below).
2. **In-session compression** (if continuing in the same thread): state explicitly that the next step must ignore all prior tool output and exploration, and bootstrap from the next roadmap step ID only.
3. **Session reset** (default for multi-step roadmaps): recommend starting a **new Agent conversation** with a resume prompt from the templates below.
4. **Budget & boundary check**:
   - Default: stop after one step and ask permission to continue, unless the user explicitly authorized unattended execution of an entire phase.
   - Unattended phase: still emit Step Handoff after each step; still read only files listed in each step.
   - If context usage is high (when visible) or the thread is long: strongly recommend a new session before the next step.

### Optional: handoff file (tier 2)

For very long roadmaps, you may overwrite `.cursor/stepwise-handoff.md` (gitignored) with the same fields as the Step Handoff block. Next session: read the roadmap + that file only. Do not require this for normal use.

## Step Handoff Format

After each completed, blocked, or paused step, append this block (max ~12 lines; no code dumps):

```markdown
## Step Handoff
- **Roadmap:** path/to/roadmap.md
- **Step:** UR2-3.2
- **Status:** completed | blocked | wip
- **Commit(s):** kopds:abc1234 | kosync:— | koserver:—
- **Files touched:** (max 10 paths)
- **Verify:** command + pass/fail
- **Next step:** UR2-3.3 (first `[ ]` in phase)
- **Resume prompt:** Use stepwise-implementor on @path/to/roadmap.md
```

This is the **only** in-chat artifact the next in-thread step may trust besides the roadmap file itself.

## Resume Prompt Templates

Copy-paste for the user when starting a new session:

```text
Use stepwise-implementor on @ur2-roadmap.md — implement the next incomplete step only.
```

Variants:

- `Use stepwise-implementor on @ur2-roadmap.md — resume interrupted step` (when `[-]` exists)
- `Use stepwise-implementor on @ur2-roadmap.md — run Phase 3 unattended` (explicit multi-step authorization)

## Output

### Step Handoff

Required after every step (see schema above).

### User-facing summary

Keep brief; defer detail to the commit and roadmap checkbox:

- Step ID and status
- One-line outcome
- Pointer to commit hash(es) in the roadmap line
- Suggested manual verification (if any)
- Whether to start a new session or continue

### Interruption / blocker report

If blocked or awaiting clarification:

- Unique step ID
- What failed or is ambiguous
- Options for resolution
- Step Handoff with `Status: blocked` or `wip`

## References

- `references/annotation-format.md` — checkbox states and commit-hash audit trails
- `../roadmap-writer/references/roadmap-step-template.md` — step shape (Repos, Read, Edit, Verify; authored by roadmap-writer)

If the planning document does not use an established annotation convention, adopt `references/annotation-format.md`.
