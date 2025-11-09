# Beads: A Memory Upgrade for AI Coding Agents

> **TL;DR:** Git-backed issue tracker that gives AI agents (like Cursor, Claude) persistent memory across sessions. Tracks dependencies, auto-discovers work, prevents context loss. Works with Linear/Jira as complementary tool for implementation details.

**Quick Links:**
- 📦 [Installation](#installation--quick-start) | 🎯 [Cursor Integration](#can-beads-be-used-in-cursor) | 🤝 [Linear/Jira Comparison](#does-beads-replace-human-issue-trackers-like-linear)
- 💡 [Example Use Case](#real-world-example-use-case) | 🔧 [How It Works](#core-architecture) | 📖 [Full Docs](https://github.com/steveyegge/beads)

## Overview

**Beads** (GitHub: [steveyegge/beads](https://github.com/steveyegge/beads)) is a revolutionary graph-based issue tracking system designed specifically for AI coding agents. With over 2,500 stars, it solves one of the most critical challenges in AI-assisted development: **agent amnesia** when dealing with complex, long-horizon tasks.

### The Problem It Solves

AI coding agents like Claude, GPT-4, or other LLM-based assistants suffer from:
- **Context window limitations** - They forget tasks across sessions
- **Lost work** - Problems discovered during one session are forgotten in the next
- **Markdown chaos** - Traditional TODO lists in markdown files become "swamps of rotten half-implemented plans"
- **No dependency tracking** - Agents can't understand what blocks what
- **Merge conflicts** - Multiple agents or branches creating tasks simultaneously lead to ID collisions

### What Beads Does

Beads is a lightweight, git-backed issue tracker that gives AI agents:
- ✅ **Persistent memory** across sessions
- ✅ **Dependency management** with 4 types of relationships
- ✅ **Automatic issue filing** for discovered work
- ✅ **"Ready work" detection** - finds tasks with no blockers
- ✅ **Distributed by design** - works across multiple machines via git
- ✅ **Collision-resistant IDs** - hash-based IDs prevent multi-agent conflicts
- ✅ **Full audit trail** - every change is logged

> **💡 Quick FAQs:**
> - **Does this replace Linear/Jira?** No! Beads is complementary. [See comparison ↓](#does-beads-replace-human-issue-trackers-like-linear)
> - **Works with Cursor?** Yes! CLI, MCP server, and .cursorrules support. [See guide ↓](#can-beads-be-used-in-cursor)

## Core Architecture

### The Magic: Git-Backed Database

Beads acts like a **centralized database** but is actually **distributed via git**:

1. **Local SQLite cache** (`.beads/*.db`, gitignored) - fast queries
2. **JSONL source of truth** (`.beads/issues.jsonl`, committed to git) - synced across machines
3. **Auto-sync** - SQLite ↔ JSONL automatically sync (5-second debounce)
4. **No server needed** - just git and the `bd` CLI tool

### Dependency Types

Beads supports four types of issue relationships:

1. **blocks** - Hard blocker (A must complete before B can start)
2. **related** - Soft relationship (A and B are connected)
3. **parent-child** - Hierarchical relationship (epic → tasks)
4. **discovered-from** - Work discovered during another task (automatically inherits context)

### Hash-Based Issue IDs (v0.20.1+)

Beads uses **collision-resistant hash IDs** instead of sequential numbers:

- **Old way**: `bd-1`, `bd-2`, `bd-3` → collisions when multiple agents/branches create issues
- **New way**: `bd-a1b2`, `bd-f14c`, `bd-3e7a` → collision-free concurrent creation

**Progressive length scaling:**
- 0-500 issues: 4-character hashes (e.g., `bd-a1b2`)
- 500-1,500 issues: 5-character hashes (e.g., `bd-f14c3`)
- 1,500-10,000 issues: 6-character hashes (e.g., `bd-3e7a5b`)

**Hierarchical children** for work breakdown:
```
bd-a3f8e9      [epic] Auth System
bd-a3f8e9.1    [task] Design login UI
bd-a3f8e9.2    [task] Backend validation
bd-a3f8e9.3    [epic] Password Reset
bd-a3f8e9.3.1  [task] Email templates
```

## Real-World Example Use Case

### Scenario: Building an E-commerce Platform with AI Agents

Imagine you're using Claude (or another AI agent) to build a complex e-commerce platform. The project requires multiple features, bug fixes, and refactoring across several sessions.

#### Without Beads (Traditional Approach)

**Session 1:**
```markdown
# TODO.md
- [ ] Implement user authentication
- [ ] Add shopping cart
- [ ] Create payment integration
  - [ ] Stripe API
  - [ ] PayPal API
- [ ] Add product search
- [ ] Fix bug in checkout flow
```

**Problems:**
1. Agent discovers database migration is needed for auth → adds it to markdown
2. Agent notices security issue in payment flow → mentions it in chat
3. Session ends, context is compacted
4. **Session 2**: Agent reads TODO.md but doesn't know:
   - Which tasks block others
   - What the security issue was
   - What was discovered but not written down
   - Which tasks are ready to start

#### With Beads (Smart Approach)

**Session 1 - Agent Setup:**
```bash
# Agent runs once at project start
bd init
bd create "Build E-commerce Platform" -t epic -p 0
# Returns: bd-7a3f
```

**Session 1 - Agent Discovery During Work:**

While implementing authentication (bd-7a3f.1), the agent discovers issues:

```bash
# Agent files issues as it discovers them
bd create "Implement user authentication" -p 1 -t feature
# Returns: bd-7a3f.1

# While working, agent discovers database schema needs update
bd create "Add users table migration" -p 0 -t task
# Returns: bd-8b2e
bd dep add bd-7a3f.1 bd-8b2e --type blocks
# Auth blocked by migration

# Agent discovers security issue
bd create "Fix SQL injection in login form" -p 0 -t bug -l security,urgent
# Returns: bd-3f9a
bd dep add bd-3f9a bd-8b2e --type discovered-from
# Links security fix back to migration work

# Agent continues creating related tasks
bd create "Add shopping cart" -p 1 -t feature
# Returns: bd-7a3f.2

bd create "Stripe payment integration" -p 2 -t feature
# Returns: bd-4c1d
bd dep add bd-4c1d bd-7a3f.1 --type blocks
# Payment blocked by auth
```

**Session 1 End:**
```bash
# Agent closes completed work
bd close bd-8b2e --reason "Migration created and tested"

# Agent syncs and commits
git add .beads/issues.jsonl
git commit -m "Completed user migration, working on auth"
git push
```

**Session 2 - Different Machine/Agent:**
```bash
# New agent boots up, pulls latest
git pull

# Agent immediately knows what's ready to work on
bd ready --json
```

**Response:**
```json
[
  {
    "id": "bd-7a3f.1",
    "title": "Implement user authentication",
    "priority": 1,
    "status": "open",
    "blockers": 0,
    "type": "feature"
  },
  {
    "id": "bd-3f9a",
    "title": "Fix SQL injection in login form",
    "priority": 0,
    "status": "open",
    "blockers": 0,
    "type": "bug",
    "labels": ["security", "urgent"]
  },
  {
    "id": "bd-7a3f.2",
    "title": "Add shopping cart",
    "priority": 1,
    "status": "open",
    "blockers": 0,
    "type": "feature"
  }
]
```

The agent sees:
- ✅ **bd-3f9a** (P0 security bug) is ready - no blockers
- ✅ **bd-7a3f.1** (auth) is ready - migration is done
- ✅ **bd-4c1d** (payment) is **blocked** - needs auth first
- ✅ Full context from previous session preserved

**Session 2 - Agent Decides on Work:**
```bash
# Agent picks highest priority ready work
bd update bd-3f9a --status in_progress
# Fix the security issue

# During work, discovers another issue
bd create "Add input sanitization library" -p 1 -t task
# Returns: bd-9e4b
bd dep add bd-9e4b bd-3f9a --type discovered-from
# Links new work back to security fix

# Complete the security fix
bd close bd-3f9a --reason "SQL injection fixed with prepared statements"
```

**Session 3 - Dependency Visualization:**
```bash
# Human or agent can visualize the entire dependency tree
bd dep tree bd-7a3f
```

**Output:**
```
bd-7a3f [epic] Build E-commerce Platform
├─ bd-7a3f.1 [feature] Implement user authentication (open)
│  └─ bd-8b2e [task] Add users table migration (closed)
│     └─ bd-3f9a [bug] Fix SQL injection in login form (closed)
│        └─ bd-9e4b [task] Add input sanitization library (open)
├─ bd-7a3f.2 [feature] Add shopping cart (open)
└─ bd-4c1d [feature] Stripe payment integration (open)
   └─ blocks: bd-7a3f.1 (auth must complete first)
```

## Key Benefits in This Example

### 1. **Zero Context Loss**
- Security issue discovered in Session 1 is immediately available in Session 2
- No "I forgot to write that down" moments

### 2. **Smart Work Ordering**
- Agent automatically knows payment integration is blocked by auth
- Can work on shopping cart in parallel with auth

### 3. **Audit Trail**
- Full history of when issues were discovered and why
- `discovered-from` links show how work emerged organically

### 4. **Multi-Agent Coordination**
```bash
# Agent A on laptop
bd create "Add product search" -p 2
# Returns: bd-f7e3

# Agent B on desktop (simultaneously)
bd create "Add search indexing" -p 2
# Returns: bd-a2c9

# No collision! Hash IDs are unique across machines
```

### 5. **Protected Branch Workflows**
```bash
# For repos with protected main branches
bd init --branch beads-metadata

# Updates go to separate branch
# Main branch stays protected
# Issues still sync via git
```

### 6. **Ready Work Detection**
```bash
# Find work that's NOT blocked
bd ready --priority 0  # Critical bugs only
bd ready --label security  # Security issues
bd ready --assignee alice  # Alice's tasks
```

## Can Beads Be Used in Cursor?

**Yes! Beads works perfectly with Cursor.** There are multiple integration methods:

### Method 1: Direct CLI Usage (Simplest)

Cursor's AI can use the `bd` CLI directly via shell commands:

```bash
# Install beads
curl -fsSL https://raw.githubusercontent.com/steveyegge/beads/main/scripts/install.sh | bash

# Initialize in your project
cd your-project
bd init

# Tell Cursor's AI to use it
echo "USE bd (beads) for task tracking. Run 'bd ready --json' to find work." >> .cursorrules
```

Cursor's AI can then execute commands like:
- `bd ready --json` - Find ready work
- `bd create "Fix bug" -t bug -p 1 --json` - Create issues
- `bd update bd-a1b2 --status in_progress --json` - Update status
- `bd show bd-a1b2 --json` - View issue details

### Method 2: MCP Server Integration (Most Powerful)

Beads provides a **Model Context Protocol (MCP)** server that gives Cursor's AI native tool access:

**Install MCP Server:**
```bash
# Using uv (recommended)
uv tool install beads-mcp

# Or using pip
pip install beads-mcp
```

**Configure for Cursor:**

Add to your Cursor settings or workspace MCP configuration:

```json
{
  "mcpServers": {
    "beads": {
      "command": "beads-mcp"
    }
  }
}
```

**What You Get:**
- Native MCP tools: `init`, `create`, `list`, `ready`, `show`, `update`, `close`, `dep`, `blocked`, `stats`
- Automatic workspace detection (works across multiple projects)
- Per-project daemon routing (each project isolated)
- No manual JSON parsing - structured tool responses

### Method 3: Rules File Integration (Best for Consistency)

Create a `.cursorrules` file in your project root:

```markdown
# Beads Issue Tracker

USE the bd (beads) CLI for ALL task tracking and project memory.

## Essential Commands

1. Find work: `bd ready --json`
2. Create issue: `bd create "title" -t type -p priority --json`
3. Update status: `bd update <id> --status <status> --json`
4. Show details: `bd show <id> --json`
5. Close issue: `bd close <id> --reason "completed" --json`

## Workflow

1. At session start: Run `bd ready --json` to see what's ready
2. During work: Create issues for discovered bugs/TODOs
3. Link discoveries: `bd dep add <new-id> <parent-id> --type discovered-from`
4. Before finishing: Update/close issues and sync to git

## Types
- bug, feature, task, epic, chore

## Priorities
- 0 (critical), 1 (high), 2 (medium), 3 (low), 4 (backlog)

## Important
- Always use --json flag for programmatic parsing
- File issues as you discover them (don't wait)
- Track dependencies to prevent blocked work
```

### Multi-Project Support

Beads automatically detects which project you're working on:

```bash
# Working on Project A
cd ~/projects/webapp
bd ready --json  # Uses webapp's .beads/

# Working on Project B  
cd ~/projects/api
bd ready --json  # Uses api's .beads/

# Each project completely isolated
```

With MCP server, one configuration works for all projects via automatic daemon routing!

### Example Cursor Workflow

**Session Start:**
```bash
# Cursor AI runs this automatically
bd ready --json | jq '.[0]'
```

**During Implementation:**
```bash
# AI discovers a bug while coding
bd create "Fix memory leak in cache" -t bug -p 1 --json

# AI links it to current work
bd dep add bd-f7e3 bd-a1b2 --type discovered-from

# AI updates status
bd update bd-a1b2 --status in_progress --json
```

**Session End:**
```bash
# AI closes completed work
bd close bd-a1b2 --reason "Implemented and tested" --json

# AI commits to git
git add .beads/issues.jsonl
git commit -m "Completed bd-a1b2"
git push
```

### Why Beads + Cursor = Powerful

1. **Persistent Memory** - Cursor doesn't forget across sessions
2. **Automatic Discovery** - AI files issues as it finds problems
3. **Dependency Tracking** - AI understands what blocks what
4. **JSON Output** - Perfect for programmatic parsing
5. **Git-Backed** - Syncs across machines automatically
6. **Multi-Project** - Works seamlessly with multiple repos

### Installation & Quick Start

### For Humans (One-Time Setup)

```bash
# Install beads
curl -fsSL https://raw.githubusercontent.com/steveyegge/beads/main/scripts/install.sh | bash

# Initialize in your project
cd your-project
bd init

# Tell your AI agent to use it
echo "USE bd (beads) for ALL task tracking - run 'bd onboard' first" >> AGENTS.md
```

### For AI Agents (Automatic)

When your agent starts a session, it should:

```bash
# 1. Check for beads
bd ready --json

# 2. If no .beads/ directory exists
bd init --quiet

# 3. Find work
bd ready --json | jq '.[0]'

# 4. During work, file issues as discovered
bd create "Found bug in payment flow" -t bug -p 0
bd dep add <new-issue-id> <current-work-id> --type discovered-from

# 5. Update status
bd update <issue-id> --status in_progress

# 6. Complete work
bd close <issue-id> --reason "Implemented and tested"

# 7. Sync before session ends
bd sync
git add .beads/issues.jsonl
git commit -m "Session update"
git push
```

## Advanced Features

### Memory Decay (Compaction)
Old closed issues can be automatically summarized to save context:

```bash
# Agent-driven (recommended)
bd compact --analyze --json  # Get candidates
# Agent summarizes with any LLM
bd compact --apply --id bd-42 --summary summary.txt
```

### Labels for Organization
```bash
bd create "Fix auth bug" -l backend,security,urgent
bd list --label security  # Find all security issues
bd ready --label-any frontend,ui  # Find frontend OR ui work
```

### Multi-Repo Workflows
- **OSS Contributors**: Use `--contributor` mode for fork workflows
- **Teams**: Use `--team` mode for branch workflows
- **Protected Branches**: Use `--branch beads-metadata` for separate sync branch

### Real-Time Updates
Optional Agent Mail system for <100ms sync (vs 2-5s git sync):
```bash
# See docs/AGENT_MAIL.md for setup
bd daemon start --agent-mail
```

## Why Beads Matters

### Traditional AI Agent Problems:
1. ❌ Markdown TODOs get stale and forgotten
2. ❌ No dependency tracking
3. ❌ Agents forget issues across sessions
4. ❌ No way to find "ready work"
5. ❌ Lost context when compacting conversations

### With Beads:
1. ✅ Git-backed persistent memory
2. ✅ Four types of dependency tracking
3. ✅ Automatic issue discovery during work
4. ✅ Smart "ready work" detection
5. ✅ Full audit trail and context preservation

## Technical Details

- **Language**: Go
- **Storage**: SQLite (local cache) + JSONL (git-backed)
- **License**: MIT
- **Requirements**: 
  - Linux: glibc 2.32+ (Ubuntu 22.04+)
  - macOS/Windows: No special requirements
- **Performance**: 1000 issues imported in ~950ms

## Ecosystem

- **CLI**: `bd` command-line tool
- **Web UI**: Real-time monitoring dashboard
- **Claude Desktop MCP**: Model Context Protocol integration
- **Claude Code Plugin**: Slash commands for quick access
- **Python/Bash Agents**: Example implementations
- **Beadster**: Native macOS app for viewing issues

## Resources

- **Repository**: https://github.com/steveyegge/beads
- **Documentation**: Extensive docs/ directory with guides for:
  - Installation, quickstart, troubleshooting
  - Agent Mail (real-time sync)
  - Multi-repo workflows
  - Protected branch support
  - Advanced features and extensions

## Does Beads Replace Human Issue Trackers Like Linear?

**Short answer: No. Beads is complementary, not a replacement.**

### The Division of Labor

**Beads = AI Agent Memory**
- Designed FOR AI agents to use on your behalf
- Tracks discovered work, dependencies, and implementation details
- Git-backed, offline-first, branch-scoped
- Lives in your codebase (`.beads/` directory)
- Fast, lightweight CLI with JSON output for programmatic use

**Linear/Jira/GitHub Issues = Human Team Coordination**
- Designed FOR humans with rich UIs
- Product roadmaps, sprint planning, stakeholder communication
- Centralized service with real-time collaboration
- Cross-project dashboards, notifications, comments, discussions
- Integrations with Slack, email, calendars, etc.

### How They Work Together

```
┌─────────────────────────────────────────────┐
│  HUMAN SPACE (Linear/Jira)                  │
│  • Product planning                         │
│  • Feature requests                         │
│  • Sprint goals                             │
│  • Stakeholder communication                │
└───────────────┬─────────────────────────────┘
                │
                │ "Build authentication system"
                ↓
┌─────────────────────────────────────────────┐
│  AI AGENT SPACE (Beads)                     │
│  • bd-a3f8 [epic] Auth system               │
│  • bd-a3f8.1 [task] Database migration      │
│  • bd-a3f8.2 [task] Login UI                │
│  • bd-f7e3 [bug] SQL injection fix          │
│  • bd-9b2c [task] Add rate limiting         │
│  • bd-4d1a [chore] Update tests             │
└─────────────────────────────────────────────┘
```

### Real-World Workflow

**1. Human creates high-level ticket in Linear:**
```
LINEAR-123: Implement user authentication
- SSO with Google
- Password reset flow
- Session management
```

**2. AI agent breaks it down in Beads:**
```bash
# Agent creates epic linked to Linear ticket
bd create "Auth system" -t epic -p 1 --external-ref "LINEAR-123"
# Returns: bd-a3f8

# During implementation, agent discovers sub-tasks:
bd create "Add users table migration" -p 0  # bd-a3f8.1
bd create "Implement OAuth flow" -p 1        # bd-a3f8.2
bd create "Fix session cookie bug" -t bug    # bd-f7e3 (discovered during work)
bd create "Add rate limiting" -p 2           # bd-9b2c (discovered during work)
```

**3. Agent tracks dependencies:**
```bash
bd dep add bd-a3f8.2 bd-a3f8.1 --type blocks  # OAuth blocked by migration
bd dep add bd-f7e3 bd-a3f8.2 --type discovered-from  # Bug found during OAuth work
```

**4. Human checks Linear for high-level status:**
- LINEAR-123 → "In Progress"
- Comments from agent about technical blockers
- Links back to specific commits

**5. Agent uses Beads for execution:**
```bash
bd ready --json  # Find next task with no blockers
bd close bd-a3f8.1 --reason "Migration complete"
bd update bd-a3f8.2 --status in_progress
```

### Integration Pattern

Beads supports `external_ref` field for linking to human tools:

```bash
# Link to Linear
bd create "Fix auth bug" -t bug --external-ref "LINEAR-456"

# Link to Jira
bd create "API refactor" -t task --external-ref "PROJ-789"

# Link to GitHub Issues
bd create "Memory leak" -t bug --external-ref "gh-42"
```

You can configure integration settings:
```bash
bd config set linear.api_key "YOUR_KEY"
bd config set jira.url "https://company.atlassian.net"
```

### Key Differences

| Feature | Beads (Agent) | Linear/Jira (Human) |
|---------|---------------|---------------------|
| **Primary User** | AI coding agents | Human teams |
| **Interface** | CLI + JSON API | Rich web UI + mobile |
| **Storage** | Git-backed JSONL | Cloud database |
| **Scope** | Per-repository | Cross-organization |
| **Dependencies** | 4 types (blocks, related, parent-child, discovered-from) | Basic linking |
| **Ready Work** | Automatic detection based on blockers | Manual prioritization |
| **Offline** | Full functionality | Limited |
| **Discovery** | Agents auto-file issues during work | Humans manually create |
| **Audit Trail** | Every change logged | Activity feeds |
| **Cost** | Free, open-source | Usually paid SaaS |

### When to Use What

**Use Beads when:**
- ✅ AI agent is implementing features
- ✅ Tracking technical dependencies and sub-tasks
- ✅ Agent discovering bugs/TODOs during work
- ✅ Need perfect memory across agent sessions
- ✅ Working offline or in git-heavy workflows

**Use Linear/Jira when:**
- ✅ Planning product roadmap with stakeholders
- ✅ Sprint planning and team coordination
- ✅ Cross-project reporting and dashboards
- ✅ Human discussions and approvals
- ✅ Integrating with team communication tools

**Use Both when:**
- ✅ You have AI agents doing implementation work
- ✅ You need human oversight on product direction
- ✅ You want agents to track technical details automatically
- ✅ You want humans to focus on strategy, not implementation minutiae

### FAQ Quote

From the Beads documentation:

> **Can I use bd without AI agents?**
> 
> Absolutely! bd is a great CLI issue tracker for humans too. The `bd ready` command is useful for anyone managing dependencies. Think of it as "Taskwarrior meets git."

> **Why not just use GitHub Issues?**
>
> GitHub Issues excels for human teams in web UI with cross-repo dashboards and integrations. bd excels for AI agents needing offline, git-synchronized task memory with graph semantics and deterministic queries.

### The Bottom Line

**Beads doesn't replace Linear/Jira** — it gives your AI agents their own workspace for managing implementation details that would clutter human tools. 

Think of it this way:
- **Linear** = The project manager's whiteboard (high-level strategy)
- **Beads** = The developer's notebook (implementation details)
- **AI Agent** = The developer who can now remember everything in their notebook across sessions

Your human team continues using their preferred tools for planning and coordination. Your AI agents use Beads to track the hundreds of micro-tasks, dependencies, and discovered issues that emerge during actual coding work.

## Conclusion

Beads transforms AI coding agents from forgetful assistants into organized project managers with perfect memory. By using a git-backed issue tracker designed specifically for agents, it enables:

- **Long-horizon planning** without context loss
- **Automatic work discovery** and dependency tracking
- **Multi-agent coordination** without conflicts
- **Audit trails** for complex multi-session operations

If you're using AI agents for coding, Beads is the memory upgrade that makes them exponentially more effective at handling complex, real-world software projects — while your human team continues using their favorite tools like Linear, Jira, or GitHub Issues for product planning and coordination.

**Beads = Agent Memory | Linear/Jira = Human Coordination | Together = Powerful Workflow**

---

*Last updated: November 2025*  
*Project status: Alpha (v0.20.1+) - core features stable, expect API changes before 1.0*
