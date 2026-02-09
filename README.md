# Claude Code: Skills, Agents & Features Guide

A practical reference for creating and using Claude Code customizations.

## Quick Reference

| Type | Location | Invokable | Purpose |
|------|----------|-----------|---------|
| **Skills** | `.claude/skills/<name>/SKILL.md` | Yes (`/name`) | Discrete workflows (deploy, review) |
| **Agents** | `.claude/agents/<name>.md` | Yes (via Task tool) | Isolated workers with own context |
| **Features** | `.claude/features/<name>.md` | No | Project documentation for Claude |
| **MCP** | `.mcp.json` or `~/.claude.json` | Auto | External tool/database integrations |

---

## CLAUDE.md (Memory Files)

CLAUDE.md files provide persistent project context that Claude loads automatically every session.

### Location

```
./CLAUDE.md
```

Commit to git to share with your team.

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

### Frontmatter Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Display name for the skill |
| `description` | string | Brief description (shown in `/help`) |
| `disable-model-invocation` | boolean | If `true`, only manual `/invoke` works |
| `allowed-tools` | string[] | Tools the skill can use |
| `context` | object | Files/URLs to include as context |

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
```

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

### Frontmatter Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Agent identifier |
| `description` | string | What the agent does |
| `tools` | string[] | Available tools for this agent |
| `model` | string | Model to use (`sonnet`, `opus`, `haiku`) |
| `permissionMode` | string | `default`, `bypassPermissions`, `plan` |
| `skills` | string[] | Skills available to the agent |
| `hooks` | object | Lifecycle hooks (preToolCall, postToolCall) |

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
permissionMode: bypassPermissions
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

Agents are spawned automatically by Claude when appropriate, or can be referenced in skills.

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

### Example: Payment Feature

```markdown
<!-- .claude/features/payments.md -->
# Payment Processing

## Provider
Stripe API v2023-10-16

## Key Files
- `src/services/stripe.ts` - Stripe client wrapper
- `src/webhooks/stripe.ts` - Webhook handlers
- `src/models/subscription.ts` - Subscription model

## Webhook Events Handled
- `checkout.session.completed` - New subscription
- `invoice.paid` - Renewal success
- `customer.subscription.deleted` - Cancellation

## Testing Webhooks Locally
```bash
stripe listen --forward-to localhost:3000/webhooks/stripe
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

## Subagents (Multi-Agent via Task Tool)

Claude automatically spawns **subagents** for complex tasks. You don't invoke them directly - Claude decides when to delegate.

### Built-in Subagent Types

| Type | Purpose | Tools Available |
|------|---------|-----------------|
| `Explore` | Fast codebase exploration | Read-only (Glob, Grep, Read) |
| `Plan` | Architecture design | Read-only + planning |
| `Bash` | Command execution | Bash only |
| `general-purpose` | Full capabilities | All tools |

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

### Swarm / Teams (Advanced)

Internal parameters (`team_name`, `name` in Task tool) suggest team orchestration exists, but this is **not publicly documented** as a user-invokable feature. The standard approach is using subagents as described above.

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
```

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

### Environment Variables

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

## References & Changelog

| Resource | URL |
|----------|-----|
| **Changelog** | https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md |
| **Official Docs** | https://docs.anthropic.com/en/docs/claude-code |
| **MCP Docs** | https://docs.anthropic.com/en/docs/claude-code/mcp |
| **Best Practices** | https://docs.anthropic.com/en/docs/claude-code/best-practices |
| **MCP Registry** | https://github.com/modelcontextprotocol/servers |
| **GitHub Issues** | https://github.com/anthropics/claude-code/issues |

### Recent Features (v2.1.x)

- **Keyboard Shortcuts** - `/keybindings` to customize
- **Session Forking/Rewind** - Branch conversations
- **History Autocomplete** - Tab to complete bash from history
- **Skill Hot-Reload** - Skills reload immediately on save
- **Vim Motions Extended** - `;`, `,`, `y`, `p`, text objects
- **Wildcard Bash Permissions** - `Bash(npm *)` patterns

Check the changelog regularly for new features.
