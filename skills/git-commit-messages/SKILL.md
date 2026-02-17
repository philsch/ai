---
name: git-commit-messages
description: "Commit message structure, wording, and conventions. Apply whenever a commit message is written — whether the user asks for one or the agent autonomously prepares a commit. Also use when reviewing, improving, squashing, or amending commits, or when explaining commit standards."
---

# Git Commit Messages

## Format

```
<type>(<ticket-or-scope>): <short description>

[optional body]
```

### Subject line (required)

- **Imperative mood**: "Add", "Fix", "Remove" — not past tense
- **Lowercase** after the prefix
- **~50 characters** — move detail to the body
- **No trailing period**

### Prefix

- **Ticket exists**: use the ticket key — `feat(PROJ-42): add bulk export`
- **No ticket**: use a short scope — `feat(auth): add MFA option`
- **No clear scope**: omit parentheses — `docs: update README`
- **Branch alignment**: match the ticket from branch name (e.g. `feat/PROJ-42-bulk-export` → `feat(PROJ-42): ...`)

### Types

| Type | Use for |
|------|---------|
| `feat` | New feature or user-facing capability |
| `fix` | Bug fix (behavior or correctness) |
| `docs` | Documentation only |
| `chore` | Build, tooling, dependencies, config |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `test` | Adding or changing tests only |
| `style` | Formatting, whitespace — no logic change |

**Special rule:** Dependency/package security updates (e.g. `npm audit fix`, bumping a vulnerable package) are always `chore`, not `fix` — reserve `fix` for bugs in *your* code.

### Body (optional)

Add a body when the *why* isn't obvious from the diff:

- Breaking or subtle behaviour change
- Context, design-doc, or ADR reference
- Multiple small edits forming one logical change

Wrap at ~72 characters. Blank line between subject and body.

## Examples

```
feat(PROJ-42): add CSV export for user records
fix(PROJ-108): return 400 when tenant id is missing
docs(PROJ-7): add runbook for deployment rollback
chore(deps): bump axios to 1.6.2
refactor(auth): extract token validation into shared helper
```

With body:

```
fix(PROJ-108): return 400 when tenant id is missing

Previous behaviour silently fell through to a 500. The upstream
service now validates tenant id on its side, so we surface the
error early.
```

## Avoid

- **Vague subjects**: "Fix bug", "Update code", "Changes" — be specific
- **Empty summary after prefix**: always include a human-readable description after the colon
- **Multiple concerns**: one logical change per commit
- **Implementation-only detail**: prefer product-facing summary; put filenames in the body

## Agent workflow

**Drafting a commit message:**

1. Infer **type** and **ticket/scope** from the diff (and branch name if available).
2. Write the subject — imperative, lowercase, ~50 chars, no period.
3. Add a body only if context or rationale is needed.
4. Output the full message ready to paste.

**Answering "what's our convention?":**

Summarise the format and give 2–3 examples (with and without ticket).

**Reviewing PR history:**

Suggest clearer subjects following the rules above — don't rewrite code or history unless asked.

**Scope:** commit message content only — not branch naming.
