# Agent Sessions Protocol - Quick Reference

> **📖 For detailed examples and troubleshooting:** See [SESSIONS-REFERENCE.md](../knowledge/llm-coding-agent-patterns/SESSIONS-REFERENCE.md)

## Quick Start Commands

```bash
# Claim and activate session
./_bin/claim-session 2025-10-14-feature-x

# Activate session environment (in session codebase)
cd _AGENTS/sessions/active/2025-10-14-feature-x/.codebase
source ../.session-env

# Complete session (unlocks SESSION.md for final updates)
cd ../../../../..
./_bin/complete-session 2025-10-14-feature-x
```

## Essential Rules

### 1. Session States
- **drafting/** - Being defined, not ready
- **planned/** - Ready to claim
- **active/** - Currently being worked on
- **completed/** - Successfully finished
- **abandoned/** - Cancelled/incomplete

### 2. File Permissions
- **SESSION.md is READ-ONLY** during active sessions (chmod 444)
- Use **worklog.md** for progress updates
- Use **active-plan.md** for task changes
- Use **subsessions.md** for scope additions

### 3. Session Naming
- Format: `YYYY-MM-DD-descriptive-slug`
- Examples: `2025-10-14-auth-system`, `2025-10-14-api-refactor`
- KB merge sessions: `kb-YYYY-MM-DD-merge-topic`

### 4. Git Branch Naming
- Format: `session/{session-id}`
- Examples: `session/2025-10-14-auth-system`

### 5. Commit Message Format
- Format: `[{session-id}] <type>: <description>`
- Examples: `[2025-10-14-auth-system] feat: add user authentication`

### 6. Knowledge Base Rules
- **Read KB**: Anytime from `knowledge/`
- **Write Learnings**: During work to `knowledge/sessions/{session}/`
- **Merge to Canonical**: Only via KB merge sessions to `knowledge/`

### 7. Multi-Agent Coordination
- **Always pull before claiming** sessions
- **Handle race conditions gracefully** - pick different session if claim fails
- **Namespace everything** - use `{session-slug}/.codebase/`
- **Session-prefixed commits** - every commit tagged with `[{session-id}]`
- **Coordinate via git** - no external tools needed

### 8. Environment Variables (Set automatically)
- `GIT_AUTHOR_NAME` - Agent-specific git author
- `GIT_AUTHOR_EMAIL` - Agent-specific git email  
- `SESSION_SLUG` - Session identifier
- `SESSION_BRANCH` - Session branch name

### 9. Critical Directories
- **Main workspace**: `_AGENTS/sessions/active/{session-slug}/.codebase/`
- **Session metadata**: `_AGENTS/sessions/active/{session-slug}/`
- **Session lock**: `.agents/sessions.lock`

### 10. Emergency Override
- Only use in critical situations
- Document reason in worklog.md first
- Use: `chmod 644 sessions/active/{session}/SESSION.md`
- Always restore read-only after: `chmod 444 sessions/active/{session}/SESSION.md`

## Verification Checklist
- [ ] Session claimed successfully
- [ ] Environment variables set (`echo $GIT_AUTHOR_NAME`)
- [ ] Working in correct directory (`.codebase/`)
- [ ] Using session-prefixed commits
- [ ] Updating worklog.md regularly
- [ ] Capturing learnings in `knowledge/sessions/{session}/`
- [ ] SESSION.md remains read-only during work

## Common Issues
- **Claim fails**: Another agent got it first, try different session
- **Can't edit SESSION.md**: Expected behavior, use worklog.md instead
- **Wrong git author**: Check environment variables are sourced
- **Merge conflicts**: Document resolution in worklog.md

---
**📚 For complete examples and troubleshooting:** See [SESSIONS-REFERENCE.md](../knowledge/llm-coding-agent-patterns/SESSIONS-REFERENCE.md)