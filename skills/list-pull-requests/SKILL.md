---
name: list-pull-requests
description: "Lists all open pull requests and their descriptions for a repository. Apply when the user asks to see, list, or read open pull requests or PRs."
---

# List Pull Requests

## Preconditions

Infer the repository path from context (current working directory or explicitly provided path).

## Step 1 — Detect remote type

Run the following, substituting `<repo-path>` with the identified repository path:

```
git -C <repo-path> remote get-url origin
```

- Remote URL contains `github.com` → proceed with **Step 2a (GitHub)**
- Remote URL contains `bitbucket.org` → proceed with **Step 2b (Bitbucket Cloud)**
- Anything else → skip to **Step 2c (Fallback)**

## Step 2a — GitHub (gh CLI)

Requires `gh` to be installed and authenticated (`gh auth status`).

Extract `<owner>/<repo>` from the remote URL. Then run:

```
gh pr list -R <owner>/<repo> --state open --json number,title,author,body,url,headRefName,baseRefName,createdAt
```

Format and display each PR using the output format below.

If there are no open PRs, display:

> No open pull requests found for `<owner>/<repo>`.

## Step 2b — Bitbucket Cloud (REST API v2)

### Credentials

Read `BITBUCKET_EMAIL` (your Atlassian account email) and `BITBUCKET_API_TOKEN` from the file:

```
<path-to-this-skill>/.env
```

Where `<path-to-this-skill>` is the directory containing this `SKILL.md` file (i.e.
`skills/list-pull-requests/.env` relative to the root of the skill library).

The API token must have the `pullrequests:read` scope.

If the `.env` file is missing or either variable is unset, stop and tell the user:

> **Bitbucket credentials not found.**
> Copy `skills/list-pull-requests/.env.example` to `skills/list-pull-requests/.env` and fill in
> your Atlassian account email (`BITBUCKET_EMAIL`) and an API token with the `pullrequests:read`
> scope (`BITBUCKET_API_TOKEN`).
> Then re-run this skill.

### Extract workspace and repo slug

Parse `<workspace>` and `<repo-slug>` from the remote URL:

- HTTPS format: `https://bitbucket.org/<workspace>/<repo-slug>.git`
- SSH format: `git@bitbucket.org:<workspace>/<repo-slug>.git`

### Call the API

```
curl -X GET \
  -u '<BITBUCKET_EMAIL>:<BITBUCKET_API_TOKEN>' \
  -H 'Accept: application/json' \
  'https://api.bitbucket.org/2.0/repositories/<workspace>/<repo-slug>/pullrequests?state=OPEN'
```

If the response includes a `next` link, paginate by repeating the request with the `next` URL until
all pages are retrieved.

Extract the following fields from each entry in the `values` array:

- `id` — PR number
- `title`
- `description`
- `author.display_name`
- `source.branch.name` — head branch
- `destination.branch.name` — base branch
- `created_on`
- `links.html.href` — PR URL

Format and display each PR using the output format below.

If `values` is empty, display:

> No open pull requests found for `<workspace>/<repo-slug>`.

On failure, show the HTTP status and response body and ask the user how to proceed.

## Step 2c — Fallback (unknown remote)

Display the remote URL and instruct the user to check PRs manually:

```
Remote: <remote-url>

Unable to list pull requests automatically for this remote type.
Please visit the repository's web interface to view open pull requests.
```

## Output Format

For each PR, display:

```
#<number> — <title>
  Author:      <author>
  Branch:      <head> → <base>
  Created:     <date>
  URL:         <url>
  Description: <body truncated to ~500 chars if longer>
```

If the description is empty, omit the Description line.

# When to Apply

- The user asks to "list PRs", "show open pull requests", "what PRs are open", or "read pull
  requests" for a repository.
- A calling skill needs an overview of in-progress work before taking further action.
