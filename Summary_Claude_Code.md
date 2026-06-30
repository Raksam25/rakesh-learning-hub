# Claude Code — Complete Team Reference Guide

> From first install to advanced automation: everything your team needs to use Claude Code effectively in daily development and testing.

**Sources:** [Claude Code 101](https://anthropic.skilljar.com/claude-code-101) · [Claude Code in Action](https://anthropic.skilljar.com/claude-code-in-action/303233) · [AI Fluency Framework](https://anthropic.skilljar.com/ai-fluency-framework-foundations) · [Claude with the Anthropic API](https://anthropic.skilljar.com/claude-with-the-anthropic-api) · [Official docs](https://code.claude.com/docs)

---

## Table of Contents

**Part 1 — Foundations**
1. [What is Claude Code?](#1-what-is-claude-code)
2. [How Claude Code Works — The Agentic Loop](#2-how-claude-code-works--the-agentic-loop)
3. [Tools Available to Claude](#3-tools-available-to-claude)
4. [System Requirements](#4-system-requirements)
5. [Installation](#5-installation)
6. [Authentication & Account Types](#6-authentication--account-types)
7. [Your First Session](#7-your-first-session)

**Part 2 — Daily Workflows**
8. [The Explore → Plan → Code → Commit Workflow](#8-the-explore--plan--code--commit-workflow)
9. [Permission Modes](#9-permission-modes)
10. [Context Window & Context Management](#10-context-window--context-management)
11. [Keyboard Shortcuts & Interactive Features](#11-keyboard-shortcuts--interactive-features)
12. [Slash Commands Reference](#12-slash-commands-reference)
13. [Common Development Workflows](#13-common-development-workflows)
14. [Working with Images & Files](#14-working-with-images--files)
15. [Sessions: Resume, Fork, Worktrees](#15-sessions-resume-fork-worktrees)

**Part 3 — Persistent Memory & Configuration**
16. [CLAUDE.md — Project Memory](#16-claudemd--project-memory)
17. [Auto Memory](#17-auto-memory)
18. [Settings & Configuration Files](#18-settings--configuration-files)

**Part 4 — Extending Claude Code**
19. [Skills (Custom Commands)](#19-skills-custom-commands)
20. [Subagents](#20-subagents)
21. [MCP Servers — External Tools Integration](#21-mcp-servers--external-tools-integration)
22. [Hooks — Automation & Guardrails](#22-hooks--automation--guardrails)

**Part 5 — Advanced Topics**
23. [GitHub Integration & CI/CD](#23-github-integration--cicd)
24. [Non-Interactive / Headless Mode](#24-non-interactive--headless-mode)
25. [The Claude Code SDK](#25-the-claude-code-sdk)
26. [Claude with the Anthropic API](#26-claude-with-the-anthropic-api)
27. [AI Fluency: The 4D Framework](#27-ai-fluency-the-4d-framework)
28. [Security & Permissions Best Practices](#28-security--permissions-best-practices)
29. [Troubleshooting](#29-troubleshooting)
30. [Further Reading](#30-further-reading)

---

# Part 1 — Foundations

## 1. What is Claude Code?

Claude Code is an **agentic AI coding assistant** that runs in your terminal. Unlike chat tools, it can:

- Read your entire codebase, not just the current file
- Edit files, run shell commands, and execute tests
- Search the web, fetch documentation, look up errors
- Coordinate changes across multiple files simultaneously
- Delegate subtasks to specialized subagents

> **Key distinction:** Chat-based AI sees what you paste. Claude Code sees your whole project and can act on it.

### Where Claude Code runs

| Interface | Access |
|---|---|
| Terminal (CLI) | `claude` in any directory |
| VS Code extension | Sidebar panel + inline completions |
| JetBrains IDEs | Integrated panel |
| Claude Desktop app | GUI wrapper around the CLI |
| claude.ai/code | Web browser |
| Remote Control | Control your local machine from the web UI |
| Slack | `/claude` in channels |
| GitHub Actions | CI/CD automation |

---

## 2. How Claude Code Works — The Agentic Loop

Every task goes through three phases that blend together:

```
Your prompt
    ↓
┌─────────────────────┐
│  Gather Context      │  Reads files, searches code, fetches docs
│  Take Action         │  Edits files, runs commands, calls APIs
│  Verify Results      │  Runs tests, checks output, corrects errors
└─────────────────────┘
    ↓ (loops until done)
Final result
```

Claude decides how many loops to run. A question might need only one context-gather pass. A bug fix might cycle through all three phases a dozen times. You can interrupt at any point to steer in a different direction.

**The three components:**

| Component | Role |
|---|---|
| **Model** (Claude Sonnet/Opus/Haiku) | Reasons about what to do next |
| **Tools** | Let Claude act: read/edit/run/search |
| **Agentic harness** (Claude Code) | Manages context, permissions, tool execution |

**Sessions are independent.** Each new session starts with a fresh context window — no memory of previous conversations unless you use CLAUDE.md or auto memory.

---

## 3. Tools Available to Claude

Claude chooses tools automatically based on the task:

| Category | What Claude can do |
|---|---|
| **File operations** | Read files, edit code, create/rename/reorganize files |
| **Search** | Find files by pattern, search content with regex, explore codebases |
| **Execution** | Run shell commands, start servers, run tests, use git |
| **Web** | Search the web, fetch documentation, look up error messages |
| **Code intelligence** | See type errors after edits, jump to definitions, find references (requires plugins) |
| **Subagents** | Spawn isolated assistants for delegated work |

**Example — what Claude does for "fix the failing tests":**
1. Runs `npm test` to see what's failing
2. Reads the error output
3. Searches for the relevant source files
4. Reads those files to understand the code
5. Edits the files to fix the issue
6. Runs the tests again to verify

---

## 4. System Requirements

| Requirement | Minimum |
|---|---|
| **OS** | macOS 13.0+, Windows 10 1809+, Ubuntu 20.04+, Debian 10+, Alpine 3.19+ |
| **Hardware** | 4 GB+ RAM, x64 or ARM64 processor |
| **Network** | Internet connection required |
| **Shell** | Bash, Zsh, PowerShell, or CMD |
| **Account** | Claude Pro, Max, Team, Enterprise, or API key |

---

## 5. Installation

### Quick install (recommended)

```bash
# macOS / Linux / WSL
curl -fsSL https://claude.ai/install.sh | bash

# Windows PowerShell
irm https://claude.ai/install.ps1 | iex

# Windows CMD
curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
```

### Alternative install methods

```bash
# Homebrew (macOS)
brew install --cask claude-code           # stable channel (~1 week behind)
brew install --cask claude-code@latest    # latest channel

# WinGet (Windows)
winget install Anthropic.ClaudeCode

# npm (Node 18+ required)
npm install -g @anthropic-ai/claude-code

# apt (Debian/Ubuntu)
sudo install -d -m 0755 /etc/apt/keyrings
sudo curl -fsSL https://downloads.claude.ai/keys/claude-code.asc \
  -o /etc/apt/keyrings/claude-code.asc
echo "deb [signed-by=/etc/apt/keyrings/claude-code.asc] https://downloads.claude.ai/claude-code/apt/stable stable main" \
  | sudo tee /etc/apt/sources.list.d/claude-code.list
sudo apt update && sudo apt install claude-code
```

### Verify and check health

```bash
claude --version
claude doctor      # diagnoses common configuration issues
```

### Update

```bash
claude update                        # manual update
# Native installs update automatically in the background
brew upgrade claude-code             # Homebrew
winget upgrade Anthropic.ClaudeCode  # WinGet
npm install -g @anthropic-ai/claude-code@latest  # npm
```

### Uninstall

```bash
# macOS/Linux native
rm -f ~/.local/bin/claude && rm -rf ~/.local/share/claude

# Remove all config/settings (irreversible)
rm -rf ~/.claude ~/.claude.json
```

---

## 6. Authentication & Account Types

After installing, run `claude` and follow the browser prompts to log in.

| Account type | How to authenticate |
|---|---|
| Claude Pro/Max/Team | Browser login via claude.ai |
| Enterprise | Browser login via your org's SSO |
| API key (direct) | Set `ANTHROPIC_API_KEY` env var |
| Amazon Bedrock | Configure AWS credentials |
| Google Vertex AI | Configure GCP credentials |

> **Note:** The free Claude.ai plan does NOT include Claude Code access.

---

## 7. Your First Session

```bash
cd /path/to/your/project
claude
```

Claude Code starts and reads your project context. Try these first prompts:

```
give me an overview of this codebase
```
```
what are the main architectural patterns here?
```
```
/init
```

`/init` creates a `CLAUDE.md` file customized for your project by analyzing your codebase.

---

# Part 2 — Daily Workflows

## 8. The Explore → Plan → Code → Commit Workflow

This is the core daily workflow for any non-trivial task:

### Step 1: Explore

Let Claude understand the codebase first, without making changes:

```
Read src/auth/ and understand how we handle sessions.
```

Or use **Plan Mode** (`Shift+Tab` twice or `--permission-mode plan`): Claude reads files and builds a plan but cannot edit any source files.

### Step 2: Plan

```
Based on what you've read, create a plan for adding OAuth support.
Don't write any code yet, just outline the approach.
```

Review and refine the plan through conversation before committing to it.

### Step 3: Code

Once the plan is approved:
```
Now implement the OAuth changes we discussed.
Write failing tests first, then make them pass.
```

### Step 4: Commit

```
Run the tests to verify everything passes, then commit the changes.
```

Or guide it:
```
create a pr for my changes
```

### Why this workflow works

- **Explore first** → fewer wrong assumptions, fewer wasted edits
- **Plan before coding** → you catch problems in the plan, not in the diff
- **Test-first coding** → Claude verifies its own work automatically
- **Small iterations** → press `Esc` at any time to redirect

---

## 9. Permission Modes

Press `Shift+Tab` to cycle through modes, or set with `--permission-mode`:

| Mode | Behavior | Best for |
|---|---|---|
| **Default** | Asks before file edits and shell commands | Normal work, unfamiliar codebases |
| **Accept Edits** | Auto-accepts file edits and filesystem commands; still asks for other commands | Faster iteration on trusted projects |
| **Plan Mode** | Reads and proposes plans; no source file edits | Reviewing what Claude will do before committing |
| **Auto Mode** | Background safety classifier evaluates actions | Research preview; experienced users |

### Pre-approve specific commands

In `.claude/settings.json`:
```json
{
  "permissions": {
    "allow": ["Bash(npm test)", "Bash(git status)", "Bash(git diff *)"]
  }
}
```

---

## 10. Context Window & Context Management

Claude's context window holds your conversation, file contents, command outputs, CLAUDE.md, loaded skills, and system instructions. As you work, it fills up.

### See what's consuming context

```
/context
```

### Commands for managing context

| Command | What it does |
|---|---|
| `/compact` | Summarize the conversation and free space, preserving key code |
| `/compact focus on the API changes` | Compact with a specific focus |
| `/clear` | Start a fresh session (conversation cleared, project files still accessible) |
| `/context` | See a breakdown of what's using context space |

### What survives compaction

- Instructions from CLAUDE.md are re-injected
- Recently invoked skills are re-attached
- Your conversation summary

### Auto-compaction

Claude Code auto-compacts when the context window is nearly full. To control what's preserved, add a `Compact Instructions` section to your CLAUDE.md:

```markdown
## Compact Instructions
When compacting, always preserve:
- The current task description
- Any failing test output
- The list of files modified so far
```

### Tips for large codebases

- Use **subagents** for research tasks — they get their own clean context window
- Skills load on demand — only their descriptions are in context until invoked
- Use `@filename` to explicitly include a file rather than asking Claude to find it
- Enable MCP tool search deferred loading (default) — tool schemas load only when used

---

## 11. Keyboard Shortcuts & Interactive Features

### Essential shortcuts

| Shortcut | Action |
|---|---|
| `Esc` | Interrupt Claude mid-turn; Claude keeps work done so far |
| `Esc` + `Esc` | Clear input draft OR open rewind menu (when input is empty) |
| `Shift+Tab` | Cycle permission modes (default → accept edits → plan → ...) |
| `Ctrl+C` | Interrupt running operation; second press exits |
| `Ctrl+B` | Background a running task (tmux users press twice) |
| `Ctrl+O` | Toggle transcript viewer (shows detailed tool usage) |
| `Ctrl+R` | Reverse search through command history |
| `Ctrl+V` / `Cmd+V` | Paste image from clipboard |
| `Ctrl+T` | Toggle task list |
| `Option+P` / `Alt+P` | Switch model without clearing prompt |
| `Option+T` / `Alt+T` | Toggle extended thinking mode |
| `Ctrl+G` | Open prompt in external text editor |

### Multiline input

| Method | Works in |
|---|---|
| `\` + Enter | All terminals |
| `Shift+Enter` | iTerm2, WezTerm, Ghostty, Kitty, Warp, Apple Terminal, Windows Terminal |
| `Ctrl+J` | Any terminal (no configuration needed) |
| `Option+Enter` | macOS (after enabling Option as Meta) |

### Input prefixes

| Prefix | Behavior |
|---|---|
| `/` | Open skills/commands menu |
| `!` | Shell mode — run a command directly, output added to context |
| `@` | File/directory path autocomplete |

### Shell mode (`!` prefix)

```bash
! npm test
! git status
! ls -la src/
```

Shell mode adds the command and its output to the conversation context. As of v2.1.186, Claude automatically responds to shell command output — no need to send a second prompt.

### Side questions (`/btw`)

Ask a quick question without adding it to conversation history:
```
/btw what was the name of the config file again?
```

- Sees your full conversation context
- No tool access (can't read new files)
- Answer is ephemeral — press `Space` to dismiss
- Press `f` to fork it into a full session if you need to continue

---

## 12. Slash Commands Reference

Type `/` to see all available commands. Key built-in commands:

| Command | Purpose |
|---|---|
| `/init` | Generate a CLAUDE.md for this project |
| `/compact` | Compact conversation to free context |
| `/clear` | Start a fresh conversation |
| `/context` | Show context usage breakdown |
| `/memory` | Browse and edit CLAUDE.md and auto memory files |
| `/model` | Switch AI model for the session |
| `/config` | Open settings (theme, editor mode, auto-updates, etc.) |
| `/resume` | Resume a previous session |
| `/agents` | Manage custom subagents |
| `/skills` | Browse and manage skills |
| `/permissions` | Manage tool permissions |
| `/doctor` | Diagnose installation and configuration issues |
| `/mcp` | Manage MCP server connections |
| `/theme` | Change color theme |
| `/btw` | Ask a side question without affecting conversation |
| `/fork` | Fork the current conversation to a new session |
| `/recap` | Generate a session summary |
| `/powerup` | Interactive lessons with animated demos |
| `/voice` | Configure voice dictation |
| `/loop` | Run a command on a recurring interval |
| `/code-review` | Review current diff (bundled skill) |
| `/debug` | Debug errors and test failures (bundled skill) |
| `/run` | Launch and test your app (bundled skill) |
| `/verify` | Confirm a code change works in the running app (bundled skill) |
| `/batch` | Run multiple tasks in parallel (bundled skill) |

---

## 13. Common Development Workflows

### Understand a new codebase

```
give me an overview of this codebase
explain the main architecture patterns used here
what are the key data models?
how is authentication handled?
trace the login process from front-end to database
```

### Fix a bug

```
I'm seeing this error when I run npm test: [paste error]
suggest a few ways to fix this
update user.ts to add the null check you suggested
run the tests to verify the fix
```

### Refactor code

```
find deprecated API usage in our codebase
suggest how to refactor utils.js to use modern JavaScript features
refactor utils.js to ES2024 features while maintaining the same behavior
run the tests for the refactored code
```

### Write tests

```
find functions in NotificationsService.swift not covered by tests
add tests for the notification service
add test cases for edge conditions
run the new tests and fix any failures
```

### Create a PR

```
summarize the changes I've made to the authentication module
create a pr
enhance the PR description with more context about the security improvements
```

Resume a PR session later:
```bash
claude --from-pr 123    # or paste the PR URL into /resume picker
```

### Documentation

```
find functions without proper JSDoc comments in the auth module
add JSDoc comments to undocumented functions in auth.js
check if the documentation follows our project standards
```

### Parallel work with worktrees

Work on two features simultaneously without collision:

```bash
# Terminal 1: feature branch
claude --worktree feature-auth

# Terminal 2: bug fix branch (isolated)
claude --worktree bugfix-payment
```

Each worktree is a separate git checkout on its own branch with its own Claude session.

---

## 14. Working with Images & Files

### Add images to the conversation

```bash
# Method 1: Paste from clipboard
Ctrl+V  (or Cmd+V on macOS iTerm2)

# Method 2: Drag and drop into the Claude Code window

# Method 3: Reference by path
Analyze this image: /path/to/screenshot.png
```

### Visual prompts

```
Here's a screenshot of the error. What's causing it?
This is our current database schema. How should we modify it?
Generate CSS to match this design mockup
What HTML structure would recreate this component?
```

### Reference files with `@`

```
Explain the logic in @src/utils/auth.js
What's the structure of @src/components?
Compare @src/old-module.ts and @src/new-module.ts
```

- File references include full file content
- Directory references include the file listing (not contents)
- `@ file` references also add CLAUDE.md files from that directory to context

---

## 15. Sessions: Resume, Fork, Worktrees

### Resume a session

```bash
claude --continue         # resume most recent session in this directory
claude --resume           # choose from a list of sessions
/resume                   # picker within a running session
```

### Fork a session

Branching preserves the original and creates a new session with the same history:

```bash
claude --fork-session     # fork from CLI
/branch                   # fork from within a session
/fork do X differently    # fork with a directive
```

### Session storage

Every session is saved as a JSONL file under `~/.claude/projects/`. Sessions are independent — no cross-session history by default. Use CLAUDE.md and auto memory for persistence.

### Worktrees for parallel sessions

```bash
claude --worktree branch-name    # creates a new git worktree + isolated session
```

Each worktree = separate directory + separate branch + separate Claude session. Use this when you need to work on two independent tasks simultaneously.

---

# Part 3 — Persistent Memory & Configuration

## 16. CLAUDE.md — Project Memory

CLAUDE.md files give Claude instructions that persist across every session.

### File locations and scope

| Location | Scope | Shared with |
|---|---|---|
| `/Library/Application Support/ClaudeCode/CLAUDE.md` (macOS) | All users on the machine | Organization (managed) |
| `~/.claude/CLAUDE.md` | All your projects | Just you |
| `./CLAUDE.md` or `./.claude/CLAUDE.md` | This project | Team (commit to git) |
| `./CLAUDE.local.md` | This project | Just you (add to `.gitignore`) |

### Creating your CLAUDE.md

```
/init
```

This analyzes your codebase and generates a starting CLAUDE.md with:
- Build and test commands
- Project coding conventions
- Architecture notes
- Common workflows

### What to put in CLAUDE.md

**Good additions:**
- Build and test commands: `npm test`, `make build`
- Coding conventions: indentation, naming, file organization
- Project architecture: folder structure, key modules
- Always-on rules: "Never commit directly to main", "Always run linter before committing"
- Common workflows: how to deploy, how to create a PR

**Move to skills instead:**
- Multi-step procedures (e.g., deployment process)
- Instructions that only apply to one part of the codebase
- Long reference material

### Writing effective instructions

```markdown
# Project Setup
- Run `npm test` before committing
- Use `npm run dev` for local development
- API handlers live in `src/api/handlers/`

# Coding Standards
- Use 2-space indentation
- Functions named in camelCase, files in kebab-case
- All API endpoints must include input validation

# Compact Instructions
When compacting, always preserve:
- Current task description
- Any failing test output
```

**Rules:**
- Target under 200 lines per file — longer files consume more context
- Use markdown headers and bullets for structure
- Be concrete: "Use 2-space indentation" beats "Format code properly"

### Path-scoped rules with `.claude/rules/`

Create rules that only load when Claude works with matching files:

```markdown
.claude/rules/api.md:
---
paths:
  - "src/api/**/*.ts"
---
# API Rules
- All endpoints must include input validation
- Use the standard error response format
```

### Import additional files

```markdown
# CLAUDE.md
See @README for project overview.
See @package.json for available scripts.

# Git workflow
@docs/git-workflow.md
```

### CLAUDE.md for monorepos

Exclude irrelevant CLAUDE.md files from other teams:
```json
// .claude/settings.local.json
{
  "claudeMdExcludes": ["**/other-team/.claude/rules/**"]
}
```

---

## 17. Auto Memory

Claude automatically saves notes across sessions — build commands it discovers, debugging insights, code style preferences, architectural patterns.

### Enable/disable

Auto memory is on by default.

```json
// .claude/settings.json
{
  "autoMemoryEnabled": false
}
```

Or: `export CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`

### Where it lives

```
~/.claude/projects/<project>/memory/
├── MEMORY.md          # Index (first 200 lines loaded every session)
├── debugging.md       # Topic files (loaded on demand)
├── api-conventions.md
└── ...
```

### How to use it

Ask Claude to remember something:
```
always use pnpm, not npm — remember this
```

Ask Claude to consult memory:
```
check your memory before starting — do we have patterns for this?
```

Ask Claude to update memory after a task:
```
save what you learned about our deployment process to memory
```

### View and edit

```
/memory
```

Opens a browser of all memory files and CLAUDE.md files loaded in the current session.

---

## 18. Settings & Configuration Files

### Settings hierarchy (highest to lowest priority)

1. `--settings` CLI flag (current session only)
2. Managed policy settings (organization-wide, admins only)
3. `.claude/settings.local.json` (project, personal, gitignored)
4. `.claude/settings.json` (project, team-shared)
5. `~/.claude/settings.json` (user, all projects)

### Common settings

```json
// .claude/settings.json
{
  "model": "claude-sonnet-4-6",
  "autoUpdatesChannel": "stable",
  "permissions": {
    "allow": [
      "Bash(npm test)",
      "Bash(npm run lint)",
      "Bash(git status)",
      "Bash(git diff *)"
    ],
    "deny": [
      "Bash(rm -rf *)"
    ]
  },
  "autoMemoryEnabled": true,
  "disableBundledSkills": false,
  "env": {
    "NODE_ENV": "development"
  }
}
```

### Configure via `/config`

Run `/config` to change settings interactively:
- Theme (dark, light, custom)
- Editor mode (standard or vim)
- Auto-update channel (latest or stable)
- Prompt suggestions
- Session recap
- Model selection

---

# Part 4 — Extending Claude Code

## 19. Skills (Custom Commands)

Skills are reusable markdown instruction files. Create a `SKILL.md` file and invoke it with `/skill-name`.

### Quick example

```bash
mkdir -p ~/.claude/skills/fix-issue
```

```yaml
# ~/.claude/skills/fix-issue/SKILL.md
---
description: Fix a GitHub issue by number. Use when asked to fix or work on an issue.
disable-model-invocation: true
allowed-tools: Bash(gh *) Read Edit Bash(npm test)
---

Fix GitHub issue $ARGUMENTS following our coding standards.

1. Read the issue: !`gh issue view $ARGUMENTS`
2. Understand the requirements
3. Implement the fix
4. Write tests
5. Run `npm test` to verify
6. Create a commit with the issue reference
```

Invoke: `/fix-issue 123`

### Key frontmatter fields

| Field | Purpose |
|---|---|
| `description` | How Claude decides when to auto-trigger the skill |
| `disable-model-invocation: true` | Manual-only (you type `/skill-name`, Claude doesn't auto-trigger) |
| `allowed-tools` | Tools pre-approved for this skill (no permission prompt) |
| `context: fork` | Run in an isolated subagent |
| `model` | Override model for this skill |
| `paths` | Only activate for matching file paths |
| `arguments` | Named positional arguments: `$0`, `$1`, `$name` |

### Dynamic context injection

```yaml
---
description: Summarize git changes
---

## Current diff
!`git diff HEAD`

## Instructions
Summarize the above changes in 3 bullets. Flag any risks.
```

The `` !`command` `` syntax runs the command and injects its output before Claude reads the skill.

### Skill locations

| Path | Scope |
|---|---|
| `~/.claude/skills/<name>/SKILL.md` | All your projects (personal) |
| `.claude/skills/<name>/SKILL.md` | This project (commit to git to share with team) |

> See `agent_summary.md` for the complete Skills reference.

---

## 20. Subagents

Subagents are specialized assistants that run in their own isolated context window. Use them to keep your main conversation clean.

### Built-in subagents

| Subagent | Model | Tools | Use case |
|---|---|---|---|
| **Explore** | Haiku | Read-only | File search, codebase exploration |
| **Plan** | Inherits | Read-only | Research during plan mode |
| **General-purpose** | Inherits | All | Complex multi-step tasks |

### Create a custom subagent

```bash
/agents    # opens the subagent management UI
```

Or create a file:

```markdown
# .claude/agents/code-reviewer.md
---
name: code-reviewer
description: Reviews code for quality and security. Use proactively after code changes.
tools: Read, Grep, Glob, Bash
model: sonnet
---

You are a senior code reviewer. When invoked:
1. Run `git diff` to see recent changes
2. Review modified files
3. Report: Critical issues → Warnings → Suggestions
```

### Use subagents

```
Use the code-reviewer subagent to check my recent changes
@"code-reviewer (agent)" review the auth changes
```

```bash
claude --agent code-reviewer    # run the whole session as this subagent
```

### When to use subagents

- Task produces verbose output (test runs, log analysis, large searches)
- You want to enforce specific tool restrictions (read-only researcher)
- Independent parallel investigations
- Long context-heavy research that would flood your main session

> See `agent_summary.md` for the complete Subagents reference.

---

## 21. MCP Servers — External Tools Integration

MCP (Model Context Protocol) lets Claude connect to external tools, APIs, and data sources.

### Install an MCP server

```bash
# Example: GitHub MCP server
claude mcp add github -- npx @modelcontextprotocol/server-github

# Example: filesystem server
claude mcp add filesystem -- npx @modelcontextprotocol/server-filesystem /path/to/dir

# Example: Playwright browser automation
claude mcp add playwright -- npx @playwright/mcp@latest
```

### Manage MCP servers

```
/mcp            # list connected servers and their status
```

```bash
claude mcp list           # list configured servers
claude mcp remove github  # remove a server
```

### MCP server types

| Transport | Use case |
|---|---|
| **stdio** | Local tools (filesystem, git, databases) |
| **Streamable HTTP** | Remote services (GitHub, Slack, Sentry, Notion) |

### Configure in .mcp.json

```json
{
  "mcpServers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"]
    },
    "sentry": {
      "type": "http",
      "url": "https://mcp.sentry.io"
    }
  }
}
```

### MCP use cases in daily dev

| Server | What you can do |
|---|---|
| GitHub | Create issues, review PRs, manage repos without leaving Claude |
| Sentry | Query error logs, understand incidents |
| Playwright | Browser automation, screenshot capture, UI testing |
| Filesystem | Cross-directory access, network drives |
| Slack/Notion | Post updates, read docs directly from Claude |
| Databases | Query live data from within Claude Code sessions |

> See `mcp_summary.md` for the complete MCP reference.

---

## 22. Hooks — Automation & Guardrails

Hooks are shell commands, HTTP calls, or LLM prompts that fire automatically at specific points in Claude Code's lifecycle.

### Hook lifecycle events

| Event | When it fires | Can block |
|---|---|---|
| `SessionStart` | Session begins | No |
| `UserPromptSubmit` | You submit a message | Yes |
| `PreToolUse` | Before Claude uses any tool | Yes |
| `PostToolUse` | After a tool completes | No |
| `Stop` | Claude finishes a turn | Yes |
| `SessionEnd` | Session terminates | No |
| `SubagentStart` | A subagent begins | No |
| `SubagentStop` | A subagent completes | No |

### Configuration structure

```json
// .claude/settings.json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/run-linter.sh"
          }
        ]
      }
    ]
  }
}
```

### Hook types

| Type | Use case |
|---|---|
| `command` | Run a shell script |
| `http` | POST to a URL (remote validation, logging) |
| `mcp_tool` | Call an MCP server tool |
| `prompt` | Ask Claude to evaluate something |

### Exit codes

| Code | Meaning |
|---|---|
| `0` | Success, proceed |
| `2` | Block the action (prevents tool execution) |
| Other | Non-blocking error (logs warning, continues) |

### Practical hook examples

#### Auto-lint after file edits

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/lint.sh",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

#### Block dangerous shell commands

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "./.claude/hooks/validate-bash.sh" }]
      }
    ]
  }
}
```

```bash
#!/bin/bash
# .claude/hooks/validate-bash.sh
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

if echo "$COMMAND" | grep -qE '\brm -rf\b'; then
  echo "Blocked: destructive rm command" >&2
  exit 2
fi
exit 0
```

#### Load context on session start

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [{ "type": "command", "command": "./.claude/hooks/load-context.sh" }]
      }
    ]
  }
}
```

```bash
#!/bin/bash
# .claude/hooks/load-context.sh
BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null)
CHANGES=$(git status --short 2>/dev/null | wc -l)

jq -n --arg branch "$BRANCH" --arg changes "$CHANGES" '{
  hookSpecificOutput: {
    hookEventName: "SessionStart",
    additionalContext: "Current branch: \($branch), uncommitted changes: \($changes) files"
  }
}'
```

#### Format JSON output from hooks

```json
{
  "continue": true,
  "suppressOutput": false,
  "systemMessage": "Warning: large file detected",
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "File exceeds 1MB limit"
  }
}
```

---

# Part 5 — Advanced Topics

## 23. GitHub Integration & CI/CD

### Set up automated code review

Configure Claude Code to run on every PR via GitHub Actions:

```yaml
# .github/workflows/claude-review.yml
name: Claude Code Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: anthropics/claude-code-action@beta
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

### Common GitHub workflows

```
# From Claude Code terminal:
summarize the changes I've made
create a pr
review the open PRs and add comments to anything that needs attention
check if there are any issues assigned to me
```

With the GitHub MCP server:
```
@github show open PRs that need review
@github create an issue for the bug we just found
```

### PR status in the footer

When on a branch with an open PR, Claude Code shows a clickable PR link in the footer:
- 🟢 Green = approved
- 🟡 Yellow = pending review
- 🔴 Red = changes requested
- ⚫ Gray = draft

`Cmd+Click` (macOS) or `Ctrl+Click` (Windows/Linux) to open it in your browser.

### Scheduled automation

| Option | Where it runs | Best for |
|---|---|---|
| **Routines** | Anthropic cloud | Tasks that need to run when your machine is off |
| **Desktop scheduled tasks** | Your local machine | Tasks needing access to local files/tools |
| **GitHub Actions** | CI pipeline | PR-triggered or repo-event workflows |
| **`/loop`** | Current session | Quick polling during active work |

---

## 24. Non-Interactive / Headless Mode

Run Claude Code in scripts, CI/CD, and batch processing:

```bash
# Pipe stdin
git log --oneline -20 | claude -p "summarize these commits"

# Pass a prompt directly
claude -p "review this file for security issues" --file src/auth.ts

# Output formats
claude -p "..." --output-format json
claude -p "..." --output-format stream-json

# Auto-accept all permissions (use carefully in CI)
claude -p "run tests and fix failures" --permission-mode acceptEdits
```

### Useful headless patterns

```bash
# Summarize every changed file in a PR
git diff --name-only HEAD~1 | xargs -I{} claude -p "summarize changes in {}"

# Auto-generate changelog
git log --oneline v1.0..HEAD | claude -p "generate a changelog from these commits"

# Pre-commit hook
claude -p "review this diff for security issues" <<< "$(git diff --staged)"
```

---

## 25. The Claude Code SDK

The SDK lets you build applications and automation on top of Claude Code programmatically.

### Install

```bash
npm install @anthropic-ai/claude-code
```

### Basic usage

```javascript
import { query } from "@anthropic-ai/claude-code";

const messages = [];
for await (const message of query({
  prompt: "Fix the failing tests in src/auth.test.ts",
  options: {
    maxTurns: 10,
    permissionMode: "acceptEdits"
  }
})) {
  messages.push(message);
  console.log(message);
}
```

### SDK use cases

- CI/CD automation: run Claude on every PR, auto-fix lint issues
- Batch processing: apply the same change across many repositories
- Custom agents: build specialized tools on top of Claude Code
- Integration with internal tooling: wire Claude into your dev platform

### Key SDK options

| Option | Description |
|---|---|
| `maxTurns` | Maximum agentic loop iterations |
| `permissionMode` | `default`, `acceptEdits`, `bypassPermissions`, `plan` |
| `model` | Override the model |
| `systemPrompt` | Append to the system prompt |
| `agents` | Define custom subagents inline |
| `mcpServers` | Configure MCP servers |

---

## 26. Claude with the Anthropic API

For teams building AI applications on top of Claude:

### Authentication

```python
import anthropic

client = anthropic.Anthropic(api_key="your-api-key")
```

```bash
export ANTHROPIC_API_KEY="your-api-key"
```

### Basic message

```python
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Explain this code: ..."}]
)
print(message.content[0].text)
```

### Multi-turn conversation

```python
messages = []

# Turn 1
messages.append({"role": "user", "content": "What is dependency injection?"})
response = client.messages.create(model="claude-sonnet-4-6", max_tokens=500, messages=messages)
messages.append({"role": "assistant", "content": response.content[0].text})

# Turn 2
messages.append({"role": "user", "content": "Show me an example in Python"})
response = client.messages.create(model="claude-sonnet-4-6", max_tokens=500, messages=messages)
```

### System prompts

```python
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system="You are a senior Python developer. Review code for security issues only.",
    messages=[{"role": "user", "content": "Review this: ..."}]
)
```

### Streaming responses

```python
with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Write a function to..."}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

### Tool use (function calling)

```python
tools = [{
    "name": "run_tests",
    "description": "Runs the test suite and returns results",
    "input_schema": {
        "type": "object",
        "properties": {
            "test_file": {"type": "string", "description": "Path to test file"}
        },
        "required": ["test_file"]
    }
}]

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "Run the auth tests"}]
)

# If Claude wants to use a tool:
if response.stop_reason == "tool_use":
    for block in response.content:
        if block.type == "tool_use":
            print(f"Claude wants to call: {block.name}")
            print(f"With args: {block.input}")
```

### Prompt caching (reduce costs)

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[{
        "type": "text",
        "text": "You are a code reviewer. Always check for...",
        "cache_control": {"type": "ephemeral"}  # Cache this for 5 minutes
    }],
    messages=[{"role": "user", "content": "Review: ..."}]
)
```

### Extended thinking (deeper reasoning)

```python
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=16000,
    thinking={"type": "enabled", "budget_tokens": 10000},
    messages=[{"role": "user", "content": "Design the database schema for..."}]
)
```

### Structured output

Use XML tags to organize responses:

```python
prompt = """Analyze this code for issues.
Use these XML tags:
<bugs>Critical bugs found</bugs>
<warnings>Non-critical issues</warnings>
<suggestions>Improvements</suggestions>
"""
```

### RAG (Retrieval-Augmented Generation)

```python
# 1. Chunk your documents
# 2. Create embeddings
# 3. Store in a vector DB (or BM25 index for lexical search)
# 4. On query: retrieve relevant chunks
# 5. Include chunks as context in the Claude prompt

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": f"Context:\n{retrieved_chunks}\n\nQuestion: {user_question}"
    }]
)
```

### Current model IDs

| Model | ID | Best for |
|---|---|---|
| Fable 5 | `claude-fable-5` | Frontier tasks |
| Opus 4.8 | `claude-opus-4-8` | Complex reasoning, architecture |
| Sonnet 4.6 | `claude-sonnet-4-6` | Most coding tasks (best balance) |
| Haiku 4.5 | `claude-haiku-4-5-20251001` | Fast, cheap, simple tasks |

---

## 27. AI Fluency: The 4D Framework

From the AI Fluency Framework course — a practical model for working effectively with AI:

### The 4 Ds

| Pillar | What it means |
|---|---|
| **Delegation** | Knowing which tasks to assign to AI and which to keep human |
| **Description** | Writing clear, effective prompts and communication |
| **Discernment** | Critically evaluating AI outputs, catching errors and hallucinations |
| **Diligence** | Responsible, ethical oversight throughout AI interactions |

### Delegation — What to give Claude

**Good candidates for delegation:**
- Repetitive, well-defined tasks (generating boilerplate, writing tests)
- Research and summarization (reading documentation, exploring codebases)
- First drafts (PRs, commit messages, comments)
- Verification (running tests, checking linting)
- Pattern-matching (finding all usages of a deprecated API)

**Keep human judgment for:**
- Architecture decisions with long-term consequences
- Security-sensitive code paths
- Business logic that requires domain expertise
- Code that needs to pass code review for understanding, not just correctness

### Description — The Description-Discernment Loop

1. **Write a prompt** → submit to Claude
2. **Evaluate the output** → does it meet your intent?
3. **Refine the description** → be more specific about what was wrong
4. **Repeat** until satisfied

**Prompt improvement tips:**
- Name specific files and line numbers when possible
- State constraints upfront ("must not change the public API")
- Describe the desired outcome, not the steps
- Give Claude something to verify against (test cases, examples)
- Use XML tags for structured instructions

### Discernment — Evaluating AI Output

Never blindly accept Claude's output. Check:

| Dimension | What to look for |
|---|---|
| **Correctness** | Does the code actually do what was asked? |
| **Safety** | Any security vulnerabilities (XSS, SQL injection, exposed secrets)? |
| **Completeness** | Are there edge cases or error paths missing? |
| **Style** | Does it match the project's conventions? |
| **Tests** | Do the tests actually verify the intended behavior, or are they trivially passing? |

### Diligence — Responsible Use

- Review AI-generated code before committing (never auto-merge without review)
- Don't include sensitive data (API keys, PII) in prompts
- Understand the code Claude writes — don't ship what you can't explain
- Use hooks to enforce guardrails automatically
- Keep humans in the loop for high-stakes decisions

---

## 28. Security & Permissions Best Practices

### The principle of least privilege

Only grant Claude the permissions it needs for the current task.

```json
// For a read-only research session
{
  "permissions": {
    "deny": ["Write", "Edit", "Bash(rm *)", "Bash(git push *)"]
  }
}
```

### Dangerous command protection

Always add a PreToolUse hook for destructive commands:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{"type": "command", "command": "./.claude/hooks/validate-bash.sh"}]
      }
    ]
  }
}
```

Block patterns to add to your validation script:
```bash
rm -rf /   # root deletion
git push --force  # force push to main
DROP TABLE  # database destructive operations
curl ... | bash   # remote script execution
```

### Protect sensitive files

```json
{
  "permissions": {
    "deny": [
      "Write(.env*)",
      "Write(*secrets*)",
      "Bash(cat .env*)"
    ]
  }
}
```

### Review project skills before trusting

When opening a project with `.claude/skills/`, Claude Code shows a workspace trust dialog. Review what tools and permissions those skills request before accepting.

### Team settings checklist

```json
// .claude/settings.json (commit to git)
{
  "permissions": {
    "allow": [
      "Bash(npm test)",
      "Bash(npm run lint)",
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log *)"
    ],
    "deny": [
      "Bash(git push --force *)",
      "Bash(npm publish *)"
    ]
  }
}
```

---

## 29. Troubleshooting

### Diagnose issues

```bash
claude doctor    # comprehensive health check
claude --version # verify installation
```

### Common problems

| Symptom | Likely cause | Fix |
|---|---|---|
| `command not found` | Install didn't add to PATH | Restart terminal; check `~/.local/bin` is in PATH |
| Login loop in browser | Cookie/auth issue | Clear browser cookies for claude.ai, try again |
| Claude not following CLAUDE.md | File not in loaded path, or instructions too vague | Run `/memory` to verify it's loaded; make instructions more specific |
| Context fills up too fast | Large files or verbose tool outputs | Use subagents for research; use `/compact`; trim CLAUDE.md |
| Skill not triggering | Description doesn't match your phrasing | Make description include exact keywords you use |
| Hook not running | Wrong matcher or script path | Check matcher matches tool name exactly; use absolute paths |
| MCP server not connecting | Config error or missing binary | Run `/mcp` to see error; check the server binary is installed |
| Permission prompts too frequent | Not pre-approving common commands | Add `allow` rules in settings.json |

### Get more help

```bash
claude doctor                  # runs a full diagnostic
/doctor                        # same from within a session
claude --help                  # CLI flag reference
claude mcp doctor <server>     # diagnose a specific MCP server
```

### Debug configuration

```
/debug-your-config             # interactive config debugger skill (if installed)
```

---

## 30. Further Reading

| Resource | URL |
|---|---|
| Claude Code 101 course | https://anthropic.skilljar.com/claude-code-101 |
| Claude Code in Action course | https://anthropic.skilljar.com/claude-code-in-action/303233 |
| AI Fluency Framework | https://anthropic.skilljar.com/ai-fluency-framework-foundations |
| Claude with the Anthropic API | https://anthropic.skilljar.com/claude-with-the-anthropic-api |
| Claude Code official docs | https://code.claude.com/docs |
| How Claude Code works | https://code.claude.com/docs/en/how-claude-code-works |
| Installation guide | https://code.claude.com/docs/en/getting-started |
| CLAUDE.md memory | https://code.claude.com/docs/en/memory |
| Interactive mode reference | https://code.claude.com/docs/en/interactive-mode |
| Common workflows | https://code.claude.com/docs/en/common-workflows |
| Hooks reference | https://code.claude.com/docs/en/hooks |
| Skills reference | https://code.claude.com/docs/en/skills |
| Subagents reference | https://code.claude.com/docs/en/sub-agents |
| MCP integration | https://code.claude.com/docs/en/mcp |
| GitHub Actions | https://code.claude.com/docs/en/github-actions |
| Permissions | https://code.claude.com/docs/en/permissions |
| Anthropic API docs | https://docs.anthropic.com |
| Anthropic SDK (Python) | https://github.com/anthropics/anthropic-sdk-python |
| Anthropic SDK (TypeScript) | https://github.com/anthropics/anthropic-sdk-typescript |
| Agent Skills open standard | https://agentskills.io |

---

*Generated from Anthropic Academy courses and official Claude Code documentation — June 2026.*
