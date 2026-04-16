---
name: code-reviewer-json
description: Reviews changed files using the code-reviewer agent and outputs findings as a structured JSON array. Apply when a calling skill needs code review findings in a machine-readable format with file, line, severity, and description fields (e.g. review-bitbucket-pr).
disable-model-invocation: true
---

# Code Reviewer JSON Skill

This skill acts as a bridge between calling skills (such as `review-bitbucket-pr`) and the
`code-reviewer` agent. It reads the changed files, runs the review, and produces a structured
JSON list of findings for programmatic consumption.

## Inputs

The calling skill must provide:

- `<changed-files>` — list of file paths relative to the working directory
- `<review-dir>` — absolute path to the directory containing the checked-out source

## Step 1 — Read changed files

For each file in `<changed-files>`, read its full contents from `<review-dir>`. Also read
any `CLAUDE.md` files present in the repository root or in the directories of the changed
files — these contain project-specific rules that must be respected during review.

## Step 2 — Run code review

Invoke the `code-reviewer` agent on the files read in Step 1, passing the file contents and
any `CLAUDE.md` rules as context. The agent handles the review criteria and prioritization.

Nits from the agent's output must be omitted from the structured findings.

## Step 3 — Output structured findings

Return findings as a JSON array and nothing else. Each entry must have exactly these fields:

```json
[
  {
    "file": "<file-path-relative-to-review-dir>",
    "line": <line-number>,
    "severity": "must-fix",
    "description": "<concise description with suggested fix>"
  },
  {
    "file": "<file-path-relative-to-review-dir>",
    "line": <line-number>,
    "severity": "suggestion",
    "description": "<concise description with suggested fix>"
  }
]
```

Rules for the output:
- `severity` must be exactly `"must-fix"` or `"suggestion"` — no other values
- `line` must be the line number in the new (source branch) version of the file
- `description` must be self-contained — it will be posted verbatim as a PR comment
- If there are no findings, return an empty array: `[]`
- Do not include nits in the output
- Do not include any text outside the JSON array
