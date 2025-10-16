# Agent Sessions Protocol

> **📖 For detailed examples, git commands, and troubleshooting:** See [SESSIONS-REFERENCE.md](SESSIONS-REFERENCE.md)

## Purpose

Sessions are **structured units of work** that enable multiple AI agents to collaborate on a codebase concurrently without conflicts. Each session:

- Has clear **context** (what needs to be done)
- Has **acceptance criteria** (definition of done)
- Tracks **progress** (worklog, decisions, lessons learned)
- Produces **artifacts** (code changes, documentation, knowledge)
- Maintains **traceability** (git attribution, patch files)

Sessions move through states (`drafting` → `planned` → `active` → `completed`) as work progresses, creating a clear audit trail of what was done, by whom, and why.

### Basic Flow

Sessions move through states as work progresses:

```mermaid
flowchart LR
    Drafting["drafting/
    (being defined)"] -->|"Ready"| Planned["planned/
    (ready to claim)"]
    Planned -->|"Claim"| Active["active/
    (in progress)"]
    Active -->|"Complete"| Completed["completed/
    (merged)"]
    Active -->|"Cancel"| Abandoned["abandoned/
    (documented)"]
```

1. **Draft** - Session created in `drafting/` (context, criteria, plan incomplete)
2. **Ready** - Moved to `planned/` when ready for agents to claim
3. **Claim** - Agent atomically claims session from `planned/` via git push
4. **Activate** - Source `.session-env` to establish agent identity
5. **Work** - Make changes, update worklog, capture learnings
6. **Complete** - Generate patch, create KB merge session if needed, merge to main

### Multi-Agent Coordination

**Background agents can monitor `planned/`** for sessions matching their capabilities.

Multiple sessions work concurrently:
- Session `2025-10-14-auth-system` → works → completes
- Session `2025-10-14-api-refactor` → works → completes (in parallel)
- Sessions never block each other

Coordination through **git** (no orchestrator):
- Session claims via atomic git push
- Namespace isolation via branch names and commits
- Optimistic locking handles race conditions gracefully

## Quick Start

### Using Utility Scripts (Recommended)

```bash
# Claim and activate session
./_bin/claim-session 2025-10-14-feature-x
# Note: SESSION.md becomes read-only to preserve original plan

# Activate session environment (in session codebase)
cd _AGENTS/sessions/active/2025-10-14-feature-x/.codebase
source ../.session-env

# Work on session (use worklog.md, active-plan.md for updates)...

# Complete session (unlocks SESSION.md for final updates)
cd ../../../../..
./_bin/complete-session 2025-10-14-feature-x
```

### Manual Session Management

For advanced users who need to understand the underlying process, see the detailed procedures in [SESSIONS-REFERENCE.md](SESSIONS-REFERENCE.md#detailed-implementation-examples).

The manual process involves:
1. **Session claiming** - Update main repository, atomically claim session via git push, move to active, create environment file
2. **Session clone creation** - Create shallow clone in `{session-slug}/.codebase/` within session directory, create session branch, configure remote
3. **Environment activation** - Source session environment to establish agent identity
4. **Work completion** - Make changes, update documentation, capture learnings
5. **Session cleanup** - Generate patch, merge to main, archive session, remove clone, update lock file
6. **Session cancellation** - Document reasons, move to abandoned, remove clone, update lock file

**Automation:** The complete manual procedure is implemented in:
- `sessions/_bin/claim-session` - Automates steps 1-3
- `sessions/_bin/complete-session` - Automates step 5

## Implementation SOP

### Core Principles

1. **Git as Coordinator** - Use git itself for synchronization (no external orchestrator)
2. **Session-Scoped Activation** - Agent identity via environment variables, session lifecycle
3. **Hub-Spoke Architecture** - Main repo as hub, session clones as isolated spokes
4. **Complete Isolation** - Each session works in independent repository clone
5. **Optimistic Locking** - Session claims via atomic git operations
6. **Full Traceability** - Every commit attributed to specific agent
7. **Two-Phase Knowledge** - Capture learnings fast, merge deliberately via KB sessions

### Directory Structure

```
_AGENTS/sessions/
├── .gitignore        # Local gitignore for session-specific patterns
├── sessions.lock     # Active session claims (session-id:timestamp)
├── _bin/             # Utility scripts
│   ├── claim-session
│   └── complete-session
├── _templates/       # Templates for sessions and configs
│   ├── SESSION.md.j2           # Standard session template
│   ├── kb-merge-SESSION.md     # KB merge session template
│   └── session-env.template    # Session environment template
├── SESSIONS-README.md        # This file (essential protocol)
├── SESSIONS-REFERENCE.md     # Detailed examples & commands
├── abandoned/       # Cancelled/incomplete sessions
├── active/          # Currently active sessions (metadata only)
│   ├── 2025-10-14-auth-system/
│   │   ├── .session-env
│   │   ├── .codebase/           # Shallow clone for this session (ignored)
│   │   ├── SESSION.md
│   │   ├── worklog.md
│   │   └── active-plan.md
│   ├── 2025-10-14-api-work/
│   │   ├── .session-env
│   │   ├── .codebase/           # Shallow clone for this session (ignored)
│   │   └── ...
│   └── ...
├── completed/       # Finished sessions (all agents)
├── drafting/        # Sessions being defined (not ready for agents)
└── planned/         # Ready to claim (agents monitor this)

# Note: Session clones live as .codebase/ within each session directory
# Example: active/2025-10-14-auth-system/.codebase/ contains the shallow clone
# The .codebase/ directories are ignored via the local .gitignore file
```

**Utilities** (`_bin/`, `_templates/`) sort first, keeping them separate from **state directories** (`abandoned/`, `active/`, `completed/`, `drafting/`, `planned/`).

### Local .gitignore Strategy

The `_AGENTS/sessions/.gitignore` file provides **session-specific ignore patterns** that are:

- **Local to the sessions directory** - Keeps session ignore rules separate from main project
- **Version controlled** - Shared across all developers and agents
- **Scoped to session artifacts** - Ignores `.codebase/` clones, temporary files, build artifacts
- **Overrides project-level patterns** - More specific than root `.gitignore`

**Key patterns ignored:**
- `*/.codebase/` - Session clones (the most important pattern)
- `sessions.lock` - Session lock file
- Temporary files (`*.tmp`, `*.swp`, `*~`)
- IDE files (`.vscode/`, `.idea/`)
- Build artifacts (`build/`, `dist/`, `target/`)
- OS files (`.DS_Store`, `Thumbs.db`)

This approach keeps session-specific artifacts out of git while maintaining clean separation between session management and the main codebase.

### Session States

| State | Location | Description |
|-------|----------|-------------|
| **Drafting** | `drafting/` | Being defined, not ready for agents yet |
| **Planned** | `planned/` | Ready to claim, agents can monitor this |
| **Active** | `active/` | Being worked on by an agent |
| **Completed** | `completed/` | Successfully finished and merged |
| **Abandoned** | `abandoned/` | Cancelled or incomplete, documented |

### SESSION.md Read-Only Protection

**Purpose: Drift Tracking**
- SESSION.md files become **read-only** when sessions move to `active/`
- This preserves the original plan for drift analysis
- Compare original plan vs. actual work to identify scope changes
- Learn from planning inaccuracies for future sessions

**When SESSION.md is Read-Only:**
- In `active/` sessions: **Read-only** (chmod 444)
- In `completed/` sessions: **Read-only** (chmod 444)
- In `planned/` and `drafting/`: **Writable** (chmod 644)

**Update Channels During Active Work:**
- `worklog.md` - Progress, decisions, timestamps
- `active-plan.md` - Current tasks, issues, next steps
- `subsessions.md` - Scope additions (creates new sessions)

**Unlock Process (Completion Only):**
1. `_AGENTS/sessions/_bin/complete-session` script unlocks SESSION.md for final updates
2. Agent can add final notes if needed
3. SESSION.md becomes read-only again in `completed/`

**Emergency Override:** Only use in critical situations. Follow the procedure documented in [SESSIONS-REFERENCE.md](SESSIONS-REFERENCE.md#troubleshooting).

**Why This Matters:**
- Enables drift analysis between planned vs. actual work
- Catches scope creep early
- Improves future session planning accuracy
- Maintains audit trail of original intent

**FAQ: Why is SESSION.md read-only?**
- **Answer:** To preserve the original plan for drift analysis. By keeping SESSION.md unchanged, we can compare what was planned vs. what actually happened, identify scope creep, and learn from planning inaccuracies.

**FAQ: How do I track scope changes?**
- **Answer:** Use `worklog.md` for progress updates, `active-plan.md` for task changes, and `subsessions.md` for scope additions that create new sessions. These files are writable during active work.


### Naming Conventions

#### Session Folders

Format: `YYYY-MM-DD-descriptive-slug`

**Standard sessions:**
- `2025-10-14-auth-system`
- `2025-10-14-api-refactor`

**KB merge sessions:**
- `kb-2025-10-14-merge-auth-patterns`
- `kb-2025-10-14-merge-api-security`

#### Git Branches

Format: `session/{session-id}`

**Examples:**
- `session/2025-10-14-auth-system`
- `session/kb-2025-10-14-merge-auth-patterns`

#### Commit Messages

Format: `[{session-id}] <type>: <description>`

**Examples:**
- `[2025-10-14-auth-system] feat: add user authentication`
- `[2025-10-14-api-refactor] fix: resolve memory leak`
- `[2025-10-14-docs-update] docs: update API documentation`

### Session Contents

#### Standard Session Files

- **`SESSION.md`** - Context, acceptance criteria, subsessions (with TDD structure)
- **`worklog.md`** - Progress tracking with timestamps
- **`active-plan.md`** - Dynamic task lists, issues, next steps
- **`subsessions.md`** - Sub-session tracking
- **`{session-slug}.patch`** - Final patch file (generated at completion)

#### Session Templates

The `_templates/` directory provides Jinja2 templates for creating consistent session files:

**`SESSION.md.j2`** - Standard session template with comprehensive structure:
- **Variables**: `SESSION_SLUG`, `CONTEXT`, `PROBLEM_STATEMENT`, `ACCEPTANCE_CRITERIA`, `SUBSESSIONS`, `RISKS`, `DEPENDENCIES`, `EDGE_CASES`, `NOTES`
- **Usage**: Automatically used by session creation tools, or manually with template substitution
- **Structure**: Includes context, problem analysis, comprehensive acceptance criteria (combining requirements and success metrics), subsessions (with implicit TDD RED->GREEN->REFACTOR cycle), risk mitigation, dependencies, and edge case considerations

**`kb-merge-SESSION.md.j2`** - KB merge session template:
- **Variables**: `TOPIC`, `SOURCE_SESSION`, `TIMESTAMP`
- **Usage**: Automatically created by complete-session script when learnings exist
- **Purpose**: Guides systematic merging of session learnings into canonical knowledge base

**`session-env.template.j2`** - Session environment template:
- **Variables**: `SESSION_SLUG`, `USER_NAME`, `USER_EMAIL`
- **Usage**: Automatically created by claim-session script for session activation
- **Purpose**: Establishes agent identity and session-specific environment variables

#### KB Merge Session Files

Simplified structure for KB merge sessions:
- **`SESSION.md`** - Auto-generated with source session reference
- **`worklog.md`** - KB merge decisions and conflicts

> **📊 For detailed state flowcharts:** See [SESSIONS-REFERENCE.md](SESSIONS-REFERENCE.md#detailed-state-flowcharts)

## Knowledge Base SOP

### Two-Phase Strategy

**Phase 1: Session-Scoped Capture (During Work)**
- Write to: `_AGENTS/knowledge/sessions/{session-slug}/learnings.md`
- Isolated per session, zero conflicts
- Fast, autonomous documentation

**Phase 2: Canonical Merge (Dedicated Session)**
- KB merge session auto-created at completion
- Any agent can execute merge
- Deliberate review and quality control
- Merge to: `_AGENTS/knowledge/`

### KB Access Rules

| Action | Path | When | Who |
|--------|------|------|-----|
| **Read KB** | `knowledge/` | Anytime | All agents |
| **Write Learnings** | `knowledge/sessions/{session}/` | During work | Owning agent |
| **Merge to Canonical** | `knowledge/` | KB merge session only | Assigned agent |

**Critical:** Only `kb-` prefixed sessions may write to the canonical knowledge base. All other sessions must write exclusively to `knowledge/sessions/{session-id}/`.

## Git Workflow SOP

### Session Lifecycle Overview

The Git workflow follows three distinct phases: **Session Initialization**, **Ongoing Work**, and **Session Completion/Cancellation**. Each phase uses standard Git operations within the isolated session clone.

### 1. Session Initialization & Claiming

**Automated via:** `_AGENTS/sessions/_bin/claim-session {session-slug}`

**Manual procedure:**
1. Update main repository from remote
2. Check session availability in `_AGENTS/sessions/sessions.lock`
3. Atomically claim session via git push to lock file
4. Move session from `planned/` to `active/` with read-only SESSION.md protection
5. Create session environment file (`.session-env`) with agent identity variables
6. Create shallow clone in `active/{session-slug}/.codebase/` from main repository
7. Create session-specific branch within clone and configure remote as upstream
8. Provide activation instructions for working in the session

### 2. Ongoing Work

**Working Environment:**
- Primary workspace: `_AGENTS/sessions/active/{session-slug}/.codebase/` (shallow clone)
- Session metadata: `_AGENTS/sessions/active/{session-slug}/` (environment, tracking files)
- Remote configuration: `origin` renamed to `upstream` (points to main repo)

**Branch Strategy:**
- Session works on `session/{session-slug}` branch within session clone
- Main repository remains on base branch (e.g., `main`)
- Session branch is isolated from other sessions
- Periodic pushes to main repository for backup

**Commit Strategy:**
- All commits automatically attributed via session environment variables
- Code changes committed with session-prefixed messages
- Session metadata updated in main repository separately from code changes
- Knowledge capture in session-scoped directories
- Always use specific file paths rather than `git add .`

**File Management:**
- Code changes: Made within `{session-slug}/.codebase/`, committed to session branch
- Session metadata: Updated in `{session-slug}/` directory, committed to main branch
- Knowledge artifacts: Created in `_AGENTS/knowledge/sessions/{session}/`

### 3. Session Completion

**Automated via:** `_AGENTS/sessions/_bin/complete-session {session-slug}`

**Manual procedure:**
1. Generate patch file from session work in `.codebase/`
2. Check for KB learnings and create merge session if needed
3. Remove session clone directory (`.codebase/`)
4. Merge session branch to main via squash merge
5. Remove session entry from `_AGENTS/sessions/sessions.lock`
6. Move session from `active/` to `completed/`
7. Delete session branch and clean up
8. Provide instructions for environment deactivation

**Post-Completion:**
- Session remains read-only in `completed/` for audit purposes
- Patch file available for review
- KB merge sessions created automatically when learnings exist
- Environment variables should be unset

### 4. Session Cancellation

**Automated via:** `_AGENTS/sessions/_bin/cancel-session {session-slug}` (to be implemented)

**Manual procedure:**
1. Document cancellation reasons in session `worklog.md`
2. Remove session clone directory (`.codebase/`)
3. Remove session entry from `_AGENTS/sessions/sessions.lock`
4. Move session from `active/` to `abandoned/`
5. Create abandonment commit with clear documentation
6. Optionally create KB merge session for partial learnings
7. Clean up session branch if created

**Post-Cancellation:**
- Session moved to `abandoned/` with documented reasons
- Partial work available for future reference
- Lock file updated to free session slot

### Branch and Merge Strategy

**Session Branches:**
- Format: `session/{session-slug}`
- Created during session initialization within session clone
- Used exclusively within session's `.codebase/` directory
- Merged to main via squash merge during completion
- Deleted after successful completion or cancellation

**Main Repository Integration:**
- Main repository receives squash merges from completed sessions
- Clean history maintained with single commit per session
- Session contributions clearly attributed via commit messages
- Conflict resolution documented in session worklog when needed

## Conflict Resolution

### Conflict Types

| Type | Strategy | How It Works |
|------|----------|--------------|
| **Session Files** | Namespace isolation | Each session in `active/{session-slug}/` |
| **KB Learnings** | Session-scoped | Each session in `sessions/{session-slug}/` |
| **Canonical KB** | KB merge sessions | Only via dedicated sessions |
| **Code Files** | Git merge | Standard resolution, document in worklog |
| **Session Claims** | Optimistic locking | Retry with different session |

See [SESSIONS-REFERENCE.md](SESSIONS-REFERENCE.md#conflict-resolution-examples) for detailed examples.

## Best Practices

### General
1. Update documentation frequently
2. Document decisions for future agents
3. Be honest about failures and learnings
4. Clean up temporary files

### Multi-Agent Specific
5. **Always pull before claiming** - Get latest state first
6. **Handle race conditions gracefully** - Pick different session if claim fails
7. **Namespace everything** - Use `{session-slug}/.codebase/` and `session/{session-id}`
8. **Session-prefixed commits** - Every commit tagged with `[{session-id}]`
9. **KB learnings are session-scoped** - Never write directly to `knowledge/`
10. **Create KB merge sessions** - Auto-generate at session completion
11. **Verify session activation** - Check environment variables are set (`echo $GIT_AUTHOR_NAME`)
12. **Coordinate via git** - No file system locks or external tools

---

**📚 For more details:** See [SESSIONS-REFERENCE.md](SESSIONS-REFERENCE.md) for complete examples, git commands, and troubleshooting.
