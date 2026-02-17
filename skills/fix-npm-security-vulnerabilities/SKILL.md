---
name: fix-npm-security-vulnerabilities
description: "Audits and fixes npm security vulnerabilities in repositories containing a package.json. Handles the full workflow from audit report through fix, changelog review, and PR preparation with Jira ticket linking. Use when the user asks to check, audit, or fix npm/Node.js \"security vulnerabilities\" or \"vulnerabilities\" in a repository."
---

# Fix npm Security Vulnerabilities

Use this skill when the user asks to audit, check, or fix npm security vulnerabilities (or just "vulnerabilities") in a project containing a package.json. The skill covers the full lifecycle: audit, triage, fix, breaking-change review, and PR preparation.

## Agent Workflow

The different phases below are ordered intentionally. You MUST follow the order of these phases when solving the task.

### Phase 0 - Setup

- Check if `nvm` is available
- **if there is a** `.nvmrc` file in the current project and check with `nvm current` if the correct version is in use, if not change to the correct Node.JS version using `nvm use`
- **if there is NO** `.nvmrc`, ensure **Node.js 24** is active. If not, switch via `nvm use 24`.
- Whenever you formulate any git commit message, use the skill `git-commit-messages`
- Whenever you create or manage branches, use the skill `git-branch-workflow`

### Phase 1 — Audit & Report

1. Run `npm audit --json` in the target repository.
2. Parse the JSON output; the `fixAvailable` property indicates whether an automatic fix exists for each vulnerability.
3. Present a **summary table** to the user:

| Package | Severity | Fix available? |
|---------|----------|----------------|
| example | high     | Yes            |

4. Wait for the user's go-ahead before proceeding to fixes.

### Phase 2 — Fix (only after user approval)

1. **Create a feature branch.** If a Jira ticket is provided, use `feat/<ticket>-npm-fix`. Otherwise use `feat/<timestamp>-npm-fix` (e.g. `feat/20260217-npm-fix`).
2. **Record current package versions** — capture the relevant sections of `package-lock.json` or `npm ls` output so changes can be compared later.
3. Run `npm audit fix` to apply automatic fixes.
4. **Verify the project still builds and passes tests** after applying fixes:
   - Run `npm install` (if `audit fix` modified `package.json`) to ensure the lockfile is consistent and all dependencies resolve.
   - **TypeScript projects:** Run the TypeScript compiler (`npx tsc --noEmit`) and confirm there are no type errors introduced by updated packages.
   - **Build step:** Run `npm run build` (or the equivalent build script defined in `package.json`). If the build fails, investigate whether an updated dependency introduced a breaking change.
   - **Tests:** Run `npm test`. All previously-passing tests must still pass. If tests fail, determine whether the failure is caused by an updated dependency and report it to the user before continuing.
   - **Linting:** If a lint script exists (`npm run lint`), run it to catch any new issues (e.g. updated type definitions that trigger lint rules).
   - **Re-run `npm audit --json`** and confirm the targeted vulnerabilities are resolved. Report any remaining or newly introduced advisories back to the user.
   - If any of the above checks fail, include the findings in the report created in the next step.
5. **Compare new vs. old package versions.** For each updated package:
   - Identify the version change (e.g. `1.2.3 → 1.3.0`).
   - Search the package's changelog or release notes (GitHub releases, npm page, or CHANGELOG.md) for **potential breaking changes** between the old and new versions.
6. Summarize findings for the user. If there are were issues detected earlier, make suggestions on how to follow up. Only continue with Phase 3 after review by the user.

### Phase 3 — PR Preparation

1. Git commit the changes
2. **Draft a PR description** containing:
   - Referenced Jira tickets (if found).
   - A summary table of applied fixes (package, old version → new version, severity, breaking-change notes).
   - Any manual follow-up items (e.g. packages that could not be auto-fixed, or breaking changes that need testing).
3. Present the draft to the user for review before creating the PR.

### Phase 4 — PR Preparation

1. Git push the changes and retrieve the URL to create a PR if provided from the remote server
2. Show a summary and offer the copy the PR message into the users clipboard, so they can create the PR manually.

## When to Apply

- User asks to "audit", "check", or "fix" npm vulnerabilities.
- User mentions security vulnerabilities in the context of a Node.js project.
- User asks to prepare a security-fix PR for a npm project.

Do **not** use this skill for non-npm ecosystems (pip, maven, etc.) or for general dependency upgrades unrelated to security.
