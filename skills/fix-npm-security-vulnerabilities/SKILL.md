---
name: fix-npm-security-vulnerabilities
description: "Audits and fixes npm security vulnerabilities in repositories containing a package.json. Handles the full workflow from audit report through fix, changelog review, and PR preparation with Jira ticket linking. Use when the user asks to check, audit, or fix npm/Node.js \"security vulnerabilities\" or \"vulnerabilities\" in a repository."
---

# Fix npm Security Vulnerabilities - step by step instruction

## Step 1: Preparation

- Check if `nvm` is available
- **if there is a** `.nvmrc` file in the current project and check with `nvm current` if the correct version is in use, if not change to the correct Node.JS version using `nvm use`
- **if there is NO** `.nvmrc`, ensure **Node.js 24** is active. If not, switch via `nvm use 24`.
- Whenever you formulate any git commit message, use the skill `git-commit-messages`
- Whenever you create or manage branches, use the skill `git-branch-workflow`

## Step 2: Audit & Report

1. Run `npm audit --json` in the target repository.
2. Parse the JSON output; the `fixAvailable` property indicates whether an automatic fix exists for each vulnerability.
3. **Fetch open PRs and identify already-in-progress packages:**
   - Use the `list-pull-requests` skill to fetch all open PRs for this repository.
   - For each open PR, check either signal to determine which package it covers:
     - **Branch name** matches the pattern `chore/*-<package>-npm-fix` → extract `<package>`.
     - **PR title** matches the pattern `chore(...): bump <package> from ...` → extract `<package>`.
   - Build a set of **already-in-progress packages** from those matches.
4. Present a **summary table** to the user, including a **"PR open?"** column:

| Package | Severity | Fix available? | PR open? |
|---------|----------|----------------|----------|
| lodash  | high     | Yes            | No       |
| semver  | moderate | Yes            | Yes      |

5. **Select the target package:**
   - If the user's invocation context names a specific package:
     - If that package already has an open PR (`PR open?` is `Yes`), warn the user and ask for confirmation before proceeding.
     - Otherwise → use that package.
   - Otherwise → automatically select the highest-severity package where `fixAvailable` is not `false` (i.e. not requiring `--force`) **and `PR open?` is `No`**.
   - If **all** fixable packages already have open PRs, report this and stop:
     > All fixable vulnerabilities already have open fix PRs. No action needed.
   - Announce the selection: _"Proceeding to fix `<package>` (<severity>). Remaining vulnerabilities will be listed at the end."_
6. Wait for the user's go-ahead before proceeding to fixes. The user's approval implicitly confirms the selected package.

## Step 3: Fix (only after user approval)

### Step 3.1 — Create feature branch

Use `chore/<ticket>-<package>-npm-fix` (if a Jira ticket is provided) or `chore/<timestamp>-<package>-npm-fix` (e.g. `chore/20260217-lodash-npm-fix`). Include the package name to keep branches distinct per vulnerability. Delegate to the `git-branch-workflow` skill.

### Step 3.2 — Determine the fix version

Run:
```
npm audit fix --dry-run --json
```

From the output, extract the exact `name@version` to install for the selected package:
- If `fixAvailable` is an object `{ name, version }`, use that (handles transitive cases where a parent must be upgraded).
- Record the current version from `package-lock.json` for comparison (old version → new version).

### Step 3.3 — Apply the fix

Run: `npm install <name>@<version>`

Then run `npm install` if `package.json` was modified (to sync the lockfile).

### Step 3.4 — Verify

- `npx tsc --noEmit` (TypeScript projects only)
- `npm run build` (if build script exists)
- `npm test`
- `npm run lint` (if lint script exists)
- Re-run `npm audit --json` and confirm the target vulnerability is resolved.

If any check fails, report the failure details to the user and **pause for guidance** before continuing.

### Step 3.5 — Research breaking changes

- Get repo URL via `npm view <pkg> repository.url` or `npm view <pkg> homepage`.
- Fetch GitHub release notes for the version range using the URL pattern `https://github.com/<owner>/<repo>/releases/tag/v<version>`. If the page fails to load, fall back to fetching `CHANGELOG.md` directly via `https://raw.githubusercontent.com/<owner>/<repo>/main/CHANGELOG.md`.
- Summarise any **breaking changes, behaviour changes, or API removals** found between the old and new versions.

### Step 3.6 — Report and wait for review

Present a single-package status block:
```
lodash 4.17.20 → 4.17.21
  Verification: build OK, tests OK, lint OK
  Breaking changes: none
```
Include any failures or breaking changes found. **Wait for user review before moving to Step 4.**

## Step 4: PR Preparation

1. **Git commit** the changes using the `git-commit-messages` skill with the message format:
   ```
   chore(<ticket-or-package>): bump <package> from <old> to <new>

   Fixes <severity> severity vulnerability (npm advisory).
   Breaking changes: <none | one-line summary>
   ```
   Use the Jira ticket as scope if provided, otherwise use the package name.

2. **Draft a PR description** scoped to this single package, containing:
   - Referenced Jira tickets (if provided).
   - The package fix: old version → new version, severity, breaking-change notes.
   - Any manual follow-up items (e.g. packages that could not be auto-fixed).
3. Present the draft to the user for review before creating the PR.

## Step 5: PR Creation + Remaining Vulnerabilities Report

1. Git push the changes.
2. Use the skill `create-pull-request` to create a draft PR, handover to the skill the working directory (the current checkout path) and that it should reuse the previously drafted PR message.
3. After the PR is created, re-run `npm audit --json` (or reuse earlier output) to list **remaining unfixed vulnerabilities**:

   ```
   Remaining vulnerabilities (re-invoke the skill to fix the next one):
   | Package | Severity | Fix available? |
   |---------|----------|----------------|
   | semver  | moderate | Yes            |
   | ms      | low      | No (--force)   |
   ```

4. **Stop.** Re-invoke the skill to fix the next vulnerability.

# When to Apply

- User asks to "audit", "check", or "fix" npm vulnerabilities.
- User mentions security vulnerabilities in the context of a Node.js project.
- User asks to prepare a security-fix PR for a npm project.

Do **not** use this skill for non-npm ecosystems (pip, maven, etc.) or for general dependency upgrades unrelated to security.

The steps above are ordered intentionally. You MUST follow the order Step 1 -> Step 2 -> Step 3 -> Step 4 -> Step 5.
