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

### Step 3.2 — Probe stale overrides

An existing override for the target package can be stale: a *direct* dependency may have since shipped a release whose own tree no longer pulls in the vulnerable transitive version, making our pinned override unnecessary. Probe for this before deciding on a fix.

1. **Detect overrides for the target package** in root `package.json`:
   - Top-level: `overrides.<package>`
   - Path-scoped: `overrides.<parent>.<package>`, including nested forms and `parent@version` keys
2. **If no override exists for the target package, skip this step** and proceed to Step 3.3.
3. **Remove every override entry that names the target package** (top-level + all path-scoped occurrences). If removing a `<package>` entry leaves a parent block empty, also remove that empty parent block. Do not touch overrides for *other* packages. Record the old override version(s) for the Step 3.9 report.
4. Run `npm install` to re-resolve the lockfile against the cleaned `package.json`.
5. Re-run `npm audit --json` and inspect the entry for the target package:
   - **Vulnerability gone** (no advisory for the package, or severity below the original threshold): the override was stale. Record resolution path = "stale override removed". Skip Steps 3.3–3.6 entirely and proceed to **Step 3.7 (Verify)**.
   - **Vulnerability still present:** keep the override removal staged and continue with the normal flow at Step 3.3 (determine fix version on the clean baseline). The downstream Step 3.5 (parent update) and Step 3.6 (overrides) operate on the post-removal state.
6. **Report** the probe outcome to the user before moving on:
   - "Probe: removed stale override for `<package>` — vulnerability resolved, no further fix needed." or
   - "Probe: removed stale override for `<package>` — vulnerability still present, continuing with fix flow."

### Step 3.3 — Determine the fix version

Run:
```
npm audit fix --dry-run --json
```

From the output, extract the exact `name@version` to install for the selected package:
- If `fixAvailable` is an object `{ name, version }`, use that (handles transitive cases where a parent must be upgraded).
- Record the current version from `package-lock.json` for comparison (old version → new version).

### Step 3.4 — Apply the fix

**Before running `npm install`, check whether `<name>` is already listed as a direct dependency** in root `package.json` under `dependencies`, `devDependencies`, or `optionalDependencies`.

- **If it is a direct dependency:** run `npm install <name>@<version>` to update it in place. Then run `npm install` if `package.json` was modified (to sync the lockfile). Skip Steps 3.5 and 3.6, proceed to **Step 3.7** (Verify).
- **If it is NOT a direct dependency:** skip `npm install` here. Do not run `npm install <name>@<version>`, as that would add it as a new direct dependency. Instead, proceed to **Step 3.5** to check whether a parent direct dependency can be updated to transitively resolve the vulnerability before falling back to overrides.

### Step 3.5 — Try resolving indirect dependency via parent update

**Run only when the fixed package is NOT a direct dependency** (skipped `npm install` in Step 3.4). If the package was a direct dependency, skip this step and proceed to Step 3.7.

1. **Identify parents:** Run `npm ls <package> --all --json` to map which packages directly depend on `<package>`. Extract each immediate parent from the tree.

2. **Filter to direct parents:** From those parents, keep only those that appear as a key in root `package.json` under `dependencies`, `devDependencies`, or `optionalDependencies`. Discard any transitively-introduced parents. If none remain, skip this step and proceed to Step 3.6.

3. **Find non-breaking updates for each direct parent:** For each direct parent `<parentPkg>`:
   - Run `npm view <parentPkg> versions --json` to list all published versions.
   - Determine the currently installed version from `package-lock.json`.
   - A **non-breaking candidate** is any published version that shares the **same major version** and is **greater than** the currently installed version.
   - For each non-breaking candidate (starting from the highest), inspect whether it would resolve `<package>` to `>= <fixVersion>` (from Step 3.3). Use `npm view <parentPkg>@<candidate> dependencies --json` or simulate with `npm install <parentPkg>@<candidate> --dry-run` to check.
   - Record the **lowest viable candidate** (prefer the smallest bump to reduce risk).

4. **Decision:**
   - **If at least one direct parent has a viable non-breaking update:** Choose the parent with the smallest version bump. Run:
     ```
     npm install <parentPkg>@<resolvedCandidateVersion>
     ```
     **Skip Step 3.6** - do not add an override. Proceed to **Step 3.7** (Verify).
   - **If no direct parent has a viable non-breaking update** (all require a major bump or none exist): Proceed to **Step 3.6** (overrides).

### Step 3.6 — Check/add overrides for indirect dependencies

**Run only when both conditions are true:** (a) the fixed package is not a direct dependency (not in root `package.json` under `dependencies`, `devDependencies`, or `optionalDependencies`), AND (b) Step 3.5 found no viable non-breaking parent update. If the package was direct, or if Step 3.5 successfully resolved the vulnerability via a parent update, skip this step entirely.

1. **Confirm indirect:** Check that the fixed package name is not listed in root `package.json` dependencies/devDependencies/optionalDependencies. If it is, skip this step.
2. **Discover parent paths:** Use `npm ls <package> --all` (or parse `package-lock.json`'s `packages` / lockfile v3 structure) to list which parent packages depend on `<package>` and at what depth. Record for each occurrence: parent chain or top-level parent, and the resolved version after fix (the fix version from Step 3.3 is the target version to lock).
3. **Decide global vs path overrides:**
   - If the package appears under **only one** parent path, or under multiple parents but the **same** fixed version applies to all: add or extend a **single top-level** override: `"<package>": "<fix-version>"`.
   - If the package appears under **multiple parents** and the tree indicates **different versions** are required for different parents: use **path overrides** so each parent gets the correct version, e.g. `"overrides": { "parentA": { "<package>": "<versionA>" }, "parentB": { "<package>": "<versionB>" } }`. Optionally scope by parent version with `parent@version` if needed.
4. **Merge with existing overrides:** If `package.json` already has an `overrides` object, extend it: add the new key(s) without removing or overwriting unrelated overrides. If the project has no existing `overrides`, create the `overrides` object. Preserve existing nesting (e.g. existing path overrides under the same parent); nested keys follow npm's override semantics.
5. **Apply and re-sync:** After editing `package.json`, run `npm install` to apply overrides and update the lockfile.
6. When overrides were added or extended, note it for the status block in Step 3.9 (e.g. path overrides for parentA, parentB, or global override for `<package>@<version>`).

### Step 3.7 — Verify

- `npx tsc --noEmit` (TypeScript projects only)
- `npm run build` (if build script exists)
- `npm test`
- `npm run lint` (if lint script exists)
- Re-run `npm audit --json` and confirm the target vulnerability is resolved.

If any check fails, report the failure details to the user and **pause for guidance** before continuing.

### Step 3.8 — Research breaking changes

- Get repo URL via `npm view <pkg> repository.url` or `npm view <pkg> homepage`.
- Fetch GitHub release notes for the version range using the URL pattern `https://github.com/<owner>/<repo>/releases/tag/v<version>`. If the page fails to load, fall back to fetching `CHANGELOG.md` directly via `https://raw.githubusercontent.com/<owner>/<repo>/main/CHANGELOG.md`.
- Summarise any **breaking changes, behaviour changes, or API removals** found between the old and new versions.

### Step 3.9 — Report and wait for review

Present a single-package status block:
```
lodash 4.17.20 → 4.17.21
  Verification: build OK, tests OK, lint OK
  Resolution: <only if indirect dep — omit for direct deps — one of:>
    - "removed stale override for <package> (direct dependencies now ship a fixed version)" — Step 3.2 path
    - "parent <parentPkg> bumped to <version> (transitively resolves <package>)" — Step 3.5 path
    - "added global override for <package>@<fixVersion>" — Step 3.6 path
    - "added path overrides for <parentA>, <parentB>" — Step 3.6 path, multiple parents
  Breaking changes: none
```
Include any failures or breaking changes found. When the fixed package is direct, omit the Resolution line. When Step 3.2 (probe) resolved the issue on its own, that is the only Resolution entry and the "old → new" line reads `<package> <oldOverrideVersion> → (override removed; resolved version <resolvedVersion>)`. When Step 3.5 resolved the issue via a parent update, note the parent bump. When Step 3.6 ran, note the override details. **Wait for user review before moving to Step 4.**

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
