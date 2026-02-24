---
name: create-pull-request
description: "Creates a draft pull request. Apply when a PR needs to be created after changes are committed and pushed, or when the user asks to create a PR."
---

# Create Pull Request

## Preconditions

When this skill is called, infer from the previous context which repository we are currently working on, and if a pull request message has already been drafted, you can use that; otherwise, generate one from the commit log in Step 2.5.

Changes must already be committed on a feature branch before invoking this skill.

## Step 1 — Detect remote type

Run the following, substituting `<repo-path>` with the identified repository path:

```
git -C <repo-path> remote get-url origin
```

- Remote URL contains `github.com` → proceed with **Step 2 → Step 3a (GitHub)**
- Remote URL contains `bitbucket.org` → proceed with **Step 2 → Step 3b (Bitbucket Cloud)**
- Anything else → skip to **Step 3c (Fallback)**

## Step 2 — Get current branch and push

```
git -C <repo-path> branch --show-current
git -C <repo-path> push -u origin <current-branch>
```

Record `<current-branch>` for use in Steps 2.5, 3a, and 3b.

## Step 2.5 — Review commit log and verify PR message

```
git -C <repo-path> log <base-branch>..<current-branch> --pretty=format:"%s%n%b"
```

- Read all commit subjects and bodies on the feature branch that are not yet on `<base-branch>`.
- Compare them against the supplied PR title and body.
- If any commit introduces context not reflected in the PR description (e.g. a fix, a caveat, a
  breaking change mentioned in a commit body), incorporate it into the PR body.
- Do **not** silently discard or overwrite the caller-supplied content — extend it only.
- If the PR title is empty, synthesise one from the most recent commit subject.
- If the PR body is empty, synthesise it from all commit subjects and bodies on the branch.
- If the title and body already cover all commits, proceed without changes.

The updated title and body are used in Steps 3a / 3b / 3c.

## Step 3a — GitHub (gh CLI)

Requires `gh` to be installed and authenticated (`gh auth status`).

Extract `<owner>/<repo>` from the remote URL. Then run:

```
gh pr create \
  -R <owner>/<repo> \
  --draft \
  --title "<title>" \
  --body "<body>" \
  --base <base-branch> \
  --head <current-branch>
```

Output the PR URL returned by `gh` and stop.

## Step 3b — Bitbucket Cloud (REST API v2)

### Credentials

Read `BITBUCKET_EMAIL` (your Atlassian account email) and `BITBUCKET_API_TOKEN` from the file:

```
<path-to-this-skill>/.env
```

Where `<path-to-this-skill>` is the directory containing this `SKILL.md` file (i.e.
`skills/create-pull-request/.env` relative to the root of the skill library).

The API token must have the `pullrequests:write` scope.

If the `.env` file is missing or either variable is unset, stop and tell the user:

> **Bitbucket credentials not found.**
> Copy `skills/create-pull-request/.env.example` to `skills/create-pull-request/.env` and fill in
> your Atlassian account email (`BITBUCKET_EMAIL`) and an API token with the `pullrequests:write`
> scope (`BITBUCKET_API_TOKEN`).
> Then re-run this skill.

### Extract workspace and repo slug

Parse `<workspace>` and `<repo-slug>` from the remote URL:

- HTTPS format: `https://bitbucket.org/<workspace>/<repo-slug>.git`
- SSH format: `git@bitbucket.org:<workspace>/<repo-slug>.git`

### Fetch default reviewers

Before creating the PR, retrieve the repository's default reviewers:

```
curl -X GET \
  -u '<BITBUCKET_EMAIL>:<BITBUCKET_API_TOKEN>' \
  -H 'Accept: application/json' \
  'https://api.bitbucket.org/2.0/repositories/<workspace>/<repo-slug>/default-reviewers'
```

Collect the `uuid` field from each entry in the `values` array of the response.
Build `<reviewers-payload>` as a JSON array:

```json
[{ "uuid": "<uuid-1>" }, { "uuid": "<uuid-2>" }, ...]
```

If this request fails or returns an empty list, set `<reviewers-payload>` to `[]` and continue.

### Call the API

```
curl -X POST \
  -u '<BITBUCKET_EMAIL>:<BITBUCKET_API_TOKEN>' \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
    "title": "<title>",
    "description": "<body>",
    "source": { "branch": { "name": "<current-branch>" } },
    "destination": { "branch": { "name": "<base-branch>" } },
    "draft": true,
    "reviewers": <reviewers-payload>
  }' \
  'https://api.bitbucket.org/2.0/repositories/<workspace>/<repo-slug>/pullrequests'
```

On success, output the value of `links.html.href` from the response (the PR URL) and stop.

On failure, show the HTTP status and response body and ask the user how to proceed.

## Step 3c — Fallback (unknown remote)

Display the PR content in a formatted block so the user can create the PR manually:

```
--- PR READY TO SUBMIT ---

Title:
<title>

Body:
<body>

Branch: <current-branch> → <base-branch>
Remote: <remote-url>
--------------------------
```

# When to Apply

- A calling skill (e.g. `fix-npm-security-vulnerabilities`) instructs the agent to use this skill
  after preparing and committing changes.
- The user asks to "create a PR", "open a pull request", or "push and create a PR" for a
  repository.
