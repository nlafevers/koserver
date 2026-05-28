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

- Ensure a detailed planning document with a roadmap or checklist is specified to guide the changes, and if absent, request one.
- Review the roadmap or checklist for annotations that indicate how much progress has been made already, and note conventions used for indicating work-in-progress versus work-completed.
- Begin with the first uncompleted step:
    1. Review the instructions for that step, and plan how you will implement them.  If you foresee any problems, or need any clarification, stop and report the issue, and when possible provide options for moving forward.
    2. When ready to proceed, annotate the checkbox for that step to indicate work in progress (use the project convention if already established, otherwise `[-]`).
    3. Implement the changes as you planned to, modified according to any feedback you received.
    4. When finished implementing the changes for that step, annotate the checkbox for that step to indicate work completed (use the project convention if already established, otherwise `[x]`).
    5. Change to the parent directory of the repository in which you made changes.
    6. Stage the files you made changes to if they are not listed in `.gitignore`.
    7. Commit the changes using the `git-commit` skill if available, otherwise use conventional commit format.
- Proceed to the next uncompleted step within the same phase following the same seven step procedure specified above.
- Unless specifically told to proceed through multiple phases, stop when all uncompleted steps in a phase have been completed, and provide a report on the work completed.
- If told to proceed through multiple phases, stop when all uncompleted steps have been completed in the specified phases, and provide a report on the work completed.

## Output

- If reporting a problem, provide details about the problem and provide options for resolution if possible.
- If requesting clarification, report why you found the instructions to be deficient, and suggest what information you need to resolve the deficiency.
- If reporting work completed, provide a summary of the steps completed, the changes made for each step, any tests conducted, any documentation updated, any commits made, and suggest a verification process to the user in case they would like to independently check your work.

## References

- If the planning document does not already use an established convention for annotating work-in-progress and work-completed, use the format in `references/annotation-format.md`.
