---
name: git-branch-workflow
description: "Best practices for creating, naming, and managing Git branches. Apply whenever the agent creates a branch, switches branches, or prepares work on a new task."
---

# Git Branch Workflow

## 1. Always branch from an up-to-date base

- Before creating any branch, fetch the latest remote state (`git fetch origin`).
- Ensure the base branch (usually `main`) is up to date: `git checkout main && git pull origin main`.
- Never branch from a stale local copy of `main`.

## 2. Branch naming conventions

Use the pattern: `<type>/<ticket-or-scope>-<short-kebab-description>`

### With a Jira ticket

Use the ticket key as the scope:

```
feat/PROJ-42-bulk-export
fix/PROJ-108-missing-tenant-id
chore/PROJ-55-bump-axios
```

### Without a ticket

Use a short descriptive slug:

```
feat/add-csv-export
fix/check-on-login
chore/update-eslint-config
```

### Type prefixes

Use the same prefixes as the `git-commit-messages` skill:

| Prefix | Use for |
|-----------|---------|
| `feat/` | New feature or user-facing capability |
| `fix/` | Bug fix (behavior or correctness) |
| `docs/` | Documentation only |
| `chore/` | Build, tooling, dependencies, config |
| `refactor/` | Code change that neither fixes a bug nor adds a feature |
| `test/` | Adding or changing tests only |

### Naming rules

- **kebab-case**, lowercase only, no special characters beyond hyphens.
- Keep branch names short but descriptive — aim for **under 50 characters**.
- No trailing hyphens or double hyphens (`--`).
- No generic names like `my-branch`, `temp`, `wip`, or `test`.

## 3. One concern per branch

- Each branch should address a **single logical change** (one feature, one bug fix, one refactor).
- Avoid mixing unrelated changes — they complicate review and rollback.

## 4. Start with a clean working tree

- Before creating or switching branches, ensure there are no uncommitted changes (`git status` must be clean).
- If there are uncommitted changes, **ask the user** what to do: either commit them, stash them (`git stash`), or discard them before branching.
- Never create a new branch with a dirty working tree — uncommitted changes carry over to the new branch and cause confusion.

## Agent workflow

When the agent needs to create a branch:

1. Run `git fetch origin` and check out the latest base branch.
2. Verify the working tree is clean (`git status`). Ask for advice if there are uncommitted changes.
3. Derive the branch name from the ticket (if provided) + type + short description, following the naming conventions above.
4. Create and switch to the new branch (`git checkout -b <branch-name>`).
5. Use the `git-commit-messages` skill when committing on the branch.
