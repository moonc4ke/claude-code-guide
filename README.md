# Claude Code: Complete Guide

A practical reference for Claude Code customizations, configuration, and features (current as of v2.1.41).

## Quick Reference

| Type | Location | Invokable | Purpose |
|------|----------|-----------|---------|
| **Skills** | `.claude/skills/<name>/SKILL.md` | Yes (`/name`) | Discrete workflows (deploy, review) |
| **Agents** | `.claude/agents/<name>.md` | Yes (via Task tool) | Isolated workers with own context |
| **Features** | `.claude/features/<name>.md` | No | Project documentation for Claude |
| **Hooks** | Settings or frontmatter | Auto | Lifecycle automation & quality gates |
| **Plugins** | `.claude-plugin/plugin.json` | Mixed | Distributable skill/agent/hook packages |
| **MCP** | `.mcp.json` or `~/.claude.json` | Auto | External tool/database integrations |

---

## CLAUDE.md (Memory Files)

CLAUDE.md files provide persistent project context that Claude loads automatically every session.

### Locations

| File | Scope | Notes |
|------|-------|-------|
| `./CLAUDE.md` | Project root | Commit to git to share with team |
| `~/.claude/CLAUDE.md` | User-global | Personal preferences across all projects |
| `--add-dir` directories | Additional | Requires `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` |

### What to Include

```markdown
# Project Name

## Stack
- Framework: Next.js 14 with App Router
- Database: PostgreSQL with Prisma
- Auth: NextAuth.js

## Commands
- `npm run dev` - Start development server
- `npm test` - Run tests
- `npm run lint` - Check code style

## Code Style
- Use TypeScript strict mode
- Prefer named exports
- Use `async/await` over `.then()`

## Testing
- Tests live in `__tests__/` next to source files
- Use `vitest` for unit tests
- Run `npm test -- --watch` during development
```

### Best Practices

| Do | Don't |
|----|-------|
| Keep it concise (<500 lines) | Embed entire style guides |
| Include build/test commands | Duplicate linter rules |
| Document project structure | Add temporary notes |
| Update when patterns change | Include secrets/credentials |

**Tips:**
- Press `#` during a session to have Claude add learnings
- Iterate like you would a prompt - refine based on results

### Auto-Memory

Claude automatically records and recalls memories as it works. Memories persist across sessions in `~/.claude/projects/<project>/memory/`.

- Disable with `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`
- First 200 lines of `MEMORY.md` are injected into the system prompt

### Docs Index Pattern (Recommended)

[Vercel's research](https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals) found that a compressed documentation index in the memory file achieved 100% pass rate vs 53% for skills.

**Why this works:**
- No decision overhead - agent always has context
- Consistent availability every turn
- Avoids instruction sequencing issues

**Example: Compressed API Index**
```markdown
## API Reference
| Function | Location | Purpose |
|----------|----------|---------|
| createUser() | src/api/users.ts:23 | Create user record |
| validateJWT() | src/auth/jwt.ts:45 | Validate token |
| processPayment() | src/billing/stripe.ts:89 | Charge customer |

## Key Components
| Component | Path | Props |
|-----------|------|-------|
| Button | src/ui/Button.tsx | variant, size, disabled |
| Modal | src/ui/Modal.tsx | open, onClose, title |
```

**Guidelines:**
- Target <8KB total for docs index
- Use tables, not prose
- Include file:line references for navigation
- Link to details rather than embedding full content

---

## Skills

Skills are reusable instruction sets that Claude can invoke with `/skill-name`.

### When to Use Skills vs CLAUDE.md

Based on [Vercel's agent evaluations](https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals):

| Content Type | Best Location | Why |
|--------------|---------------|-----|
| API/framework reference | CLAUDE.md | 100% vs 53% pass rate with passive context |
| Design tokens & system | CLAUDE.md | Frequently needed, no invoke overhead |
| Build/test commands | CLAUDE.md | Universal, small footprint |
| **Code review workflow** | **Skill** | User-triggered, discrete task |
| **Deploy/release process** | **Skill** | Multi-step, needs explicit invocation |
| **Migration guides** | **Skill** | Version-specific, occasional use |

**Rule of thumb:**
- Reference material → CLAUDE.md (compressed)
- Discrete workflows → Skills

### Location
```
.claude/skills/<skill-name>/SKILL.md
```

Skills are auto-discovered from nested `.claude/skills/` directories and from `--add-dir` directories. Character budget scales with context window (2% of context).

### Frontmatter Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Display name for the skill |
| `description` | string | Brief description (shown in `/help`) |
| `disable-model-invocation` | boolean | If `true`, only manual `/invoke` works |
| `allowed-tools` | string[] | Tools the skill can use |
| `context` | object | Files/URLs to include as context |
| `context: fork` | | Run skill in a subagent context |

### Arguments

Skills support argument substitution:

```markdown
# In skill body
Process the file at $ARGUMENTS
# Or use indexed arguments
First arg: $0, second arg: $1
# Bracket syntax also works
First: $ARGUMENTS[0]
```

### Example: Code Review Skill

```markdown
<!-- .claude/skills/review/SKILL.md -->
---
name: Code Review
description: Review code changes for quality and best practices
allowed-tools:
  - Read
  - Glob
  - Grep
context:
  include:
    - CONTRIBUTING.md
    - .eslintrc.*
---

# Code Review Instructions

When reviewing code:

1. Check for security vulnerabilities
2. Verify error handling is present
3. Ensure tests cover new functionality
4. Look for performance issues

Use the project's coding standards from CONTRIBUTING.md.
```

### Invoking Skills
```bash
# Manual invocation
/review

# With arguments
/review src/auth/

# Plugin skills use namespaced format
/plugin-name:skill-name
```

Skills without extra permissions or hooks are auto-allowed without approval.

### Example: Design System in CLAUDE.md

Based on Vercel's findings, put design system reference in CLAUDE.md rather than a Skill:

```markdown
## Design System

### Tokens (from src/styles/tokens.css)
| Token | Value | Usage |
|-------|-------|-------|
| --color-primary | #2563eb | Buttons, links |
| --color-error | #dc2626 | Error states |
| --spacing-unit | 4px | Base grid unit |

### Typography
| Style | Font | Weight | Size |
|-------|------|--------|------|
| heading | Plus Jakarta Sans | 600 | 1.5-3rem |
| body | Inter | 400 | 1rem |

### Components (src/components/ui/)
| Component | Key Props |
|-----------|-----------|
| Button | variant: primary/secondary, size: sm/md/lg |
| Input | type, error, helperText |
| Card | padding: sm/md/lg, shadow: boolean |
```

This compressed format loads every session without invoke overhead.

---

## Agents

Agents are isolated workers spawned via the `Task` tool with their own context and tool access.

### Location
```
.claude/agents/<name>.md
```

Agents can also be defined inline via `--agents` CLI flag (JSON), managed with `/agents`, and distributed via plugins. Scoping priority: CLI flag > project > user > plugin.

### Frontmatter Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Agent identifier |
| `description` | string | What the agent does |
| `tools` | string[] | Available tools for this agent |
| `disallowedTools` | string[] | Tools to deny |
| `model` | string | Model to use (`sonnet`, `opus`, `haiku`) |
| `maxTurns` | number | Max agentic turns before stopping |
| `permissionMode` | string | `default`, `bypassPermissions`, `plan`, `delegate` |
| `skills` | string[] | Skills preloaded into agent context |
| `mcpServers` | string[] | MCP servers available to the agent |
| `hooks` | object | Lifecycle hooks scoped to this agent |
| `memory` | object | Persistent cross-session memory |

### Agent Memory

Agents can retain insights across sessions with the `memory` frontmatter field:

```yaml
memory:
  scope: project  # user | project | local
```

| Scope | Storage Path |
|-------|-------------|
| `user` | `~/.claude/agent-memory/<name>/` |
| `project` | `.claude/agent-memory/<name>/` |
| `local` | `.claude/agent-memory-local/<name>/` |

Auto-creates `MEMORY.md`, first 200 lines injected into system prompt. Automatically enables Read/Write/Edit tools.

### Example: Test Runner Agent

```markdown
<!-- .claude/agents/test-runner.md -->
---
name: test-runner
description: Runs tests and reports failures with suggested fixes
tools:
  - Bash
  - Read
  - Grep
model: haiku
maxTurns: 10
permissionMode: bypassPermissions
memory:
  scope: project
---

# Test Runner Agent

You are a test execution specialist.

## Responsibilities

1. Run the test suite using `npm test` or `pytest`
2. Parse test output for failures
3. Read failing test files to understand context
4. Suggest specific fixes for each failure

## Output Format

Return a summary with:
- Total tests run
- Pass/fail count
- For each failure: file, test name, error, suggested fix
```

### Spawning Agents

Agents are spawned automatically by Claude when appropriate, or can be referenced in skills. Use `Task(agent_type)` syntax to restrict which subagents can be spawned. Agents can be resumed with the `resume` parameter to continue from where they stopped.

- **Ctrl+B** - Background a running agent task
- Foreground vs background execution supported
- Auto-compaction via `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`

---

## Features

Features document project-specific functionality for Claude's reference. They are **not invokable** - they provide context about how parts of your system work.

### Location
```
.claude/features/<feature-name>.md
```

### Purpose
- Document complex systems (auth, payments, etc.)
- Explain architectural decisions
- Provide context for related tasks

### Example: Authentication Feature

```markdown
<!-- .claude/features/authentication.md -->
# Authentication System

## Overview
JWT-based authentication with refresh tokens.

## Key Files
- `src/auth/jwt.ts` - Token generation/validation
- `src/middleware/auth.ts` - Express middleware
- `src/routes/auth.ts` - Login/logout endpoints

## Flow
1. User submits credentials to `POST /auth/login`
2. Server validates and returns `{ accessToken, refreshToken }`
3. Client stores tokens and sends `Authorization: Bearer <token>`
4. Middleware validates token on protected routes

## Environment Variables
- `JWT_SECRET` - Signing key (required)
- `JWT_EXPIRES_IN` - Access token TTL (default: 15m)
- `REFRESH_EXPIRES_IN` - Refresh token TTL (default: 7d)

## Testing
```bash
npm test -- --grep "auth"
```
```

---

## Key Differences Summary

| Aspect | CLAUDE.md | Skills | Agents | Features |
|--------|-----------|--------|--------|----------|
| **Best for** | Reference material, docs index | Discrete workflows | Parallel/specialized work | System knowledge |
| **Pass rate** | 100% (Vercel evals) | 53-79% (Vercel evals) | N/A | N/A |
| **Invocation** | Automatic | `/skill-name` | Task tool (automatic) | Not invokable |
| **Context** | Always loaded | Loaded on invoke | Fresh per spawn | Reference material |

### When to Use Each

**Use CLAUDE.md when:**
- API/framework reference docs
- Design tokens and system rules
- Build/test commands
- Any frequently-needed context

**Use Skills when:**
- User-triggered discrete workflows (review, deploy)
- Multi-step processes needing explicit invocation
- Version-specific migrations or occasional tasks

**Use Agents when:**
- Work can be parallelized
- Task needs isolated context
- Specialized expertise is needed (e.g., security audit)

**Use Features when:**
- Documenting how a system works
- Providing context Claude needs for related tasks
- Recording architectural decisions

---

## Hooks

Hooks are user-defined automations that run at specific lifecycle points. They can be shell commands, LLM prompts, or agent evaluations.

### Hook Events

| Event | Trigger | Common Use |
|-------|---------|------------|
| `SessionStart` | Session begins | Setup env, load context |
| `SessionEnd` | Session ends | Cleanup, reporting |
| `UserPromptSubmit` | User sends message | Input validation, context injection |
| `PreToolUse` | Before tool execution | Permission checks, input modification |
| `PostToolUse` | After tool execution | Output validation, logging |
| `PostToolUseFailure` | Tool execution fails | Error handling, retry logic |
| `PermissionRequest` | Permission prompt shown | Auto-approve/deny patterns |
| `Notification` | Notification emitted | Alerts, integrations |
| `SubagentStart` | Subagent spawns | Resource tracking |
| `SubagentStop` | Subagent finishes | Result validation |
| `Stop` | Claude stops generating | Quality checks |
| `TeammateIdle` | Agent team member idle | Task reassignment |
| `TaskCompleted` | Task marked complete | Quality gates |
| `PreCompact` | Before context compaction | Save important context |
| `Setup` | `--init`/`--maintenance` run | Project bootstrapping |

### Hook Types

| Type | Description | Use Case |
|------|-------------|----------|
| `command` | Shell script execution | File ops, API calls, linting |
| `prompt` | Single LLM call | Content review, classification |
| `agent` | Multi-turn subagent | Complex validation, code review |

### Configuration

Hooks can be defined in multiple locations:

| Location | Scope |
|----------|-------|
| `~/.claude/settings.json` | User-global |
| `.claude/settings.json` | Project (shared) |
| `.claude/settings.local.json` | Project (local) |
| Skill/Agent frontmatter | Scoped to that skill/agent |
| Plugin hooks | Scoped to plugin |
| Managed policy | Organization-wide |

### Example: Lint on Edit

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "type": "command",
        "command": "eslint --fix $CLAUDE_FILE_PATH"
      }
    ]
  }
}
```

### Example: Auto-Approve Read-Only Tools

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read|Glob|Grep",
        "type": "command",
        "command": "echo '{\"permissionDecision\": \"allow\"}'"
      }
    ]
  }
}
```

### Hook Capabilities

- **`permissionDecision`** - Return `allow`, `deny`, or `ask` from PreToolUse hooks
- **`updatedInput`** - Modify tool input before execution
- **`additionalContext`** - Inject context into the conversation
- **`async: true`** - Run hooks in the background
- **Matchers** - Regex patterns (e.g., `Bash`, `Edit|Write`, `mcp__memory__.*`)

### Managing Hooks

Use `/hooks` in Claude Code for interactive hook management.

**Environment variables available in hooks:**
- `$CLAUDE_PROJECT_DIR` - Project root directory
- `$CLAUDE_PLUGIN_ROOT` - Plugin root (for plugin hooks)
- `$CLAUDE_ENV_FILE` - Env file path (SessionStart only, for persisting env vars)

---

## Plugins

Plugins are distributable packages containing skills, agents, hooks, MCP servers, and LSP servers.

### Structure

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json          # Plugin manifest
├── commands/                 # Slash commands
├── agents/                   # Agent definitions
├── skills/                   # Skills (namespaced as /plugin:skill)
├── hooks/                    # Hook definitions
├── .mcp.json                 # MCP servers
└── .lsp.json                 # LSP servers
```

### Installing Plugins

Plugins can be distributed via:
- GitHub repos
- Git URLs (with SHA pinning)
- NPM packages
- URLs
- Local directories (for development: `--plugin-dir`)

### Configuration

```json
{
  "enabledPlugins": ["github-url-or-name"],
  "extraKnownMarketplaces": ["https://marketplace.example.com"],
  "strictKnownMarketplaces": true
}
```

### Plugin Skills

Plugin skills are namespaced to avoid conflicts:

```bash
/my-plugin:deploy       # Invoke plugin's deploy skill
/my-plugin:review       # Invoke plugin's review skill
```

### Managing Plugins

- VSCode has native plugin management support
- Search installed plugins list
- Pin plugins to specific git commit SHAs for reproducibility
- Install counts displayed for marketplace plugins
- Trust warnings shown for new plugins

---

## Subagents (Multi-Agent via Task Tool)

Claude automatically spawns **subagents** for complex tasks. You don't invoke them directly - Claude decides when to delegate.

### Built-in Subagent Types

| Type | Purpose | Tools Available |
|------|---------|-----------------|
| `Explore` | Fast codebase exploration | Read-only (Glob, Grep, Read, WebFetch, WebSearch) |
| `Plan` | Architecture design | Read-only + planning |
| `Bash` | Command execution | Bash only |
| `general-purpose` | Full capabilities | All tools |
| `statusline-setup` | Configure status line | Read, Edit |
| `claude-code-guide` | Claude Code help | Glob, Grep, Read, WebFetch, WebSearch |

### How to Use

```
# Ask Claude to use a specific agent
"Use the Explore agent to find all authentication-related files"

# Or just describe the task - Claude auto-delegates
"Search the codebase for how payments are processed"
```

### In Skills (agent field)

```yaml
---
name: Security Audit
agent: Explore  # Skill runs via Explore subagent
---
```

### Subagent Features

- **Resume** - Continue a subagent from where it stopped using agent ID
- **Background execution** - Run agents in background, check results later
- **Ctrl+B** - Background a currently running task
- **Token metrics** - Token count, tool uses, and duration reported in results
- **MCP access** - Subagents can access MCP tools
- **Auto-compaction** - `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` for long-running agents

---

## Agent Teams (Experimental)

Multi-agent collaboration where multiple Claude Code instances work together with shared task lists and inter-agent messaging.

### Enable

```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

Or set in `settings.json`.

### How It Works

- One instance acts as **team lead**, others as **teammates**
- Shared task list with dependency tracking
- Direct teammate-to-teammate messaging
- Broadcast messages to all teammates
- Plan approval flow for teammates

### Display Modes

| Mode | Setting | Description |
|------|---------|-------------|
| `in-process` | Default | All in one terminal, Shift+Up/Down to navigate |
| `tmux` | `teammateMode: "tmux"` | Split panes in tmux |
| iTerm2 | Automatic | Split panes in iTerm2 |

### Configuration

```bash
# CLI flag
claude --teammate-mode tmux

# Or in settings
{ "teammateMode": "auto" }  # auto | in-process | tmux
```

### Hook Events for Teams

- **`TeammateIdle`** - Triggered when a teammate has no work; use for task reassignment
- **`TaskCompleted`** - Triggered when a task is marked complete; use for quality gates

### Delegate Mode

Set `permissionMode: "delegate"` on the team lead agent for coordination-only behavior (lead doesn't write code directly).

---

## Task Management

Built-in task tracking with dependency management, shared across agents and sessions.

### Commands

| Action | How |
|--------|-----|
| Toggle task list | **Ctrl+T** |
| Open task dialog | `/tasks` |
| Share across sessions | Set `CLAUDE_CODE_TASK_LIST_ID` env var |
| Disable | Set `CLAUDE_CODE_ENABLE_TASKS=false` |

### Features

- Create tasks with subject, description, status
- Status workflow: `pending` → `in_progress` → `completed`
- Dependency blocking (`blocks` / `blockedBy`)
- Task deletion
- Shared task lists for agent team coordination
- Task assignment to specific agents

---

## Checkpointing & Rewind

Claude Code automatically tracks file state at every user prompt, allowing you to restore previous states.

### Access

- **Esc + Esc** or `/rewind` - Open checkpoint selector

### Actions

| Action | Description |
|--------|-------------|
| Restore code + conversation | Roll back both files and chat history |
| Restore conversation only | Reset chat but keep file changes |
| Restore code only | Revert files but keep chat history |
| Summarize from here | Compact conversation from a specific point |

### Notes

- Checkpoints persist across sessions
- Bash command file changes are not tracked (only Edit/Write operations)
- External file changes (outside Claude Code) are not tracked

---

## Fast Mode

Faster output from the same Opus 4.6 model at higher token cost.

### Usage

```bash
/fast          # Toggle fast mode on/off
```

### Details

- Same model (Opus 4.6), different API configuration for faster output
- Pricing: $30/$150 MTok (input/output) under 200K context
- Auto-fallback to standard mode on rate limits
- Persists across sessions
- Visual indicator: lightning bolt icon next to prompt
- `fastMode` setting in `settings.json`

---

## Settings & Configuration

### Scope Hierarchy (highest priority first)

| Scope | Location | Purpose |
|-------|----------|---------|
| Managed | `/etc/claude-code/` (Linux), `/Library/Application Support/ClaudeCode/` (macOS) | Organization policy |
| CLI args | Command line flags | Session override |
| Local | `.claude/settings.local.json` | Personal project settings (gitignored) |
| Project | `.claude/settings.json` | Shared team settings |
| User | `~/.claude/settings.json` | Personal global settings |

### Key Settings

| Setting | Type | Description |
|---------|------|-------------|
| `model` | string | Override default model |
| `fastMode` | boolean | Enable fast mode |
| `language` | string | Preferred response language |
| `outputStyle` | string | Adjust system prompt style (e.g., "Explanatory") |
| `statusLine` | object | Custom status line display |
| `showTurnDuration` | boolean | Show/hide turn duration |
| `spinnerVerbs` | string[] | Customize spinner text |
| `prefersReducedMotion` | boolean | Reduce animations |
| `plansDirectory` | string | Custom plan file storage |
| `cleanupPeriodDays` | number | Session cleanup period |
| `autoUpdatesChannel` | string | `stable` or `latest` |
| `alwaysThinkingEnabled` | boolean | Always use extended thinking |
| `temperatureOverride` | number | Override temperature |
| `fileSuggestion` | string | Custom file suggestion command |
| `attribution` | object | Customize commit/PR attribution |

### Managing Settings

```bash
/config          # Interactive settings editor (supports search)
/doctor          # Diagnostics (shows update channel, warnings)
/debug           # Troubleshoot current session
```

Config backups are timestamped and rotated (5 most recent kept).

---

## Permissions

### Permission Modes

| Mode | Description |
|------|-------------|
| `default` | Ask for permission on each action |
| `acceptEdits` | Auto-accept file edits, ask for bash |
| `reviewEdits` | Show diffs for review |
| `bypassPermissions` | Skip all permission prompts |

Set with `defaultMode` in settings. Disable bypass with `disableBypassPermissionsMode`.

### Permission Rules

Define patterns to auto-allow specific tools:

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(npm run *)",
      "Bash(git *)",
      "Edit(src/**)",
      "WebFetch(domain:example.com)",
      "MCP(**)"
    ],
    "deny": [
      "Read(./.env)"
    ]
  }
}
```

- `Bash(npm *)` - Wildcard matching for commands
- `Bash(*)` is treated as equivalent to `Bash`
- `additionalDirectories` for extending allowed paths
- Unreachable rules generate warnings

### Shift+Tab

Cycle through permission modes during a session. In agent teams, also toggles delegate mode.

---

## Sandbox

OS-level sandboxing for bash commands to restrict file and network access.

### Configuration

**Note:** Writes to `.claude/skills` are blocked in sandbox mode for security.

```json
{
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "excludedCommands": ["docker"],
    "network": {
      "allowedDomains": ["api.example.com"],
      "allowLocalBinding": true,
      "httpProxyPort": 8080,
      "socksProxyPort": 1080,
      "allowUnixSockets": ["/var/run/docker.sock"],
      "allowAllUnixSockets": false
    }
  }
}
```

### Commands

```bash
/sandbox         # View sandbox status and dependency info
```

---

## MCP (Model Context Protocol)

MCP connects Claude Code to external tools, databases, and APIs.

### What You Can Do

- Query databases directly
- Integrate with GitHub, Jira, Sentry, etc.
- Access file systems, APIs, and services
- Automate workflows across tools

### Adding MCP Servers

```bash
# HTTP server (recommended for cloud services)
claude mcp add --transport http <name> <url>

# With pre-configured OAuth credentials
claude mcp add --transport http <name> <url> --client-id <id> --client-secret <secret>

# SSE server (deprecated, use HTTP)
claude mcp add --transport sse <name> <url>

# Local stdio server
claude mcp add --transport stdio <name> -- <command> [args...]
```

### Common Examples

```bash
# GitHub integration
claude mcp add --transport http github https://api.githubcopilot.com/mcp/

# Sentry error monitoring
claude mcp add --transport http sentry https://mcp.sentry.dev/mcp

# PostgreSQL database
claude mcp add --transport stdio db -- npx -y @bytebase/dbhub \
  --dsn "postgresql://user:pass@host:5432/dbname"

# Filesystem access
claude mcp add --transport stdio files -- npx -y @anthropic/mcp-server-filesystem /path/to/dir
```

### Configuration Scopes

| Scope | Flag | Storage | Use Case |
|-------|------|---------|----------|
| `local` | (default) | `~/.claude.json` | Personal, project-specific |
| `project` | `--scope project` | `.mcp.json` | Shared with team (commit to git) |
| `user` | `--scope user` | `~/.claude.json` | Personal, all projects |

### Managing Servers

```bash
claude mcp list              # List all servers
claude mcp get <name>        # Get server details
claude mcp remove <name>     # Remove a server
/mcp                         # In Claude Code: status & auth
/mcp enable <name>           # Enable a specific server
```

### MCP Tool Search

When MCP tool descriptions exceed 10% of context, tools are deferred to an `MCPSearch` meta-tool for efficiency.

- Enabled by default (`auto` mode)
- Configure threshold: `auto:N` syntax (where N is percentage)
- Set via `ENABLE_TOOL_SEARCH` env var

### Project Config (`.mcp.json`)

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/"
    },
    "db": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@bytebase/dbhub", "--dsn", "${DATABASE_URL}"],
      "env": {}
    }
  }
}
```

### Environment Variables in Config

Use `${VAR}` or `${VAR:-default}` in `.mcp.json`:

```json
{
  "mcpServers": {
    "api": {
      "type": "http",
      "url": "${API_URL:-https://api.example.com}/mcp",
      "headers": {
        "Authorization": "Bearer ${API_KEY}"
      }
    }
  }
}
```

### MCP Settings

| Setting | Description |
|---------|-------------|
| `allowedMcpServers` | Allowlist of MCP servers |
| `deniedMcpServers` | Denylist of MCP servers |
| `enableAllProjectMcpServers` | Auto-enable all project MCP servers |

### MCP Environment Variables

| Variable | Description |
|----------|-------------|
| `MAX_MCP_OUTPUT_TOKENS` | Max output tokens for MCP responses |
| `MCP_TIMEOUT` | Connection timeout |
| `MCP_TOOL_TIMEOUT` | Tool execution timeout |
| `MCP_CLIENT_SECRET` | OAuth client secret |
| `MCP_OAUTH_CALLBACK_PORT` | OAuth callback port |

### Using MCP Resources

Reference MCP resources with `@` mentions:

```
> Analyze @github:issue://123
> Review @postgres:schema://users
```

### Authentication

Many servers require OAuth. Use `/mcp` in Claude Code to authenticate:

```
> /mcp
# Select server → Authenticate → Follow browser flow
```

---

## Auto-Claude (Autonomous Workflows)

[Auto-Claude](https://github.com/AndyMik90/Auto-Claude) is an open-source autonomous coding framework that plans, builds, and validates software with minimal human intervention.

### Requirements

- Claude Pro or Max subscription
- Claude Code CLI (`npm install -g @anthropic-ai/claude-code`)
- Initialized git repository

### Installation

Download the desktop app from [GitHub Releases](https://github.com/AndyMik90/Auto-Claude/releases):
- **Windows**: x64 executable
- **macOS**: Apple Silicon and Intel builds
- **Linux**: AppImage, Debian package, or Flatpak

### Quick Start

1. Download and install the application
2. Select your git repository folder
3. Complete OAuth setup with Claude
4. Create a task describing your goal
5. Monitor autonomous agent execution

### Key Features

| Feature | Description |
|---------|-------------|
| **Parallel Agents** | Up to 12 agent terminals running simultaneously |
| **Git Worktrees** | Isolates changes from your main branch |
| **QA Loop** | Built-in validation before code review |
| **Integrations** | GitHub/GitLab issues, Linear sync, auto merge conflict resolution |
| **Memory Layer** | Agents retain insights across sessions |

### Desktop Interface

Three main views:
- **Kanban Board** – Visual task management from planning to completion
- **Agent Terminals** – AI-powered terminals with context injection
- **Roadmap** – Feature planning with competitive analysis

### CLI Usage

For terminal-only or CI/CD workflows:

```bash
cd apps/backend

# Create specs interactively
python spec_runner.py --interactive

# Execute autonomous build
python run.py --spec 001

# Review changes
python run.py --spec 001 --review

# Merge to main branch
python run.py --spec 001 --merge
```

### Security

Three-layer security model:
1. OS sandboxing for bash commands
2. Filesystem restrictions to project directories
3. Dynamic command allowlisting based on project stack

### License

AGPL-3.0 (free for open-source). Commercial licensing available for closed-source projects.

---

## CLI Commands & Flags

### In-Session Commands

| Command | Description |
|---------|-------------|
| `/help` | Show available commands |
| `/config` | Settings editor (with search) |
| `/doctor` | Diagnostics and update channel info |
| `/debug` | Troubleshoot current session |
| `/mcp` | MCP server status and auth |
| `/mcp enable <name>` | Enable a specific MCP server |
| `/hooks` | Interactive hooks manager |
| `/agents` | Interactive agent manager |
| `/tasks` or **Ctrl+T** | Task list management |
| `/fast` | Toggle fast mode |
| `/compact` | Manually compact context |
| `/rewind` | Checkpoint selector (also Esc+Esc) |
| `/keybindings` | Customize keyboard shortcuts |
| `/sandbox` | Sandbox status and config |
| `/stats` | Usage statistics (with date filtering via `r` key) |
| `/context` | Token count display |
| `/plan` | Enter plan mode |
| `/rename` | Rename current session (auto-generates name from context if no argument) |
| `/tag` | Tag current session |
| `/teleport` | Connect to claude.ai |
| `/remote-env` | Remote environment config |

### Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| **Ctrl+T** | Toggle task list |
| **Ctrl+B** | Background running task/agent |
| **Ctrl+G** | Open external editor |
| **Esc+Esc** | Open rewind/checkpoint selector |
| **Shift+Tab** | Cycle permission modes |
| **Shift+Up/Down** | Navigate agent team teammates |
| **Tab** | Autocomplete (bash history, files) |
| `/keybindings` | Customize all shortcuts |

### Auth Subcommands

```bash
claude auth login          # Log in to Claude
claude auth status         # Check current authentication status
claude auth logout         # Log out
```

### CLI Flags

| Flag | Description |
|------|-------------|
| `--from-pr <num/url>` | Resume session linked to a PR |
| `--add-dir <path>` | Add additional working directories |
| `--agent <name>` | Start with a specific agent |
| `--agents <json>` | Inline JSON agent definitions |
| `--plugin-dir <path>` | Load plugins from directory |
| `--teammate-mode <mode>` | Agent team display mode |
| `--init` | Trigger Setup hooks |
| `--init-only` | Run Setup hooks and exit |
| `--maintenance` | Run maintenance Setup hooks |
| `--tools <list>` | Restrict available tools |

---

## Environment Variables

### Core

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | Enable agent teams (`1`) |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | Disable auto memory (`1`) |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | Disable background tasks |
| `CLAUDE_CODE_ENABLE_TASKS` | Enable/disable task system (`false` to disable) |
| `CLAUDE_CODE_TASK_LIST_ID` | Share task list across sessions |
| `CLAUDE_CODE_TMPDIR` | Override temp directory |
| `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD` | Load CLAUDE.md from `--add-dir` dirs (`1`) |

### Model & Output

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_SUBAGENT_MODEL` | Override subagent model |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | Max output tokens (default 32K, max 64K) |
| `CLAUDE_CODE_EFFORT_LEVEL` | Effort level: `low` / `medium` / `high` |
| `MAX_THINKING_TOKENS` | Extended thinking token budget |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | Subagent auto-compaction threshold |

### Bash & Shell

| Variable | Description |
|----------|-------------|
| `BASH_DEFAULT_TIMEOUT_MS` | Default bash command timeout |
| `BASH_MAX_OUTPUT_LENGTH` | Max bash output characters |
| `CLAUDE_CODE_SHELL` | Override shell detection |
| `CLAUDE_CODE_SHELL_PREFIX` | Command prefix for all bash |

### MCP & Network

| Variable | Description |
|----------|-------------|
| `ENABLE_TOOL_SEARCH` | MCP tool search mode |
| `MAX_MCP_OUTPUT_TOKENS` | Max MCP output tokens |
| `MCP_TIMEOUT` | MCP connection timeout |
| `MCP_TOOL_TIMEOUT` | MCP tool execution timeout |
| `MCP_CLIENT_SECRET` | MCP OAuth client secret |
| `MCP_OAUTH_CALLBACK_PORT` | MCP OAuth callback port |
| `CLAUDE_CODE_CLIENT_CERT` | mTLS client certificate |
| `CLAUDE_CODE_CLIENT_CERT_KEY` | mTLS client key |
| `CLAUDE_CODE_CLIENT_CERT_KEY_PASSPHRASE` | mTLS key passphrase |

### Teams & Agents

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_TEAM_NAME` | Agent team name |

### Miscellaneous

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_PLAN_MODE_REQUIRED` | Require plan approval (read-only mode) |
| `CLAUDE_CODE_DISABLE_TERMINAL_TITLE` | Disable terminal title updates |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | Disable non-essential traffic |
| `FORCE_AUTOUPDATE_PLUGINS` | Force plugin auto-updates |

---

## References & Changelog

### Documentation

| Resource | URL |
|----------|-----|
| **Official Docs** | https://code.claude.com/docs |
| **Changelog** | https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md |
| **Hooks Guide** | https://code.claude.com/docs/en/hooks-guide |
| **Hooks Reference** | https://code.claude.com/docs/en/hooks |
| **Plugins** | https://code.claude.com/docs/en/plugins |
| **Agent Teams** | https://code.claude.com/docs/en/agent-teams |
| **MCP Docs** | https://code.claude.com/docs/en/mcp |
| **Settings** | https://code.claude.com/docs/en/settings |
| **Permissions** | https://code.claude.com/docs/en/permissions |
| **Sandboxing** | https://code.claude.com/docs/en/sandboxing |
| **Memory** | https://code.claude.com/docs/en/memory |
| **Fast Mode** | https://code.claude.com/docs/en/fast-mode |
| **Checkpointing** | https://code.claude.com/docs/en/checkpointing |
| **Best Practices** | https://code.claude.com/docs/en/best-practices |
| **GitHub Actions** | https://code.claude.com/docs/en/github-actions |
| **GitLab CI/CD** | https://code.claude.com/docs/en/gitlab-ci-cd |
| **Headless/SDK** | https://code.claude.com/docs/en/headless |
| **MCP Registry** | https://github.com/modelcontextprotocol/servers |
| **GitHub Issues** | https://github.com/anthropics/claude-code/issues |

### Platforms

| Platform | URL |
|----------|-----|
| **Desktop** | https://code.claude.com/docs/en/desktop |
| **Chrome Extension** | https://code.claude.com/docs/en/chrome |
| **Slack Integration** | https://code.claude.com/docs/en/slack |
| **Claude Code on the Web** | https://code.claude.com/docs/en/claude-code-on-the-web |
| **Dev Containers** | https://code.claude.com/docs/en/devcontainer |

### Recent Features (v2.1.x)

| Version | Feature |
|---------|---------|
| v2.1.41 | **`claude auth` subcommands** (login/status/logout), Windows ARM64, `/rename` auto-generates names |
| v2.1.39 | Guard against nested Claude Code sessions, OTel `speed` attribute for fast mode |
| v2.1.38 | Security: blocked `.claude/skills` writes in sandbox, improved heredoc parsing |
| v2.1.37 | `/fast` + `/extra-usage` compatibility fix |
| v2.1.36 | **Fast mode** for Opus 4.6 |
| v2.1.33 | **Agent memory** (persistent cross-session), `TeammateIdle`/`TaskCompleted` hooks |
| v2.1.32 | **Claude Opus 4.6**, **agent teams** (research preview), **auto-memory**, "Summarize from here" |
| v2.1.30 | **PDF page-range reading**, `/debug` command, pre-configured MCP OAuth, task result metrics |
| v2.1.27 | `--from-pr` flag, auto PR-session linking |
| v2.1.20 | PR review status indicator, CLAUDE.md from `--add-dir`, `/sandbox` command |
| v2.1.19 | `$0`/`$1` argument shorthand for skills, `CLAUDE_CODE_ENABLE_TASKS` |
| v2.1.18 | **Customizable keybindings** (`/keybindings`) |
| v2.1.16 | **Task management system**, native plugin management (VSCode) |
| v2.1.14 | History autocomplete, plugin search and SHA pinning |
| v2.1.10 | `Setup` hook event, plugin install counts |
| v2.1.9 | MCP `auto:N` threshold, `plansDirectory` setting, external editor in questions |
| v2.1.6 | `/config` search, `/stats` date filtering, nested skill discovery |
| v2.1.3 | Skills merged with slash commands, auto-update channel toggle |
| v2.1.2 | Clickable file paths, Windows Package Manager support |
| v2.1.0 | Skill hot-reload, `context: fork`, vim motions, wildcard bash permissions, `/teleport` |

Check the [changelog](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) regularly for new features.
