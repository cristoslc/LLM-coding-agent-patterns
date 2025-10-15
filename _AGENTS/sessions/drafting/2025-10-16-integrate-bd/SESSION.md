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

## Acceptance Criteria

### Infrastructure
- [ ] bd installed and documented in project setup
- [ ] bd initializes automatically during session-claim
- [ ] .beads/ directory properly ignored/tracked (db vs jsonl)
- [ ] .gitignore updated appropriately

### Scripts Updated
- [ ] session-claim: Initialize bd in worktree
- [ ] session-complete: Validate bd state, create KB session
- [ ] session-abort: Close all bd issues, cleanup
- [ ] subsession-start: Create TDD structure in bd
- [ ] subsession-complete: Archive scratchpad, extract learnings
- [ ] subsession-abort: Archive and mark cancelled
- [ ] work-ready: Context-aware query (session/subsession/all)

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
- [ ] Scratchpad lifecycle works (archive, extract)
- [ ] KB learnings extraction functional

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
   - Update "Session Contents" to include .beads/
   - Add subsession workflow with bd examples
2. Create BD-INTEGRATION.md:
   - Label conventions
   - Query patterns
   - Common workflows
3. Update SESSIONS-REFERENCE.md:
   - Add bd command examples
   - Troubleshooting section

### Phase 5: Validation
1. Create test session in drafting/
2. Run through complete lifecycle:
   - Claim session
   - Create 3 subsessions with dependencies
   - Work through TDD cycles
   - Archive scratchpads
   - Extract learnings
   - Complete session
3. Verify all artifacts correct

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

