# Agent Skills & Subagents in Claude Code — Complete Guide

> From zero to production: a team reference for understanding, building, and using Agent Skills and Subagents in daily development and testing workflows.

**Sources:** [Anthropic Academy — Introduction to Agent Skills](https://anthropic.skilljar.com/introduction-to-agent-skills) · [Introduction to Subagents](https://anthropic.skilljar.com/introduction-to-subagents) · [Claude Code official docs](https://code.claude.com/docs)

---

## Table of Contents

**Part 1 — Agent Skills**
1. [What Are Skills?](#1-what-are-skills)
2. [Skills vs Other Claude Code Features](#2-skills-vs-other-claude-code-features)
3. [Creating Your First Skill](#3-creating-your-first-skill)
4. [Where Skills Live](#4-where-skills-live)
5. [SKILL.md Frontmatter Reference](#5-skillmd-frontmatter-reference)
6. [Dynamic Context Injection](#6-dynamic-context-injection)
7. [Passing Arguments to Skills](#7-passing-arguments-to-skills)
8. [Supporting Files in Skills](#8-supporting-files-in-skills)
9. [Run Skills in a Subagent](#9-run-skills-in-a-subagent)
10. [Controlling Who Invokes a Skill](#10-controlling-who-invokes-a-skill)
11. [Tool Access in Skills](#11-tool-access-in-skills)
12. [Sharing Skills with Your Team](#12-sharing-skills-with-your-team)
13. [Evaluating and Iterating on Skills](#13-evaluating-and-iterating-on-skills)
14. [Troubleshooting Skills](#14-troubleshooting-skills)
15. [Bundled Skills Reference](#15-bundled-skills-reference)

**Part 2 — Subagents**
16. [What Are Subagents?](#16-what-are-subagents)
17. [Built-in Subagents](#17-built-in-subagents)
18. [Creating Your First Subagent](#18-creating-your-first-subagent)
19. [Subagent Scope and Storage Locations](#19-subagent-scope-and-storage-locations)
20. [Subagent Frontmatter Reference](#20-subagent-frontmatter-reference)
21. [Choosing a Model per Subagent](#21-choosing-a-model-per-subagent)
22. [Controlling Subagent Capabilities](#22-controlling-subagent-capabilities)
23. [Persistent Memory for Subagents](#23-persistent-memory-for-subagents)
24. [Hooks in Subagents](#24-hooks-in-subagents)
25. [Preloading Skills into Subagents](#25-preloading-skills-into-subagents)
26. [Invoking Subagents Explicitly](#26-invoking-subagents-explicitly)
27. [Foreground vs Background Subagents](#27-foreground-vs-background-subagents)
28. [Forking the Conversation](#28-forking-the-conversation)
29. [Nested Subagents](#29-nested-subagents)
30. [Common Subagent Patterns](#30-common-subagent-patterns)
31. [When to Use Skills vs Subagents vs Main Conversation](#31-when-to-use-skills-vs-subagents-vs-main-conversation)
32. [Ready-to-Use Subagent Examples](#32-ready-to-use-subagent-examples)
33. [Further Reading](#33-further-reading)

---

# Part 1 — Agent Skills

## 1. What Are Skills?

**Skills** are reusable markdown instruction files that Claude Code adds to its toolkit. Create a `SKILL.md` file and Claude can invoke it automatically when relevant, or you invoke it directly with `/skill-name`.

> **The core idea:** Teach Claude once, not repeatedly. Instead of pasting the same checklist, procedure, or context into every conversation, write it once as a skill.

### When to create a skill

| Signal | Example |
|---|---|
| You keep pasting the same instructions | "Always check for SQL injection in code reviews" |
| A CLAUDE.md section has grown into a procedure | A deployment checklist that's 30 lines long |
| You want a reusable slash command | `/deploy`, `/fix-issue`, `/summarize-changes` |
| You want background context loaded automatically | API conventions, coding style, legacy system docs |

### How skills save context

Unlike CLAUDE.md content (which is always in context), **a skill's body loads only when invoked**. Long reference docs cost almost nothing until you actually need them.

---

## 2. Skills vs Other Claude Code Features

| Feature | Best for | Loaded when |
|---|---|---|
| **CLAUDE.md** | Always-on facts, rules, preferences | Always (every session) |
| **Skills** | Procedures, workflows, reusable commands | On invocation only |
| **Subagents** | Isolated task execution, parallel work | When delegated a task |
| **Hooks** | Automated side effects on tool events | Event-triggered, not model-driven |
| **MCP servers** | Connecting to external data/APIs | When the server is configured |

---

## 3. Creating Your First Skill

This example creates a skill that summarizes uncommitted git changes and flags risks. It auto-triggers when you ask about your changes.

### Step 1 — Create the skill directory

```bash
# Personal skill (available in all your projects)
mkdir -p ~/.claude/skills/summarize-changes
```

### Step 2 — Write SKILL.md

Save to `~/.claude/skills/summarize-changes/SKILL.md`:

```yaml
---
description: Summarizes uncommitted changes and flags anything risky. Use when the user asks what changed, wants a commit message, or asks to review their diff.
---

## Current changes

!`git diff HEAD`

## Instructions

Summarize the changes above in two or three bullet points, then list any risks
you notice such as missing error handling, hardcoded values, or tests that need
updating. If the diff is empty, say there are no uncommitted changes.
```

**What's happening here:**
- The `---` block is YAML frontmatter — metadata Claude reads to decide when to use the skill
- The `description` is how Claude decides to auto-trigger the skill
- The `` !`git diff HEAD` `` line runs the shell command and injects its output *before* Claude reads the skill (see [Dynamic Context Injection](#6-dynamic-context-injection))

### Step 3 — Test the skill

Open Claude Code in a git project, make a small change, then:

```
# Let Claude invoke it automatically:
What did I change?

# Or invoke it directly:
/summarize-changes
```

---

## 4. Where Skills Live

| Location | Path | Who can use it |
|---|---|---|
| **Enterprise** | Managed settings directory | All users in the organization |
| **Personal** | `~/.claude/skills/<skill-name>/SKILL.md` | All your projects |
| **Project** | `.claude/skills/<skill-name>/SKILL.md` | This project only |
| **Plugin** | `<plugin>/skills/<skill-name>/SKILL.md` | Where plugin is enabled |

### Priority rules

When skills share the same name: **Enterprise > Personal > Project > Plugin**.

A project skill overrides a bundled skill of the same name — e.g., a `.claude/skills/code-review/SKILL.md` replaces the built-in `/code-review`.

### Skill directory structure

Each skill is a directory with `SKILL.md` as the entrypoint:

```
my-skill/
├── SKILL.md           # Main instructions (required)
├── template.md        # Optional template for Claude to fill in
├── examples/
│   └── sample.md      # Optional example output
└── scripts/
    └── validate.sh    # Optional script Claude can run
```

### Monorepo support

Skills load from `.claude/skills/` in your starting directory **and every parent up to the repo root**, so you don't lose project-root skills when working in a subdirectory. Nested packages can also have their own `.claude/skills/` for package-specific skills.

```
repo/
├── .claude/skills/deploy/         → /deploy (all packages)
└── apps/web/
    └── .claude/skills/deploy/     → /apps/web:deploy (web-specific override)
```

Type `/apps/web:deploy` to invoke the nested variant explicitly.

### Live change detection

Adding, editing, or removing a skill under `~/.claude/skills/` or `.claude/skills/` takes effect within the current session — **no restart needed** (except when creating a new top-level skills directory that didn't exist when the session started).

---

## 5. SKILL.md Frontmatter Reference

Frontmatter is YAML between `---` markers at the top of `SKILL.md`. All fields are optional except `description` (strongly recommended).

```yaml
---
name: my-skill
description: What this skill does and when to use it
when_to_use: Additional trigger phrases. Use when the user asks X or Y.
argument-hint: "[issue-number]"
arguments: [issue, branch]
disable-model-invocation: true
user-invocable: false
allowed-tools: Bash(git add *) Bash(git commit *)
disallowed-tools: AskUserQuestion
model: sonnet
effort: high
context: fork
agent: Explore
paths: "src/**/*.ts,apps/web/**"
shell: bash
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate.sh"
---
```

### Full field reference

| Field | Description |
|---|---|
| `name` | Display label in skill listings. Does NOT change the `/command` name (which comes from the directory name) |
| `description` | **Most important field.** Claude reads this to decide when to auto-invoke. Put the key use case first. Combined with `when_to_use`, capped at 1,536 characters |
| `when_to_use` | Additional trigger phrases appended to `description` |
| `argument-hint` | Hint shown during autocomplete, e.g., `[filename] [format]` |
| `arguments` | Named positional args for `$name` substitution: `arguments: [issue, branch]` → `$issue`, `$branch` |
| `disable-model-invocation` | `true` = only YOU can invoke (Claude won't trigger automatically). Use for `/deploy`, `/commit`, `/send-slack-message` |
| `user-invocable` | `false` = hidden from `/` menu; only Claude can auto-load. Use for background reference knowledge |
| `allowed-tools` | Tools pre-approved for this skill (no permission prompt). E.g., `Bash(git add *) Bash(git commit *)` |
| `disallowed-tools` | Tools blocked while this skill is active. Clears after your next message |
| `model` | Override the model for this skill's turn: `sonnet`, `opus`, `haiku`, `fable`, or a full model ID |
| `effort` | Override reasoning effort: `low`, `medium`, `high`, `xhigh`, `max` |
| `context` | `fork` = run in an isolated subagent |
| `agent` | Which subagent to use when `context: fork` (e.g., `Explore`, `Plan`, `general-purpose`, or custom) |
| `paths` | Glob patterns: skill auto-activates only when working on matching files. E.g., `"src/**/*.ts"` |
| `shell` | Shell for `` !`command` `` blocks: `bash` (default) or `powershell` |
| `hooks` | Hooks scoped to this skill's lifecycle |

### Invocation behavior table

| Frontmatter | You can invoke | Claude can invoke | Description in context |
|---|---|---|---|
| (default) | Yes | Yes | Always present |
| `disable-model-invocation: true` | Yes | No | Not loaded into context |
| `user-invocable: false` | No | Yes | Always present |

---

## 6. Dynamic Context Injection

The `` !`<command>` `` syntax runs a shell command **before** the skill content is sent to Claude. The output replaces the placeholder inline.

```yaml
---
name: pr-summary
description: Summarize the current pull request
context: fork
agent: Explore
allowed-tools: Bash(gh *)
---

## Pull request context
- PR diff: !`gh pr diff`
- PR comments: !`gh pr view --comments`
- Changed files: !`gh pr diff --name-only`

## Your task
Summarize this pull request in 3–5 bullets. Highlight breaking changes.
```

When this skill runs:
1. Each `` !`<command>` `` executes immediately (before Claude sees anything)
2. Output replaces the placeholder
3. Claude receives the fully-rendered prompt with real live data

**Rules:**
- The `!` must appear at the **start of a line** or after whitespace (not after another character like `KEY=!cmd`)
- For multi-line commands, use a fenced `` ```! `` block:

````markdown
## Environment
```!
node --version
npm --version
git status --short
```
````

> **Tip:** Add `ultrathink` anywhere in the skill body to trigger deeper reasoning on that invocation.

---

## 7. Passing Arguments to Skills

```bash
/fix-issue 123
/migrate-component SearchBar React Vue
```

### Available substitutions

| Variable | Expands to |
|---|---|
| `$ARGUMENTS` | All arguments as a single string |
| `$ARGUMENTS[0]`, `$ARGUMENTS[1]` | Arguments by 0-based index |
| `$0`, `$1`, `$2` | Shorthand for `$ARGUMENTS[N]` |
| `$name` | Named argument (declared in `arguments` frontmatter) |
| `${CLAUDE_SESSION_ID}` | Current session ID |
| `${CLAUDE_EFFORT}` | Current effort level |
| `${CLAUDE_SKILL_DIR}` | Directory containing this `SKILL.md` |
| `${CLAUDE_PROJECT_DIR}` | Project root directory (v2.1.196+) |

### Example: fix a GitHub issue by number

```yaml
---
name: fix-issue
description: Fix a GitHub issue
disable-model-invocation: true
---

Fix GitHub issue $ARGUMENTS following our coding standards.

1. Read the issue description
2. Understand the requirements
3. Implement the fix
4. Write tests
5. Create a commit
```

Run: `/fix-issue 123` → Claude receives "Fix GitHub issue 123..."

### Example: named arguments

```yaml
---
name: migrate-component
description: Migrate a component from one framework to another
arguments: [component, from_framework, to_framework]
---

Migrate the $component component from $from_framework to $to_framework.
Preserve all existing behavior and tests.
```

Run: `/migrate-component SearchBar React Vue`

---

## 8. Supporting Files in Skills

Keep `SKILL.md` focused (under 500 lines). Move large reference material to separate files in the skill directory.

```
api-conventions/
├── SKILL.md           # Overview + navigation hints
├── reference.md       # Full API docs (loaded when needed)
├── examples.md        # Usage examples
└── scripts/
    └── lint-api.sh    # Validation script
```

In `SKILL.md`, tell Claude where to find the extra files:

```markdown
## Additional resources

- For complete API details, see [reference.md](reference.md)
- For usage examples, see [examples.md](examples.md)
```

Claude loads these files on demand — they don't consume context until needed.

---

## 9. Run Skills in a Subagent

Add `context: fork` to your frontmatter to run the skill in an isolated subagent. The skill content becomes the subagent's entire task prompt — it does NOT have access to your conversation history.

```yaml
---
name: deep-research
description: Research a topic thoroughly in an isolated context
context: fork
agent: Explore
---

Research $ARGUMENTS thoroughly:

1. Find relevant files using Glob and Grep
2. Read and analyze the code
3. Summarize findings with specific file references
```

When invoked:
1. A new isolated context is created
2. The subagent receives the skill content as its prompt
3. The `agent` field determines the execution environment (model, tools, permissions)
4. Results are summarized and returned to your main conversation

> **Warning:** `context: fork` only makes sense for skills with explicit task instructions. If your skill is just background knowledge ("use these API conventions"), the forked subagent gets guidelines but no actionable task and returns nothing useful.

### Skills ↔ Subagents relationship

| Approach | System prompt | Task | Also loads |
|---|---|---|---|
| Skill with `context: fork` | From agent type | SKILL.md content | CLAUDE.md (except for Explore/Plan agents) |
| Subagent with `skills` field | Subagent's markdown body | Claude's delegation message | Preloaded skills + CLAUDE.md |

---

## 10. Controlling Who Invokes a Skill

### Skill content lifecycle

Once invoked, `SKILL.md` content **stays in context for the rest of the session**. Write instructions as standing guidance, not one-time steps.

During auto-compaction, invoked skills are re-attached with up to 5,000 tokens each, sharing a combined budget of 25,000 tokens. Most-recently invoked skills have priority — older ones may be dropped.

### Restricting skill access from settings

**Disable all skills** (deny the Skill tool):
```
# In /permissions, add to deny rules:
Skill
```

**Allow only specific skills:**
```
Skill(commit)
Skill(review-pr *)
```

**Deny specific skills:**
```
Skill(deploy *)
```

### Override skill visibility with skillOverrides

Control visibility per-skill from `.claude/settings.local.json` without editing `SKILL.md`:

```json
{
  "skillOverrides": {
    "legacy-context": "name-only",
    "deploy": "off"
  }
}
```

| Value | Listed to Claude | In `/` menu |
|---|---|---|
| `"on"` (default) | Name and description | Yes |
| `"name-only"` | Name only | Yes |
| `"user-invocable-only"` | Hidden | Yes |
| `"off"` | Hidden | Hidden |

Use the `/skills` menu to toggle these interactively (highlight a skill + press Space to cycle states).

---

## 11. Tool Access in Skills

### Pre-approve tools

The `allowed-tools` field lets Claude use listed tools without asking permission while the skill is active:

```yaml
---
name: commit
description: Stage and commit the current changes
disable-model-invocation: true
allowed-tools: Bash(git add *) Bash(git commit *) Bash(git status *)
---
```

Pattern syntax: `ToolName(glob)` for scoped access, `ToolName` for full access to that tool.

### Block tools while a skill is active

Use `disallowed-tools` to remove tools from Claude's pool while the skill runs. The restriction clears after your next message:

```yaml
disallowed-tools: AskUserQuestion Write
```

---

## 12. Sharing Skills with Your Team

| Distribution method | How to do it |
|---|---|
| **Project skills** | Commit `.claude/skills/` to version control — teammates get the skills automatically |
| **Plugin** | Create a `skills/` directory in your plugin package; distribute via the plugin marketplace |
| **Managed** | Admins deploy to all users via organization managed settings |

### Example: project-level shared skill

```bash
# Create in project repo
mkdir -p .claude/skills/fix-issue
# Write SKILL.md
# Commit to git
git add .claude/skills/fix-issue/
git commit -m "Add fix-issue skill for team"
```

Everyone who clones or pulls the repo gets the skill automatically in Claude Code.

---

## 13. Evaluating and Iterating on Skills

Seeing a skill trigger is not the same as confirming it did what you intended. Measure two things separately:

1. **Does Claude invoke it** on the prompts it should? (Trigger precision)
2. **Is the output correct** when it does invoke it? (Output quality)

### The baseline comparison method

1. Collect a few realistic prompts that should trigger the skill
2. Run each in a **fresh session** with the skill available
3. Run each again with the skill [disabled via `skillOverrides`](#controlling-who-invokes-a-skill)
4. Compare results

Always test in a fresh session — leftover context from authoring masks gaps in the written instructions.

### skill-creator plugin (automated evals)

```
/plugin install skill-creator@claude-plugins-official
/reload-plugins
```

Then ask Claude: `evaluate my summarize-changes skill with skill-creator`

The plugin automates:

- **Test cases**: stores prompts, input files, and expected behavior in `evals/evals.json`
- **Isolated runs**: spawns a subagent per test case (clean context each time)
- **Grading**: checks assertions against output → `grading.json`
- **Benchmark**: pass rate, time, tokens comparison (with-skill vs without) → `benchmark.json`
- **Version comparison**: blind A/B between two skill versions
- **Description tuning**: generates should-trigger and should-not-trigger prompts, measures hit rate, proposes description edits

---

## 14. Troubleshooting Skills

### Skill not triggering

1. Check the `description` contains keywords users would naturally say
2. Verify it appears when asking Claude "What skills are available?"
3. Rephrase your request to match the description more closely
4. Invoke directly with `/skill-name` as a fallback
5. Run `--debug` to see YAML parse errors (malformed frontmatter means no `description` to match against)

### Skill triggers too often

1. Make the `description` more specific
2. Add `disable-model-invocation: true` for manual-only invocation

### Skill descriptions getting cut short

Claude Code budget-allocates skill descriptions at 1% of the model's context window. When it overflows, least-used skills lose their descriptions first.

**Diagnosis:** Run `/doctor` to see which skills are shortened or dropped.

**Fixes:**
- Set `"skillListingBudgetFraction": 0.02` in settings to raise the budget
- Mark low-priority skills as `"name-only"` in `skillOverrides`
- Trim verbose descriptions; put the key trigger phrase first (hard cap: 1,536 chars per skill)

---

## 15. Bundled Skills Reference

Claude Code ships with built-in skills available in every session:

| Skill | Purpose |
|---|---|
| `/code-review` | Review code changes for bugs, quality, style |
| `/debug` | Debug errors and test failures |
| `/batch` | Run multiple tasks in parallel |
| `/loop` | Run a command repeatedly on an interval |
| `/run` | Launch and test your app in a browser or terminal |
| `/verify` | Confirm a code change works in the running app |
| `/run-skill-generator` | Record how to build/launch your project for `/run` and `/verify` |
| `/claude-api` | Reference for the Claude API and Anthropic SDK |

Disable all bundled skills with `"disableBundledSkills": true` in settings. Override a bundled skill by creating a project skill with the same name.

---

# Part 2 — Subagents

## 16. What Are Subagents?

**Subagents** are specialized AI assistants that run in their own isolated context window. Claude delegates tasks to them; they work independently and return a summary.

> Use a subagent when a task would flood your main conversation with output you won't reference again — logs, search results, large file contents. The subagent does that work in its own context and returns only what you need.

### Key benefits

| Benefit | How it helps |
|---|---|
| **Preserve context** | Exploration and verbose work stay out of your main conversation |
| **Enforce constraints** | Limit which tools a subagent can use (e.g., read-only researcher) |
| **Reuse configurations** | Define once, use across all projects |
| **Specialize behavior** | Focused system prompts for specific domains |
| **Control costs** | Route cheap tasks to Haiku; complex work to Sonnet/Opus |

### What subagents are NOT

- Not persistent background processes (they run within a single session)
- Not the same as agent teams or background agents (those are separate features)
- Not a replacement for the main conversation when you need iterative back-and-forth

---

## 17. Built-in Subagents

Claude Code includes pre-built subagents that Claude uses automatically:

| Subagent | Model | Tools | Purpose |
|---|---|---|---|
| **Explore** | Haiku (fast) | Read-only (no Write/Edit) | File discovery, code search, codebase exploration |
| **Plan** | Inherits parent | Read-only | Research during plan mode |
| **General-purpose** | Inherits parent | All tools | Complex multi-step tasks requiring both exploration and action |
| **statusline-setup** | Sonnet | Read, Edit | Configures status line when you run `/statusline` |
| **claude-code-guide** | Haiku | Read, WebFetch, WebSearch | Answers questions about Claude Code features |

**Explore specifics:** Claude specifies thoroughness when invoking it:
- `quick` — targeted single lookup
- `medium` — balanced exploration
- `very thorough` — comprehensive analysis

**Key difference for Explore and Plan:** They skip `CLAUDE.md` files and git status to keep their context small and fast. All other subagents load both.

### Disabling built-in subagents

```json
{
  "permissions": {
    "deny": ["Agent(Explore)", "Agent(Plan)"]
  }
}
```

Or via CLI:
```bash
claude --disallowedTools "Agent(Explore)"
```

---

## 18. Creating Your First Subagent

### Method 1: The `/agents` command (recommended)

```
/agents
```

This opens a tabbed interface:
1. Switch to **Library** tab → **Create new agent** → **Personal** (saves to `~/.claude/agents/`)
2. Choose **Generate with Claude** and describe the subagent
3. Select tools (e.g., Read-only tools for a reviewer)
4. Select model
5. Pick a display color
6. Configure memory scope (optional)
7. Press `s` or `Enter` to save

The subagent is available immediately.

### Method 2: Write the file manually

Create a markdown file with YAML frontmatter:

```markdown
---
name: code-reviewer
description: Reviews code for quality and best practices. Use proactively after code changes.
tools: Read, Glob, Grep
model: sonnet
---

You are a code reviewer. When invoked:
1. Run `git diff` to see recent changes
2. Review modified files for quality, security, and best practices
3. Provide specific, actionable feedback organized by priority
```

Save to:
- Personal: `~/.claude/agents/code-reviewer.md`
- Project: `.claude/agents/code-reviewer.md`

**Note:** Files added directly to disk require a session restart to load. Files created via `/agents` take effect immediately.

### Method 3: CLI-defined (one-off or scripted)

```bash
claude --agents '{
  "code-reviewer": {
    "description": "Expert code reviewer. Use proactively after code changes.",
    "prompt": "You are a senior code reviewer. Focus on quality, security, and best practices.",
    "tools": ["Read", "Grep", "Glob", "Bash"],
    "model": "sonnet"
  }
}'
```

These exist only for the current session and aren't saved to disk — useful for automation scripts or quick experiments.

---

## 19. Subagent Scope and Storage Locations

| Location | Scope | Priority | Best for |
|---|---|---|---|
| Managed settings | Organization-wide | 1 (highest) | Admin-deployed shared subagents |
| `--agents` CLI flag | Current session only | 2 | One-off, automation |
| `.claude/agents/` | Current project | 3 | Project-specific, check into git |
| `~/.claude/agents/` | All your projects | 4 | Personal cross-project subagents |
| Plugin `agents/` dir | Where plugin is enabled | 5 (lowest) | Distributed subagents |

**Organization:** Claude Code scans agent directories recursively, so you can group them:

```
~/.claude/agents/
├── review/
│   ├── code-reviewer.md
│   └── security-reviewer.md
└── data/
    └── data-scientist.md
```

The subdirectory path does NOT affect the subagent name — identity comes from the `name` frontmatter field only.

---

## 20. Subagent Frontmatter Reference

```markdown
---
name: code-reviewer
description: Reviews code for quality. Use proactively after code changes.
tools: Read, Grep, Glob, Bash
disallowedTools: Write, Edit
model: sonnet
permissionMode: acceptEdits
maxTurns: 20
skills:
  - api-conventions
  - error-handling-patterns
mcpServers:
  - playwright:
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
memory: project
background: false
effort: medium
isolation: worktree
color: blue
---

Your system prompt here...
```

### Full field reference

| Field | Required | Description |
|---|---|---|
| `name` | Yes | Unique identifier (lowercase + hyphens). This is what hooks and mentions use |
| `description` | Yes | When Claude should delegate to this subagent. Write this like a trigger: "Use proactively after code changes" |
| `tools` | No | Allowlist of tools. If omitted, inherits all tools from the main conversation |
| `disallowedTools` | No | Denylist. Removes tools from inherited or specified list |
| `model` | No | `sonnet`, `opus`, `haiku`, `fable`, full model ID, or `inherit` (default) |
| `permissionMode` | No | `default`, `acceptEdits`, `auto`, `dontAsk`, `bypassPermissions`, or `plan` |
| `maxTurns` | No | Max agentic turns before the subagent stops automatically |
| `skills` | No | Skills to preload into the subagent's context at startup |
| `mcpServers` | No | MCP servers scoped to this subagent only |
| `hooks` | No | Lifecycle hooks (PreToolUse, PostToolUse, Stop) scoped to this subagent |
| `memory` | No | `user`, `project`, or `local` — enables persistent cross-session memory |
| `background` | No | `true` = always run as a background task |
| `effort` | No | Effort override: `low`, `medium`, `high`, `xhigh`, `max` |
| `isolation` | No | `worktree` = run in a separate git worktree (isolated file edits) |
| `color` | No | Display color: `red`, `blue`, `green`, `yellow`, `purple`, `orange`, `pink`, `cyan` |
| `initialPrompt` | No | Auto-submitted as the first user turn when used as main session via `--agent` |

### Permission modes

| Mode | Behavior |
|---|---|
| `default` | Standard permission checking with prompts |
| `acceptEdits` | Auto-accept file edits in the working directory |
| `auto` | Background classifier reviews commands automatically |
| `dontAsk` | Auto-deny prompts (explicitly allowed tools still work) |
| `bypassPermissions` | Skip all permission prompts — **use with caution** |
| `plan` | Read-only exploration (plan mode) |

> **Warning:** If the parent session uses `bypassPermissions` or `acceptEdits`, the subagent inherits it and cannot override it.

---

## 21. Choosing a Model per Subagent

```yaml
model: haiku     # Fast, cheap — use for simple lookups, search
model: sonnet    # Balanced — use for most coding tasks
model: opus      # Powerful — use for complex reasoning, architecture
model: inherit   # Same as main conversation (default)
```

### Model resolution order

1. `CLAUDE_CODE_SUBAGENT_MODEL` environment variable
2. Per-invocation model parameter (Claude can pass this when delegating)
3. Subagent definition's `model` frontmatter
4. Main conversation's model

**Cost tip:** Route the Explore subagent to Haiku for cheap, fast codebase searches. Reserve Sonnet/Opus for subagents that make decisions or write code.

---

## 22. Controlling Subagent Capabilities

### Tool allowlist

```yaml
---
name: safe-researcher
tools: Read, Grep, Glob, Bash
---
```

Only these four tools are available. No Edit, Write, or MCP tools.

### Tool denylist

```yaml
---
name: no-writes
disallowedTools: Write, Edit
---
```

Inherits everything from the main conversation except Write and Edit.

### MCP server patterns

```yaml
disallowedTools: mcp__github        # Remove all tools from 'github' MCP server
disallowedTools: mcp__*             # Remove all MCP tools from any server
tools: mcp__playwright              # Allow only tools from 'playwright' server
```

### Restrict which subagents can be spawned

For a subagent that runs as the main thread (`claude --agent`), control what sub-subagents it can spawn:

```yaml
---
name: coordinator
tools: Agent(worker, researcher), Read, Bash
---
```

Only `worker` and `researcher` subagent types can be spawned. To allow any subagent: `tools: Agent, Read, Bash`. To block spawning entirely: omit `Agent` from `tools`.

### Scope MCP servers to a subagent only

Keep heavy MCP tool descriptions out of your main conversation by scoping them to the subagent:

```yaml
---
name: browser-tester
mcpServers:
  - playwright:
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
---

Use Playwright tools to navigate, screenshot, and interact with pages.
```

The Playwright tools are only available inside this subagent. Your main conversation context stays clean.

---

## 23. Persistent Memory for Subagents

Give a subagent a persistent directory that survives across conversations:

```yaml
---
name: code-reviewer
description: Reviews code for quality and best practices
memory: project
---

You are a code reviewer. Update your agent memory with patterns, conventions,
and recurring issues you discover. This builds institutional knowledge across conversations.
```

### Memory scope options

| Scope | Location | Use when |
|---|---|---|
| `user` | `~/.claude/agent-memory/<name>/` | Knowledge applies across all projects |
| `project` | `.claude/agent-memory/<name>/` | Project-specific, shareable via git |
| `local` | `.claude/agent-memory-local/<name>/` | Project-specific, NOT committed to git |

**Recommendation:** `project` is the default good choice — shareable with the team via version control.

### How it works

When `memory` is set:
- The subagent's system prompt includes instructions for reading and writing to the memory directory
- The first 200 lines or 25KB of `MEMORY.md` is loaded at startup
- Read, Write, and Edit tools are automatically enabled for memory management

### Tips for effective memory

```markdown
# In the subagent's system prompt:
Update your agent memory as you discover codepaths, patterns, library locations,
and key architectural decisions. Write concise notes about what you found and where.
```

Ask the subagent to consult memory before starting: `"Review this PR, and check your memory for patterns you've seen before."`

Ask it to update memory after finishing: `"Now that you're done, save what you learned to your memory."`

---

## 24. Hooks in Subagents

Subagents can define hooks that run during their lifecycle. Two approaches:

### In the subagent's frontmatter (scoped to this subagent)

```yaml
---
name: code-reviewer
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-command.sh $TOOL_INPUT"
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "./scripts/run-linter.sh"
---
```

Common hook events for subagents:

| Event | When it fires |
|---|---|
| `PreToolUse` | Before the subagent uses a tool |
| `PostToolUse` | After the subagent uses a tool |
| `Stop` | When the subagent finishes (auto-converted to `SubagentStop`) |

### In project settings.json (for subagent lifecycle events)

```json
{
  "hooks": {
    "SubagentStart": [
      {
        "matcher": "db-agent",
        "hooks": [
          { "type": "command", "command": "./scripts/setup-db-connection.sh" }
        ]
      }
    ],
    "SubagentStop": [
      {
        "hooks": [
          { "type": "command", "command": "./scripts/cleanup-db-connection.sh" }
        ]
      }
    ]
  }
}
```

---

## 25. Preloading Skills into Subagents

Inject skill content into a subagent's context at startup using the `skills` field:

```yaml
---
name: api-developer
description: Implement API endpoints following team conventions
skills:
  - api-conventions
  - error-handling-patterns
---

Implement API endpoints. Follow the conventions and patterns from the preloaded skills.
```

**The difference from auto-discovery:**
- `skills` field → full skill content is injected at startup (ready to use immediately)
- Without `skills` → subagent can still discover and invoke skills during execution via the Skill tool

Note: Skills with `disable-model-invocation: true` cannot be preloaded.

---

## 26. Invoking Subagents Explicitly

Claude invokes subagents automatically based on descriptions. When you want to be explicit:

### Natural language (Claude decides whether to delegate)

```
Use the code-reviewer subagent to check my recent changes
Have the test-runner subagent fix failing tests
```

### @-mention (guarantees this subagent runs for this task)

```
@"code-reviewer (agent)" look at the auth changes
```

The `@` triggers a typeahead picker. Your full message still goes to Claude, which writes the delegation prompt.

### Run the whole session as a subagent

```bash
claude --agent code-reviewer
```

The subagent's system prompt replaces the default Claude Code system prompt. Useful for: CI/CD pipelines, specialized review sessions, cost control.

For plugin-scoped subagents:
```bash
claude --agent my-plugin:security-reviewer
```

Make it the default for a project:
```json
// .claude/settings.json
{
  "agent": "code-reviewer"
}
```

---

## 27. Foreground vs Background Subagents

| Mode | Behavior |
|---|---|
| **Foreground** | Blocks the main conversation until complete. Permission prompts come through immediately |
| **Background** | Runs concurrently while you keep working. Permission prompts surface in your main session with the subagent's name |

Claude decides foreground/background automatically. You can also:
- Ask Claude: "run this in the background"
- Press **Ctrl+B** to background a running task
- Set `background: true` in frontmatter to always run in background

---

## 28. Forking the Conversation

A **fork** is a special subagent that inherits your entire conversation history instead of starting fresh.

**When to use:**
- You want to delegate a side task without re-explaining the context
- You want to try several approaches in parallel from the same starting point

**How it differs from a regular subagent:**

| | Fork | Named subagent |
|---|---|---|
| Context | Full conversation history | Fresh context |
| System prompt | Same as main session | From definition file |
| Model | Same as main session | From `model` field |
| Prompt cache | Shared with main session | Separate cache |

Because a fork shares the parent's system prompt and tool definitions, **its first request reuses the parent's prompt cache** — making forks cheaper than fresh subagents for context-heavy tasks.

```bash
# Invoke with /fork
/fork draft unit tests for the parser changes so far
```

Enable fork mode:
```bash
export CLAUDE_CODE_FORK_SUBAGENT=1
```

### Fork controls in the UI

| Key | Action |
|---|---|
| `↑` / `↓` | Move between rows |
| `Enter` | Open fork transcript, send follow-up |
| `x` | Dismiss finished fork or stop running one |
| `Esc` | Return focus to prompt |

---

## 29. Nested Subagents

As of Claude Code v2.1.172, subagents can spawn their own subagents (up to 5 levels deep).

**Use case:** A reviewer subagent that dispatches a verifier per finding — intermediate output never reaches your main conversation, only the top-level summary comes back.

```
main conversation
  └── code-reviewer (depth 1)
        ├── verifier for finding #1 (depth 2)
        ├── verifier for finding #2 (depth 2)
        └── verifier for finding #3 (depth 2)
```

The subagent panel shows the full tree with `(+N)` descendant counts.

To prevent a subagent from spawning others, omit `Agent` from its `tools` list.

---

## 30. Common Subagent Patterns

### Pattern 1: Isolate high-volume operations

```
Use a subagent to run the full test suite and report only the failing tests with their error messages
```

The verbose test output stays in the subagent's context. You see only failures.

### Pattern 2: Parallel research

```
Research the authentication, database, and API modules in parallel using separate subagents
```

Each subagent explores its area independently; Claude synthesizes the findings.

### Pattern 3: Chain subagents

```
Use the code-reviewer subagent to find performance issues, then use the optimizer subagent to fix them
```

Each subagent completes its task, passes context forward, next one picks up where it left off.

### Pattern 4: Domain-specialist subagent

Assign expensive LLMs only to complex work, cheap ones to simple lookups:

```yaml
---
name: quick-search
description: Quick file and symbol lookups. Use for simple search queries.
tools: Read, Grep, Glob
model: haiku    # Fast + cheap
---
```

```yaml
---
name: architect
description: System design and architectural decisions. Use when planning large changes.
model: opus     # Most capable
---
```

---

## 31. When to Use Skills vs Subagents vs Main Conversation

| Situation | Best choice |
|---|---|
| You keep repeating the same instructions | **Skill** |
| You want a `/command` shortcut | **Skill** |
| Background knowledge that loads automatically | **Skill with `user-invocable: false`** |
| Task that produces lots of output you won't reference | **Subagent** |
| Research that would flood your context | **Subagent (Explore)** |
| Enforcing specific tool restrictions | **Subagent** |
| Parallel independent investigations | **Multiple subagents** |
| Iterative back-and-forth refinement | **Main conversation** |
| Quick targeted change | **Main conversation** |
| Side question, no tools needed | **`/btw` command** |
| Task needs full conversation history | **Fork** |

---

## 32. Ready-to-Use Subagent Examples

### Code Reviewer (read-only)

```markdown
---
name: code-reviewer
description: Expert code review specialist. Reviews code for quality, security, and maintainability. Use proactively after writing or modifying code.
tools: Read, Grep, Glob, Bash
model: inherit
color: blue
---

You are a senior code reviewer ensuring high standards of code quality and security.

When invoked:
1. Run `git diff` to see recent changes
2. Focus on modified files
3. Begin review immediately

Review checklist:
- Code is clear and readable
- Functions and variables are well-named
- No duplicated code
- Proper error handling
- No exposed secrets or API keys
- Input validation implemented
- Good test coverage
- Performance considerations addressed

Provide feedback organized by priority:
- Critical issues (must fix)
- Warnings (should fix)
- Suggestions (consider improving)

Include specific examples of how to fix issues.
```

### Debugger (reads and edits)

```markdown
---
name: debugger
description: Debugging specialist for errors, test failures, and unexpected behavior. Use proactively when encountering any issues.
tools: Read, Edit, Bash, Grep, Glob
color: red
---

You are an expert debugger specializing in root cause analysis.

When invoked:
1. Capture error message and stack trace
2. Identify reproduction steps
3. Isolate the failure location
4. Implement minimal fix
5. Verify solution works

For each issue provide:
- Root cause explanation
- Evidence supporting the diagnosis
- Specific code fix
- Testing approach
- Prevention recommendations

Focus on fixing the underlying issue, not the symptoms.
```

### Data Scientist

```markdown
---
name: data-scientist
description: Data analysis expert for SQL, BigQuery, and data insights. Use proactively for data analysis tasks and queries.
tools: Bash, Read, Write
model: sonnet
color: green
---

You are a data scientist specializing in SQL and BigQuery analysis.

When invoked:
1. Understand the data analysis requirement
2. Write efficient SQL queries
3. Use BigQuery CLI tools (bq) when appropriate
4. Analyze and summarize results

Key practices:
- Write optimized queries with proper filters and aggregations
- Include comments explaining complex logic
- Present findings clearly with actionable recommendations
- Ensure queries are cost-effective
```

### DB Reader with Hook Validation

```markdown
---
name: db-reader
description: Execute read-only database queries. Use when analyzing data or generating reports.
tools: Bash
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-readonly-query.sh"
---

You are a database analyst with read-only access. Execute SELECT queries to answer questions.

You cannot modify data. If asked to INSERT, UPDATE, DELETE, or modify schema, explain that you only have read access.
```

Validation script (`./scripts/validate-readonly-query.sh`):

```bash
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

if [ -z "$COMMAND" ]; then
  exit 0
fi

if echo "$COMMAND" | grep -iE '\b(INSERT|UPDATE|DELETE|DROP|CREATE|ALTER|TRUNCATE|REPLACE|MERGE)\b' > /dev/null; then
  echo "Blocked: Write operations not allowed. Use SELECT queries only." >&2
  exit 2
fi

exit 0
```

```bash
chmod +x ./scripts/validate-readonly-query.sh
```

---

## 33. Further Reading

| Resource | URL |
|---|---|
| Introduction to Agent Skills (course) | https://anthropic.skilljar.com/introduction-to-agent-skills |
| Introduction to Subagents (course) | https://anthropic.skilljar.com/introduction-to-subagents |
| Skills official docs | https://code.claude.com/docs/en/skills |
| Subagents official docs | https://code.claude.com/docs/en/sub-agents |
| Agent Skills open standard | https://agentskills.io |
| Evaluating skill quality | https://agentskills.io/skill-creation/evaluating-skills |
| Bundled skills & commands | https://code.claude.com/docs/en/commands |
| Hooks reference | https://code.claude.com/docs/en/hooks |
| Permissions reference | https://code.claude.com/docs/en/permissions |
| MCP integration | https://code.claude.com/docs/en/mcp |
| Plugins reference | https://code.claude.com/docs/en/plugins |

---

*Generated from Anthropic Academy courses and official Claude Code documentation — June 2026.*
