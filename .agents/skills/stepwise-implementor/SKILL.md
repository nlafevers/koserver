---
name: stepwise-implementor
description: Implements roadmaps or checklists sequentially, stepwise, or one-step-at-a-time.  Uses a detailed planning document as a guide.
---

# Stepwise Implementor

Uses cost-efficient lightweight AI models/agents to implement changes when a detailed planning document is available as a guide.

## When to Use This Skill

- When asked to implement changes specified in a detailed roadmap or checklist.
- When asked to implement changes in a stepwise manner.

## When Not to Use This Skill

- When detailed instructions have already been given regarding how to implement changes, those instructions take precedence.

## Workflow

1. **Locate & Parse**: Ensure a detailed planning document with unique step identifiers (e.g., `TS-101`) is present. If absent, ask the user to provide or generate one before proceeding.
2. **Assess Progress**: Review the roadmap for completed `[x]`, in-progress `[-]`, and incomplete `[ ]` steps. 
   - *If an in-progress `[-]` step is found*, assume a previous execution was interrupted. Assess the current state of the codebase, run tests, and resume this step.
   - *Otherwise*, identify the very first incomplete `[ ]` step.
3. **Plan & Mark WIP**: 
   - Formulate a specific implementation plan for this single step. 
   - If instructions are ambiguous or you foresee architectural blockers, stop and ask the user for clarification.
   - Once ready to proceed, update the roadmap file immediately by changing that specific step's checkbox to `[-]`.
4. **Implement**: Execute the code changes required *only* for this specific step. Do not modify code relevant to future steps.
5. **Verify (Definition of Done)**: Run relevant linters, compilers, or test suites to verify your changes. **NEVER** mark a step complete if the build is broken or tests fail.
6. **Mark Complete**: Update the roadmap file a second time, changing the specific step identifier from `[-]` to `[x]`.
7. **Commit Changes**: Invoke the `git-committer` skill to commit the implementation if available, otherwise do the following:
   - Identify the repository root for each file modified by substituting its path into this command: `git -C "$(dirname "/path/to/modified/file")" rev-parse --show-toplevel`
   - If the modified files span multiple repositories, repeat **Step 7. Commit Changes** for each repository modified.    
   - Navigate to the repository root explicitly for each git command (e.g., chaining `cd <path-to-repo-root> && git <command>`) to avoid directory drift.
   - Stage only the files modified in the current implementation step.
   - Create a commit using the Conventional Commits format: `type(scope): subject`. Include a body for complex changes, referencing any related issues, documentation, and unique step identifiers.
8. **Budget & Boundary Check**: 
   - Evaluate your remaining context window and CLI turn limits. 
   - Stop and provide a progress report to the user. Ask explicitly for permission to proceed to the next step, unless the user explicitly authorized fully automated execution of the entire phase.

## Output

- **Interruption/Blocker Report**: If reporting a problem or requesting clarification, provide the unique step ID, details about the roadblock, and options for resolution.
- **Step/Phase Completion Report**: After completing a step (or phase if authorized to run automatically), provide a concise summary containing:
  - The unique step ID(s) completed.
  - Specific files changed.
  - Test/verification results.
  - The resulting Git commit hash(es) and repository information.
  - Suggested manual verification step(s) for the user.

## References

- If the planning document does not already use an established convention for annotating work-in-progress and work-completed, use the format in `references/annotation-format.md`.
