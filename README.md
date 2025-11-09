# Beads: A Memory Upgrade for AI Coding Agents

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

## Installation & Quick Start

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

## Conclusion

Beads transforms AI coding agents from forgetful assistants into organized project managers with perfect memory. By using a git-backed issue tracker designed specifically for agents, it enables:

- **Long-horizon planning** without context loss
- **Automatic work discovery** and dependency tracking
- **Multi-agent coordination** without conflicts
- **Audit trails** for complex multi-session operations

If you're using AI agents for coding, Beads is the memory upgrade that makes them exponentially more effective at handling complex, real-world software projects.

---

*Last updated: November 2025*  
*Project status: Alpha (v0.20.1+) - core features stable, expect API changes before 1.0*
