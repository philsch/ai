---
name: code-reviewer
description: Reviews changed files using the code-reviewer agent and presents findings as a human-readable report grouped by severity. Apply when the user asks for a code review of the current branch or a set of files.
---

# Code Reviewer Skill

This skill runs a code review on changed files and presents the findings in a readable
format for direct human consumption.

## Inputs

The user must provide one of:

- A list of file paths to review, or
- No input — the skill will derive changed files from `git diff` against the base branch

## Step 1 — Understand intent

Run:

```bash
git log -1 --format="%s%n%b"
```

to read the commit message. If a PR description is available (e.g. from a Bitbucket or
GitHub context passed by the caller), use that instead. This frames what "correctness"
means for this specific change.

## Step 2 — Collect changed files

If the user provided explicit file paths, use those. Otherwise, run:

```bash
git diff --name-only origin/HEAD...HEAD
```

to collect added and modified files. Read test files first, then implementation files.

Also read any `CLAUDE.md` files present in the repository root or in the directories of
the changed files — these contain project-specific rules that must be respected during
the review.

## Step 3 — Run code review

Invoke the `code-reviewer` agent on the files read in Step 2, passing the file contents,
change intent, and any `CLAUDE.md` rules as context. Tell the agent to read tests first.
The agent handles the review criteria and prioritization.

## Step 4 — Present findings

Display findings grouped by severity. Omit any section that has no entries.

```
## Must Fix
1. [<file>:<line>] <description>
2. ...

## Suggestions
1. [<file>:<line>] <description>
2. ...

## Nits
1. [<file>:<line>] <description>
2. ...
```

If there are no findings at all, output:

> No issues found.
