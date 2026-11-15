---
name: review-bitbucket-pr
description: "Reviews a Bitbucket pull request: fetches PR metadata and existing comments, checks out the source branch, runs a code review, presents must-fix and suggestion findings, then posts accepted findings as inline comments on the PR. Apply when the user asks to review a Bitbucket PR, check a pull request, or audit code changes on Bitbucket."
---

# Review Bitbucket Pull Request

## Preconditions

The skill requires a Bitbucket Cloud repository. It accepts one of two inputs:

- A full Bitbucket PR URL: `https://bitbucket.org/<workspace>/<repo-slug>/pull-requests/<id>`
- A workspace/repo slug plus the source branch name: `<workspace>/<repo-slug>` + `<branch>`

## Step 1 — Parse input

Determine `<workspace>`, `<repo-slug>`, and `<pr-id>` from the input:

**From a full URL** — extract the three path segments after `bitbucket.org/`:

```
https://bitbucket.org/<workspace>/<repo-slug>/pull-requests/<pr-id>
```

**From workspace/repo + branch** — proceed to Step 2 to load credentials, then resolve
the PR id via the API:

```
curl -L -X GET \
  -u '<BITBUCKET_EMAIL>:<BITBUCKET_API_TOKEN>' \
  -H 'Accept: application/json' \
  'https://api.bitbucket.org/2.0/repositories/<workspace>/<repo-slug>/pullrequests?state=OPEN&q=source.branch.name%3D%22<branch>%22'
```

Take the `id` of the first entry in `values`. If no match is found, stop and tell the
user that no open PR was found for that branch.

## Step 2 — Load credentials

Read `BITBUCKET_EMAIL` and `BITBUCKET_API_TOKEN` from the file:

```
<path-to-this-skill>/.env
```

Where `<path-to-this-skill>` is the directory containing this `SKILL.md` file (i.e.
`skills/review-bitbucket-pr/.env` relative to the root of the skill library).

The API token must have the `pullrequests:read` and `pullrequests:write` scopes.

If the `.env` file is missing or either variable is unset, stop and tell the user:

> **Bitbucket credentials not found.**
> Copy `skills/review-bitbucket-pr/.env.example` to `skills/review-bitbucket-pr/.env`
> and fill in your Atlassian account email (`BITBUCKET_EMAIL`) and an API token with the
> `pullrequests:read` and `pullrequests:write` scopes (`BITBUCKET_API_TOKEN`).
> Then re-run this skill.

## Step 3 — Fetch PR metadata

```
curl -L -X GET \
  -u '<BITBUCKET_EMAIL>:<BITBUCKET_API_TOKEN>' \
  -H 'Accept: application/json' \
  'https://api.bitbucket.org/2.0/repositories/<workspace>/<repo-slug>/pullrequests/<pr-id>'
```

Extract and record:

- `title`
- `description`
- `state` — abort with a clear message if not `OPEN`
- `source.branch.name` — source branch to check out
- `destination.branch.name` — base branch
- `links.html.href` — PR URL for output at the end

Construct the SSH clone URL directly from the already-known `<workspace>` and `<repo-slug>`:

```
<clone-url> = git@bitbucket.org:<workspace>/<repo-slug>.git
```

Do not attempt to parse clone links from the API response — Bitbucket does not reliably
include them in the nested `source.repository` object on the pull request endpoint.

Display a one-line summary to the user:

```
Reviewing PR #<pr-id>: <title>
  <source-branch> → <destination-branch>
  <pr-url>
```

## Step 4 — Fetch existing PR comments

```
curl -L -X GET \
  -u '<BITBUCKET_EMAIL>:<BITBUCKET_API_TOKEN>' \
  -H 'Accept: application/json' \
  'https://api.bitbucket.org/2.0/repositories/<workspace>/<repo-slug>/pullrequests/<pr-id>/comments?pagelen=100'
```

If the response contains a `next` link, repeat the request with that URL until all pages
are retrieved.

From each comment that contains an `inline` field, record a tuple of:
`{ path: inline.path, line: inline.to, snippet: content.raw (first 120 chars) }`

Store this as `<existing-inline-comments>` for use in Step 9.

## Step 5 — Fetch changed files and diff

> **Note:** Bitbucket returns HTTP 302 redirects for both the `/diffstat` and `/diff`
> endpoints. Always include the `-L` flag so curl follows the redirect automatically.

### Diffstat (file list)

```
curl -L -X GET \
  -u '<BITBUCKET_EMAIL>:<BITBUCKET_API_TOKEN>' \
  -H 'Accept: application/json' \
  'https://api.bitbucket.org/2.0/repositories/<workspace>/<repo-slug>/pullrequests/<pr-id>/diffstat'
```

Collect the `new.path` of every entry with `status` of `added` or `modified`.
Store as `<changed-files>`.

### Full diff (line ranges)

Fetch the complete unified diff without any output truncation:

```
curl -L -X GET \
  -u '<BITBUCKET_EMAIL>:<BITBUCKET_API_TOKEN>' \
  'https://api.bitbucket.org/2.0/repositories/<workspace>/<repo-slug>/pullrequests/<pr-id>/diff'
```

Do not pipe through `head` or any other truncating command. The full response must be
read to build a complete line map.

Parse the unified diff text to build `<diff-lines>`, a map of
`{ "<file-path>": Set<line-number> }`, using this algorithm:

1. When a line matches `+++ b/<path>`, set `<current-file>` to `<path>` and reset
   `<current-new-line>` to 0.
2. When a line matches a hunk header `@@ -a,b +c,d @@`, set `<current-new-line>` to
   `c` (the starting line number in the new file). Do not increment yet.
3. For each subsequent line in the hunk:
   - If it starts with `+` (but is not `+++`): record `<current-new-line>` into
     `<diff-lines>[<current-file>]`, then increment `<current-new-line>`.
   - If it starts with ` ` (context line): increment `<current-new-line>` only.
   - If it starts with `-` (removed line): do not increment `<current-new-line>`.
4. Repeat for every hunk and every file in the diff.

Only `+` lines count as changed. Context lines inside a hunk are unchanged code.

This map defines the review scope: Step 7 passes it to the agent, Step 8 drops every
finding outside it, and Step 11 uses it to anchor inline comments.

## Step 6 — Check out source branch into a temp directory

```bash
REVIEW_DIR=$(mktemp -d)
git clone --depth=1 --branch <source-branch> <clone-url> "$REVIEW_DIR"
```

Record `$REVIEW_DIR` for use in Step 7.

If the clone fails (e.g. SSH key not available), stop, show the error, and ask the user
how to proceed.

## Step 7 — Run code review

For each file in `<changed-files>`, read its full contents from `<review-dir>`. Also read
any `CLAUDE.md` files present in the repository root or in the directories of the changed
files.

Convert `<diff-lines>` into changed line ranges per file (e.g.
`errors.ts: 2, 10-14`). Store as `<changed-ranges>`.

Invoke the `code-reviewer` agent on those file contents and any `CLAUDE.md` rules as
context. Also pass `<review-dir>` and `<changed-ranges>`, and instruct the agent:

- It may read any file in `<review-dir>`, in particular sibling modules of the same kind
  and imported modules, to compare conventions and placement.
- Files and lines outside `<changed-ranges>` are **reference only**. They may be cited as
  evidence in a description, but never be the target of a finding.
- Every finding must anchor on a line in `<changed-ranges>`. A placement finding anchors
  on the misplaced definition in the changed file and names the target file in the
  description.

Instruct the agent to return findings as a JSON array and nothing else, omitting nits.
These are suggestions, not nits, and must be kept:

- Comment-quality findings (verbose, narrative, or drift-prone inline comments)
- Placement and cohesion findings on added or changed code (a definition in a file whose
  responsibility does not match it, or organized differently from sibling modules)

Each entry must have exactly these fields:

```json
[
  {
    "file": "<file-path-relative-to-review-dir>",
    "line": <last line of the affected range>,
    "line_from": <first line of the affected range — omit for single-line findings>,
    "scope": "file — omit unless the finding concerns the changed file as a whole",
    "severity": "must-fix" | "suggestion",
    "description": "<concise description with suggested fix>",
    "suggestion": "<replacement code — omit when no concrete replacement can be given>"
  }
]
```

- `severity` must be exactly `"must-fix"` or `"suggestion"`
- `line` is the last (or only) line of the affected range in the new (source branch) version of the file
- `line_from` must be set when the finding spans more than one line; omit it for single-line findings
- `scope` is set to `"file"` only for findings about a changed file as a whole that no single line can carry (e.g. a new client without a test file). Omit it otherwise
- `suggestion` must only be populated when the agent can provide a correct, self-contained replacement snippet; leave it absent otherwise
- `description` must be self-contained — it will be posted verbatim as a PR comment
- If there are no findings, return an empty array: `[]`

## Step 8 — Filter to the diff

Do not rely on the agent instruction alone. Check every finding from Step 7 against
`<diff-lines>`:

| Finding | Handling |
|---|---|
| `file` not in `<diff-lines>` | Drop |
| `scope` is `"file"` and `file` is in `<diff-lines>` | Keep |
| `line`, or for multi-line findings any line from `line_from` to `line`, is in `<diff-lines>[file]` | Keep |
| Otherwise (changed file, but the range touches no added line) | Drop |

Store the dropped findings as `<out-of-diff-findings>`. They are shown in Step 10 but
never posted.

For a kept multi-line finding whose `line` is not itself an added line, move `line` to the
last added line within the range, so the inline anchor in Step 11 is valid.

## Step 9 — Filter against existing comments

For each finding kept in Step 8, check `<existing-inline-comments>`:

- If any existing comment targets the same `path` and its `line` is within ±5 lines of
  the finding's line, consider the finding already covered.
- If any existing comment's `snippet` contains the same function name or variable name as
  the finding's description, also consider it covered.

Remove covered findings from the list. Count them separately as `<already-covered-count>`.

## Step 10 — Present findings and wait for user confirmation

Display the filtered findings grouped by severity:

```
## Must Fix
1. [<file>:<line>] <description>
2. ...

## Suggestions
1. [<file>:<line>] <description>
2. ...

## Outside the diff, not posted
- [<file>:<line>] <description>

## Already covered by existing comments
<already-covered-count> finding(s) skipped.
```

Omit the "Outside the diff" section when `<out-of-diff-findings>` is empty. Its entries are
unnumbered and cannot be selected for posting.

If no findings remain after filtering, tell the user:

> No new findings. All issues appear to be covered by existing PR comments.

Then stop.

Otherwise, ask the user which findings to post:

> Reply with one of:
> - `post all` — post every finding above
> - `post must-fix` — post only Must Fix items
> - A comma-separated list of numbers (e.g. `1, 3`) to post specific items
> - `cancel` — do not post anything

Wait for the user's reply before proceeding to Step 11.

## Step 11 — Post accepted findings as inline comments

For each accepted finding, determine the comment type:

- If `scope` is `"file"` — post as a **general** PR-level comment, prefixing the
  description with `[<file>]`.
- Otherwise — post as an **inline** comment. Step 8 guarantees that `finding.line` is in
  `<diff-lines>[finding.file]`.

Never post a finding that did not pass Step 8.

### Building the comment body

- **Without `suggestion`:** use plain text only: `🤖 <description>`
- **With `suggestion`:** append the replacement snippet in a fenced suggestion block.
  The value of `content.raw` must be:

      🤖 <description>

      ```suggestion
      <suggestion>
      ```

### Building the `inline` object

- **Single-line** (`line_from` absent or equal to `line`): include only `"to": <line>`.
- **Multi-line** (`line_from` present and less than `line`): include both `"from": <line_from>` and `"to": <line>`.

### Inline comment

```
curl -X POST \
  -u '<BITBUCKET_EMAIL>:<BITBUCKET_API_TOKEN>' \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
    "content": { "raw": "<comment-body>" },
    "inline": {
      "from": <line_from — only for multi-line findings>,
      "to": <line>,
      "path": "<file>"
    }
  }' \
  'https://api.bitbucket.org/2.0/repositories/<workspace>/<repo-slug>/pullrequests/<pr-id>/comments'
```

### General PR-level comment

```
curl -X POST \
  -u '<BITBUCKET_EMAIL>:<BITBUCKET_API_TOKEN>' \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
    "content": { "raw": "🤖 [<file>] <description>" }
  }' \
  'https://api.bitbucket.org/2.0/repositories/<workspace>/<repo-slug>/pullrequests/<pr-id>/comments'
```

On failure for any individual comment, report the error and continue with the remaining
findings. Do not abort the whole batch.

### Final output

```
Posted <N> comment(s) on PR #<pr-id>.
<pr-url>
```

# When to Apply

- The user says "review PR", "review this Bitbucket PR", "check this pull request", or
  provides a Bitbucket PR URL and asks for a review.
- A calling skill needs automated code review and comment posting on a Bitbucket PR.
