# AI Agent Skills & Sub-Agents

Central repository for managing AI agent skills and sub-agents. Designed to be symlinked into tool-specific configuration directories (Cursor, Claude Code, etc.).

## Skills Overview

| Skill | Description |
|-------|-------------|
| `git-commit-messages` | Commit message structure, wording, and conventions. |
| `git-branch-workflow` | Best practices for creating, naming, and managing Git branches. |
| `create-pull-request` | Creates a draft pull request. |
| `list-pull-requests` | Lists all open pull requests and their descriptions for a repository. |
| `fix-npm-security-vulnerabilities` | Full workflow: audits and fixes npm security vulnerabilities in repositories containing a package.json, through changelog review and PR creation. |

## Symlink Setup

Symlink the directories into your AI tool's config location. Back up any existing directories first.

Installation can be **system-wide** (e.g. `~/.claude/`) or **project-specific**
(e.g. `.claude/` inside a project repo). System-wide installation makes skills
available in every project; project-specific installation scopes them to a single
repo.

This repository recommends system-wide installation — the skills here are written
to be broadly useful regardless of project. Use project-specific installation only
for skills that apply to one repository.

### Codex CLI

```bash
mkdir -p ~/.codex
ln -s /path/to/ai/skills ~/.codex/
ln -s /path/to/ai/agents ~/.codex/
```

### Claude Code / GitHub Copilot CLI

```bash
mkdir -p ~/.claude
ln -s /path/to/ai/skills ~/.claude/
ln -s /path/to/ai/agents ~/.claude/
```

### Cursor

```bash
ln -s /path/to/ai/skills ~/.cursor/
ln -s /path/to/ai/agents ~/.cursor/
```

### Removing symlinks

To remove a symlink without deleting the repo contents, use `rm` on the link itself (adjust the path for your tool):

```bash
rm ~/.cursor/skills
rm ~/.cursor/agents
```

## Structure

```
skills/          # One subdirectory per skill, each containing a SKILL.md
agents/          # One .md file per sub-agent
```

### Skills

Each skill is a directory with a `SKILL.md` file (and optional supporting files):

```
skills/
└── my-skill/
    ├── SKILL.md          # Required — instructions and metadata
    ├── reference.md      # Optional — detailed docs
    └── scripts/          # Optional — utility scripts
```

`SKILL.md` uses YAML frontmatter:

```markdown
---
name: my-skill
description: What this skill does and when to use it.
---

# My Skill

Instructions for the agent...
```

### Agents

Each sub-agent is a single `.md` file with YAML frontmatter and a system prompt body:

```
agents/
├── code-reviewer.md
└── debugger.md
```

```markdown
---
name: code-reviewer
description: Reviews code for quality and best practices.
permissionMode: plan        # Claude Code — restrict permissions (see table below)
readonly: true              # Cursor — restrict write access
---

You are a code reviewer. When invoked, analyze the code and provide
specific, actionable feedback.
```

#### Tool-specific frontmatter

| Field            | Tool        | Values                                                                                   |
|------------------|-------------|------------------------------------------------------------------------------------------|
| `permissionMode` | Claude Code | `default`, `acceptEdits`, `dontAsk`, `delegate`, `bypassPermissions`, `plan`             |
| `readonly`       | Cursor      | `true` / `false` — restricts the sub-agent to read-only operations                       |
| `model`          | Both        | Claude Code: `sonnet`, `opus`, `haiku`, `inherit`. Cursor: `fast`, `inherit`, or model ID |
| `is_background`  | Cursor      | `true` / `false` — run the sub-agent in the background without blocking                  |

Unknown fields are ignored by each tool, so both can coexist in the same file.

## Skill vs. Sub-Agent

Skills and sub-agents solve different problems:

| Aspect      | Skill                                                                     | Sub-Agent                                                      |
|-------------|---------------------------------------------------------------------------|----------------------------------------------------------------|
| What it is  | Reusable instructions, knowledge, or workflows                            | Isolated worker with its own context                           |
| Key benefit | Share content across contexts                                             | Context isolation — only a summary returns to caller           |
| Interaction | Interactive — runs inside the main agent, which keeps talking to the user | Autonomous — works on its own and presents a finished outcome  |
| Best for    | Reference material, invocable workflows                                   | Tasks that read many files, parallel work, specialized workers |

**Skills** come in two flavors:

- **Reference skills** provide knowledge the agent uses throughout a session (e.g., an API style guide).
- **Action skills** tell the agent to do something specific (e.g., a `/deploy` workflow).

**Sub-agents** are useful when you need context isolation or when the context window is getting full. A sub-agent might read dozens of files or run extensive searches, but the main conversation only receives a summary.
