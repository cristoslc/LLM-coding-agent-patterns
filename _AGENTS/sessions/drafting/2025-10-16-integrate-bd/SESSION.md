# Session: Integrate bd Issue Tracker

**Session ID:** 2025-10-16-integrate-bd
**Created:** 2025-10-16
**Status:** drafting
**Priority:** high
**Type:** infrastructure

## Context

Currently, subsessions are tracked manually in `subsessions.md` files. This requires manual dependency management, status tracking, and "what's next" queries. The bd issue tracker (https://github.com/cristoslc/llm-beads) provides:

- Dependency-aware task tracking with blocking relationships
- Cycle detection
- Queryable status via JSON API
- Git-friendly JSONL export
- `bd ready` to surface unblocked work

This session integrates bd to replace manual subsession tracking while preserving the session protocol's strengths.

## Goals

Transform the session workflow to use bd for:
- Subsession tracking (with cross-subsession dependencies)
- TDD cycle enforcement (RED → GREEN → REFACTOR → QA → FINISH)
- Fine-grained task management within TDD phases
- Agent-friendly "ready work" queries

Preserve:
- Session-level coordination via sessions.lock
- Worklog for decision history
- KB learnings workflow
- Git worktrees isolation

## Session Directory Structure

Each session directory contains files with specific roles that complement bd:

### SESSION.md (Read-Only During Active Work)
- **Created:** During session initialization in drafting/
- **Permissions:** Made read-only by session-claim when moved to active/
- **Made Writeable:** By session-complete when moved to completed/
- **Purpose:** Session contract - what we're building, acceptance criteria, technical approach
- **Updates:** Only possible in drafting/ or completed/ status; locked during active work
- **Relationship to bd:** bd issues implement the work defined here

### worklog.md (Chronological Record)
- **Purpose:** Captures WHEN and WHY decisions were made
- **Complementary to bd:** bd tracks WHAT (tasks/issues), worklog tracks decision rationale
- **Key entries:**
  - When new bd issues are created (with rationale)
  - When bd issues are completed (with outcomes)
  - Decision points and trade-offs
  - Obstacles encountered and how they were resolved
  - Context that would be lost in issue tracking alone
- **Format:** Chronological, timestamped entries
- **Used for:** Learning extraction, session completion review

### scratchpads/ (Per-Subsession Thinking)
- **Structure:** One scratchpad file per subsession (`scratchpads/subsession-1-setup.md`)
- **Purpose:** Externalize agent thought processes during active work
- **Contents:**
  - Current understanding of the problem
  - Obstacles and debugging notes
  - Experiments and their outcomes
  - Open questions and answers discovered
  - Links to relevant bd issues
- **Lifecycle:**
  - Created at subsession start
  - Updated throughout subsession
  - Archived at subsession completion
  - Combined with worklog → learnings.md (in KB per SOP)
- **Note:** Replaces `active-plan.md` with per-subsession scoping

### .beads/ (bd Database)
- **Contents:**
  - SQLite database (*.db files) - NOT tracked in git
  - JSONL exports (*.jsonl) - tracked in git for merge-friendliness
- **Purpose:** Task tracking, dependencies, status queries
- **Queried by:** work-ready script, subsession-complete validation

### Learning Extraction Workflow

```
During subsession:
  scratchpads/subsession-N.md  ← active thinking
  worklog.md                   ← decision log

At subsession completion:
  scratchpad + worklog → _AGENTS/knowledge/learnings.md
  (Extract patterns, reusable solutions, pitfalls avoided)

At session completion:
  learnings.md → new kb-* sessions in drafting/
  (Transform learnings into actionable knowledge base improvements)
```

### Relationship Between Files

```
SESSION.md (read-only)
  ↓ defines
bd issues (WHAT to do)
  ↓ worked on via
scratchpads/subsession-N.md (HOW thinking)
  ↓ decisions logged in
worklog.md (WHEN/WHY decisions)
  ↓ combined into
learnings.md (KB extraction)
  ↓ becomes
kb-* sessions (Knowledge improvement)
```

## Acceptance Criteria

### Infrastructure
- [ ] bd installed and documented in project setup
- [ ] bd initializes automatically during session-claim
- [ ] .beads/ directory properly ignored/tracked (db vs jsonl)
- [ ] .gitignore updated appropriately

### Scripts Updated
- [ ] session-claim: Initialize bd in worktree, create scratchpads/ directory, set SESSION.md read-only
- [ ] session-complete: Validate bd state, extract learnings → kb-* sessions, restore SESSION.md writeable
- [ ] session-abort: Close all bd issues, cleanup, restore SESSION.md writeable
- [ ] subsession-start: Create TDD structure in bd, initialize scratchpad
- [ ] subsession-complete: Archive scratchpad, combine with worklog → learnings.md
- [ ] subsession-abort: Archive scratchpad, mark cancelled in bd
- [ ] work-ready: Context-aware query (session/subsession/all)
- [ ] Scripts prompt for worklog updates when bd issues created/completed

### Knowledge Base
- [ ] Create `_AGENTS/knowledge/bd/` directory
- [ ] Document bd core concepts (issues, dependencies, labels, status)
- [ ] Document bd CLI commands (create, status, dep, ready, list, etc.)
- [ ] Document label conventions for this project
- [ ] Document query patterns for common workflows
- [ ] Include examples of TDD structure in bd
- [ ] Troubleshooting guide (common errors, recovery)

### Documentation
- [ ] SESSIONS-README.md updated with bd integration
- [ ] SESSIONS-REFERENCE.md includes bd examples
- [ ] New doc: BD-INTEGRATION.md (quick reference)
- [ ] Script help text includes bd commands

### Validation
- [ ] Complete test session using bd workflow
- [ ] Multi-subsession dependencies work correctly
- [ ] TDD enforcement verified (blocking structure)
- [ ] work-ready queries return correct context
- [ ] Scratchpads/ directory structure correct (one per subsession)
- [ ] Scratchpad lifecycle works (create, update, archive)
- [ ] Worklog captures bd issue creation/completion with rationale
- [ ] Learning extraction functional (scratchpad + worklog → learnings.md)
- [ ] Session completion creates kb-* sessions from learnings

### Migration
- [ ] Migration guide for existing sessions (optional)
- [ ] Example session in _templates/

## Technical Approach

### Phase 1: Setup & Infrastructure
1. Install bd (`go install github.com/steveyegge/beads/cmd/bd@latest`)
2. Add bd to project dependencies/setup docs
3. Create .gitignore rules:
   ```
   # BD databases (not tracked)
   **/.beads/*.db
   **/.beads/*.db-journal
   **/.beads/*.db-wal
   **/.beads/*.db-shm
   
   # BD exports (tracked)
   !**/.beads/*.jsonl
   ```

### Phase 2: Script Updates
Priority order (dependencies):
1. session-claim (foundation)
2. subsession-start (creates bd structure)
3. work-ready (queries bd)
4. subsession-complete (cleanup)
5. session-complete (validation)
6. abort scripts (error handling)

### Phase 3: Knowledge Base
Create `_AGENTS/knowledge/bd/` with:
1. **overview.md** - What is bd, why we use it
2. **commands.md** - Command reference with examples
3. **labels.md** - Project label conventions
4. **workflows.md** - Common patterns (create subsession, TDD cycle, etc.)
5. **troubleshooting.md** - Common issues and solutions

### Phase 4: Documentation
1. Update SESSIONS-README.md:
   - Add bd to "Directory Structure"
   - Update "Session Contents" to include .beads/ and scratchpads/
   - Document SESSION.md (read-only), worklog.md (WHEN/WHY), scratchpads/ (per-subsession)
   - Add subsession workflow with bd examples
   - Document learning extraction workflow
2. Create BD-INTEGRATION.md:
   - Label conventions
   - Query patterns
   - Common workflows
   - Scratchpad and worklog best practices
3. Update SESSIONS-REFERENCE.md:
   - Add bd command examples
   - Document scratchpad lifecycle
   - Document worklog update patterns
   - Troubleshooting section

### Phase 5: Validation
1. Create test session in drafting/
2. Run through complete lifecycle:
   - Claim session (verify scratchpads/ directory created)
   - Create 3 subsessions with dependencies
   - For each subsession:
     - Verify scratchpad created
     - Update scratchpad with thinking/obstacles
     - Log decisions in worklog.md (with bd issue references)
   - Work through TDD cycles
   - Complete subsessions (verify scratchpad archived, learnings extracted)
   - Complete session (verify kb-* sessions created from learnings)
3. Verify all artifacts correct:
   - SESSION.md unchanged and read-only in active/
   - SESSION.md writeable after session-complete in completed/
   - worklog.md has chronological entries with WHEN/WHY
   - scratchpads/ has one file per subsession (archived)
   - learnings.md exists in KB
   - kb-* session created in drafting/

## Out of Scope

- Migration of existing active sessions (manual if needed)
- BD custom features/extensions
- Integration with external issue trackers
- BD server/multi-project coordination
- Visual UI for bd issues (CLI only)

## Dependencies

- Go installed (for bd)
- jq installed (for JSON parsing in scripts)
- Existing session protocol understood

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| bd not installed | High | Add to setup docs, check in scripts |
| .beads/ merge conflicts | Medium | Clear .gitignore rules, JSONL one-per-line |
| Script complexity | Medium | Keep direct bd usage, minimal wrappers |
| Learning curve | Low | Good docs, comprehensive KB |
| BD database corruption | Low | JSONL backup, bd export/import |

## Success Metrics

- All scripts functional
- Documentation complete and accurate
- KB provides clear bd guidance
- Test session completed successfully
- No regression in existing session workflow
- Improved agent experience (faster queries, clearer dependencies)

## Open Questions

- [ ] Should bd be required or optional enhancement?
  - **Recommendation:** Required for new sessions, document manual fallback
- [ ] How to handle bd not installed?
  - **Recommendation:** Check in session-claim, fail with install instructions
- [ ] BD version pinning?
  - **Recommendation:** Document minimum version, test with latest

## Notes

- Keep backward compatibility where possible
- Document bd commands in script comments
- Preserve existing session artifacts structure
- Test with multiple concurrent sessions
- KB should be comprehensive enough that agents can learn bd from scratch
- SESSION.md is made read-only by session-claim, writeable by session-complete/abort - enforces session contract
- Worklog.md is the "why" companion to bd's "what" - don't duplicate task lists
- Scratchpads/ replace active-plan.md - one per subsession, not session-wide
- Learning extraction is a two-phase process: subsession → learnings.md, session → kb-* sessions

## Subsessions (Proposed)

This session will use bd itself! Bootstrap approach:

1. **Subsession 1: BD Setup & Infrastructure**
   - Install bd
   - Create .gitignore rules
   - Document setup in README
   
2. **Subsession 2: BD Knowledge Base**
   - Create `_AGENTS/knowledge/bd/` structure
   - Write overview.md, commands.md, labels.md
   - Write workflows.md, troubleshooting.md
   - (Blocks subsession 3, 4 - agents need to understand bd first)

3. **Subsession 3: Core Scripts (session-claim, subsession-start)**
   - Update session-claim with bd init
   - Create subsession-start with TDD scaffolding
   - (Blocked by subsession 1, 2)
   - (Blocks subsession 4, 5)

4. **Subsession 4: Query & Complete Scripts**
   - Implement work-ready
   - Implement subsession-complete
   - (Blocked by subsession 3)

5. **Subsession 5: Session Lifecycle Scripts**
   - Update session-complete
   - Implement session-abort, subsession-abort
   - (Blocked by subsession 3)

6. **Subsession 6: Session Protocol Documentation**
   - Update SESSIONS-README.md
   - Create BD-INTEGRATION.md
   - Update SESSIONS-REFERENCE.md
   - (Blocked by subsession 4, 5)

7. **Subsession 7: Validation & Testing**
   - Create test session
   - Run through complete lifecycle
   - Fix issues discovered
   - Validate KB is sufficient for agents
   - (Blocked by subsession 6)

Dependencies in bd:
```
Sub 1 (setup)
  ↓ blocks
Sub 2 (KB) ──────┬─ blocks → Sub 3 (core scripts) ─┬─ blocks → Sub 4 (query/complete)
                 │                                    └─ blocks → Sub 5 (lifecycle)
                 │                                                  ↓ blocks (both)
                 │                                                Sub 6 (docs)
                 │                                                  ↓ blocks
                 └──────────────────────────────────────────────→ Sub 7 (validation)
```

## Estimated Effort

- **Duration:** 2-3 days
- **Complexity:** Medium (new tool integration, but clear boundaries)
- **Agent Capability:** Requires bash scripting, understanding of git worktrees, bd CLI

