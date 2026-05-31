---
name: roadmap-implementer
description: Implements roadmaps one step at a time with minimal context. Use for stepwise, phased, or checkpointed roadmap execution when a detailed planning document is the guide.
---

# Roadmap Implementer

Orchestrates implementation of roadmap steps by deploying a **step-executor subagent** for each step. Each subagent starts with a clean context containing only the current step's fields, ensuring zero context buildup across steps.

## When to Use This Skill

- When asked to implement changes specified in a detailed roadmap or checklist.
- When asked to implement changes in a stepwise manner.
- When asked to execute one roadmap step at a time with minimal context carryover.

## When Not to Use This Skill

- When detailed instructions have already been given regarding how to implement changes, those instructions take precedence.

## Relationship to Other Skills

| Skill                  | Role        | When to Use                                                                    |
| :--------------------: | :---------: | :----------------------------------------------------------------------------- |
| roadmap-writer         | Planner     | Writing or rewriting implementation roadmaps, but **not** implementing them    |
| roadmap-implementer    | Implementer | Implementing changes specified in roadmaps, but **not** troubleshooting issues |
| conventional-committer | Committer   | Committing changes to the repository                                           |

## Philosophy

Heavyweight models are used to perform large context audits or planning passes and output a detailed implementation roadmap as a durable artifact of the plan.  Lightweight models are then used to implement the plan in a stepwise manner requiring only minimal context.  The planning phase agents serve as senior advisors and architects, with a relatively wide latitude for exercising independent judgment, and the freedom to explore the workspace as needed.  The implementation phase agents serve as junior engineers, are expected to stop and ask for guidance if they cannot adhere strictly to the plan, and are expected to avoid broad reads or attempts to troubleshoot issues independently.  Implementation phase agents also commit changes, using committer skills if available.

Context efficiency is achieved structurally: the orchestrator (you) deploys a subagent per step, and the subagent's context is discarded when the step completes.  This eliminates the need for manual context compression, session resets, or handoff files.

## Principles

- **Orchestrator Role** — you are the orchestrator; you read the roadmap, deploy subagents, and report results.  You do not implement code changes directly.
- **Structural Isolation** — each step executes in a subagent with a clean context.  Context buildup across steps is impossible by design.
- **Linear Sequence** — implement roadmap steps in order, unless the user explicitly authorizes out-of-sequence execution.
- **Incremental Execution** — implement one step at a time, then stop and ask for permission to continue unless the user explicitly authorizes unattended multi-step execution.  If multi-step execution is authorized, the end point must be explicitly defined or you must ask for it.
- **Strict Adherence** — the subagent implements only the changes specified in the step; it does not troubleshoot or explore beyond the step's instructions.
- **Progress Truth** — never depend on conversation memory for what was done; rely only on roadmap annotations and git.

## Architecture

```
┌──────────────┐
│     User     │ 
└─┬──────────▲─┘
  │ deploy   │ report
┌─▼──────────┴────────────────────┐        
│  Orchestrator (you)             │        
│  • Reads roadmap                │
│  • Finds next incomplete step   │
│  • Builds subagent prompt       │
│  • Deploys step executor        │
│  • Reports results to user      │
└────┬────────────────▲───────────┘
     │ deploy         │ report
┌────▼────────────────┴─────────────────┐
│ Step Executor (subagent)              │ 
│  • Reads step files only              │
│  • Implements changes                 │
│  • Verifies                           │
│  • Updates roadmap annotation         │
│  • Commits via conventional-committer │
└───────────────────────────────────────┘
```

## Orchestrator Workflow

### 1. Bootstrap

Read **only**:

- The roadmap file
- `AGENTS.md` at the workspace root (for git rules and repo structure)
- `references/annotation-format.md` if the roadmap's checkbox format is unclear

Do **not** read implementation files — that is the subagent's job.

### 2. Assess Progress

Scan the roadmap for step states:

- If a `[-]` (work-in-progress) step exists, that is the target step — it was interrupted.
- Otherwise, the target is the first `[ ]` (incomplete) step.
- If no incomplete steps remain, report completion to the user and stop.

### 3. Extract Step Fields

From the target step, extract:

- **ID** and **goal** (the checkbox line)
- **Repos** — which git roots change
- **Read** — files to read for context
- **Edit** — files allowed to change
- **Instructions** — implementation instructions
- **Verify** — exact verification commands to run
- **Done when** — acceptance criteria

If the step lacks this schema, ask the user to rewrite it with **roadmap-writer** before proceeding.

### 4. Deploy Step Executor

Deploy a `self`-type subagent with the prompt constructed from the **Step Executor Prompt Template** below.  Pass:

- The extracted step fields
- The absolute path to the roadmap file
- The workspace root and `AGENTS.md` path
- Whether this is a resume (`[-]`) or a fresh start (`[ ]`)

Wait for the subagent to complete.

### 5. Process Results

Read the subagent's return message.  It will report one of:

| Status        | Orchestrator Action                                                  |
| :-----------: | :------------------------------------------------------------------- |
| **completed** | Record commit hash(es); proceed to step 6                            |
| **blocked**   | Surface the blocker to the user; do not retry or troubleshoot        |
| **failed**    | Surface the error to the user; do not retry or troubleshoot          |

If the subagent reports **blocked** or **failed**, stop and present the details to the user.  Do not attempt to fix the problem yourself.

### 6. Continue or Stop

- **Default**: stop after one step, report the result, and ask the user for permission to continue.
- **Multi-step authorized**: re-read the roadmap (the subagent updated annotations), then loop back to step 2.
- **Phase boundary**: if the completed step was the last in its phase, note the phase completion in your report.

---

## Step Executor Prompt Template

Use this template to construct the subagent prompt.  Replace placeholders (`{...}`) with values extracted from the roadmap step.

~~~
Implement a single roadmap step.  Follow these rules strictly.

## Step

- **ID:** {step_id}
- **Goal:** {step_goal}
- **Repos:** {repos}
- **Read:** {read_files}
- **Edit:** {edit_files}
- **Instructions:** {instructions}
- **Verify:** {verify_commands}
- **Done when:** {done_when}

## Status

{Either "This is a fresh start." or "This step was previously interrupted (marked [-]).  Before implementing, run the Verify commands and inspect `git status` and `git diff` to assess what was already done.  Continue from where it left off."}

## Roadmap File

Path: {absolute_path_to_roadmap}

## Rules

1. Read only the files listed under **Read** and **Edit** above.  Do not explore beyond these files.
2. Edit only the files listed under **Edit**.  Do not modify files outside this list.
3. If the instructions are ambiguous or you encounter an architectural blocker, stop immediately and report back with status **blocked** and a description of the problem.  Do not attempt to troubleshoot independently.
4. Run the **Verify** commands after implementing.  If verification fails, stop and report back with status **failed**, including the command output.

## Annotation

Before starting implementation:

1. If the step is not already marked `[-]`, update the roadmap file to change `[ ]` to `[-]` for step **{step_id}**.

After implementation and verification pass:

1. Update the roadmap file:
   - Change `[-]` to `[x]` for step **{step_id}**
   - Append the commit hash to the checkbox line: `(Commit: {hash})`
   - For multi-repo steps, use: `(Commits: {repo1}:{hash1}, {repo2}:{hash2})`

## Commit

Read and follow the `conventional-committer` skill instructions at `.agents/skills/conventional-committer/SKILL.md`.  Read the project git rules at {agents_md_path}.  If files span multiple repositories, commit separately in each repository.

## Report

When finished, report back with exactly these fields:

- **Step:** {step_id}
- **Status:** completed | blocked | failed
- **Commit(s):** repo:hash (or "none" if blocked/failed)
- **Files touched:** list of files modified
- **Verify result:** command + pass/fail
- **Next step:** the ID of the next `[ ]` step in the roadmap (or "none" if all complete)
- **Notes:** any observations relevant to future steps (optional)
~~~

---

## Resume Prompt Templates

Copy-paste for the user when starting a new session:

```text
Use roadmap-implementer on @{roadmap}.md — implement the next incomplete step only.
```

Variants:

- `Use roadmap-implementer on @{roadmap}.md — resume interrupted step` (when `[-]` exists)
- `Use roadmap-implementer on @{roadmap}.md — run Phase 3 unattended` (explicit multi-step authorization)

## Output

### User-facing summary

After each step (or when blocked/failed), report to the user:

- Step ID and status
- One-line outcome
- Commit hash(es)
- Suggested manual verification (if any)
- Whether there are remaining steps and the next step ID
- Resume prompt if suggesting a new session

### Blocker report

If the subagent reports blocked or failed:

- Step ID
- What failed or is ambiguous
- The subagent's error output or description
- Options for resolution (e.g., rewrite the step with roadmap-writer, fix the issue manually)

## References

- [`references/annotation-format.md`](references/annotation-format.md) — checkbox states and commit-hash audit trails
- [`../roadmap-writer/references/roadmap-template.md`](../roadmap-writer/references/roadmap-template.md) — step shape (Repos, Read, Edit, Verify; authored by roadmap-writer)

If the planning document does not use an established annotation convention, adopt `references/annotation-format.md`.
