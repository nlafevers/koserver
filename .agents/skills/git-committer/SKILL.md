---
name: git-committer
description: Creates git commits following Conventional Commits format with type/scope/subject. Use when user wants to commit changes, create commit, save work, or stage and commit. Handles regular branch commits (development) and merge commits (PR closure). Enforces project-specific conventions from CLAUDE.md.
---

# Git Committer

Creates git commits following Conventional Commits format with proper type, scope, and subject.

## Project-Specific Conventions

First, always check for project-specific commit conventions by searching for "commit" in `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, or similar files.  If the project specifies a commit format, then that format takes precedence over the default format described below.

## Workflow

1. Identify the root directory of the repository in which you made changes.  Some workspaces may have nested repositories with either loose or defined submodules.  If you aren't sure, check `AGENTS.md`, or similar files, in the workspace root to learn about the git repository structure.  If you made changes to files in multiple repositories, you will need to create a separate commit for each repository, starting this workflow over at Step 1 for each commit.
    ```bash
    export REPO_DIR=$(git -C "$(dirname "<absolute-path-to-file>")" rev-parse --show-toplevel)
    ```
    [!NOTE]
    If your bash execution environment is stateless (does not persist variables between tool calls), either combine these steps into a single bash execution block, or explicitly replace $REPO_DIR with the resolved absolute path in your subsequent commands.
2. Review the changes you made.
    ```bash
    git -C "$REPO_DIR" status
    git -C "$REPO_DIR" --no-pager diff --staged  # if already staged
    git -C "$REPO_DIR" --no-pager diff           # if not staged
    ```
3. Stage the files you made changes to if they are not listed in `.gitignore`.  **NEVER commit**: `.env`, `.venv`, `credentials.json`, secrets, or large binary files without explicit approval.
    ```bash
    git -C "$REPO_DIR" add <specific-files>  # preferred
    # or
    git -C "$REPO_DIR" add -A  # all changes
    ```
4. Create the commit using `type(scope): subject` format.  See below for `type` options, `scope` rules, and subject line rules.
    **Simple change (without body)**:
    ```bash
    git -C "$REPO_DIR" commit -m "type(scope): subject"
    ```
    
    **Complex change (with body)**:
    ```bash
    cat << 'EOF' > $REPO_DIR/.git/agent_commit_msg.txt
    type(scope): subject
    
    Body
    References
    EOF
    git -C "$REPO_DIR" commit -F $REPO_DIR/.git/agent_commit_msg.txt
    rm $REPO_DIR/.git/agent_commit_msg.txt
    ```
5. Verify the commit.
    ```bash
    git -C "$REPO_DIR" --no-pager log -1 --format="%h %s"
    git -C "$REPO_DIR" --no-pager show --stat HEAD
    ```

## Default Commit Format

```
type(scope): subject

Body (optional, for complex changes)
References: Task X.Y, Req N (optional)
```

### Commit Types

| Type       | Purpose                                   |
| :--------: | :---------------------------------------- |
| `feat`     | New feature or functionality              |
| `fix`      | Bug fix or issue resolution               |
| `refactor` | Code refactoring without behavior change  |
| `perf`     | Performance improvements                  |
| `test`     | Test additions or modifications           |
| `ci`       | CI/CD configuration changes               |
| `docs`     | Documentation updates                     |
| `chore`    | Maintenance, dependencies, tooling        |
| `style`    | Code formatting, linting (non-functional) |
| `security` | Security vulnerability fixes or hardening |

### Scope (Required, kebab-case)

Examples: `validation`, `auth`, `cookie-service`, `template`, `config`, `tests`, `api`

### Subject Line Rules

- Maximum of 72 characters including type and scope
- Use present tense imperative.  For example: add, implement, fix, improve, enhance, refactor, remove, prevent
- DO NOT use a period at the end
- Specific and descriptive - state WHAT, not WHY

### Body Format (Recommended for Complex Changes)

- Explain HOW and WHY the change was made
- Use bullet points for multiple items
- Wrap at 72 characters
- Include references to tasks, requirements, or issues if applicable

## Git Trailers

| Trailer                        | Purpose                         |
| :----------------------------: | :------------------------------ |
| `Fixes #N`                     | Links and closes issue on merge |
| `Closes #N`                    | Same as Fixes                   |
| `Co-authored-by: Name <email>` | Credit co-contributors          |

Place trailers at end of the body after a blank line. See `references/commit-examples.md` for examples.

## Breaking Changes

For incompatible API/behavior changes, use `!` after scope.  Include a `BREAKING CHANGE:` section if including a body.  These should triggers major version bump in semantic-release.  Example:

```
feat(api)!: change response format to JSON:API

BREAKING CHANGE: Response envelope changed from `{ data }` to `{ data: { type, id, attributes } }`.
```

## Important Rules

- **ALWAYS** check project conventions (AGENTS.md) before committing
- **ALWAYS** use project format if it differs from default
- **ALWAYS** include scope in parentheses
- **ALWAYS** use present tense imperative verb in the subject line
- **NEVER** end the subject line with a period
- **NEVER** commit secrets or credentials
- **NEVER** use generic messages (eg. "update code", "fix bug", "changes")
- **NEVER** exceed 72 characters in subject line inclusive of type and scope
- Group related changes into a single focused commit

## Examples

**Good**:
```
feat(validation): add URLValidator with domain whitelist
fix(auth): use hmac.compare_digest for secure key comparison
refactor(template): consolidate filename sanitization logic
test(security): add 102 path traversal prevention tests
```

**Bad**:
```
update validation code           # no type, no scope, vague
feat: add stuff                  # missing scope, too vague
fix(auth): fix bug               # circular, not specific
chore: make changes              # missing scope, vague
feat(security): improve things.  # has period, vague
```

## References

- `references/commit-examples.md` - Extended examples by type
