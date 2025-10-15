# Request for Comments (RFC): Structured Subsession Tracking with TDD Integration

**Author(s):** cristos

**Date:** October 16, 2025

**Revisit Date:** TBD — review adoption after initial usage, assess TDD enforcement effectiveness, and evaluate whether manual markdown tracking should remain as a fallback option.

---

### 1. Context / Problem Statement

The current session protocol tracks subsessions using markdown files (`subsessions.md`, `active-plan.md`) with manual task lists. While this approach is simple and git-friendly, it has several limitations:

**Dependency Management:** When subsessions have dependencies on each other, agents must manually parse markdown to determine what work is unblocked. There is no cycle detection, making it possible to create circular dependencies that block progress.

**Status Queries:** Answering "what should I work on next?" requires reading multiple markdown files and mentally tracking completion state. There is no programmatic way to query for ready work across sessions or within complex subsession structures.

**TDD Discipline:** The protocol encourages test-driven development but doesn't enforce it structurally. Agents can skip phases (write code without tests, refactor without validation) because there's no blocking mechanism to ensure the RED → GREEN → REFACTOR cycle is followed.

**Context Switching:** The `active-plan.md` file accumulates tasks across the entire session lifespan. When working on subsession 3, agents must mentally filter out completed work from subsessions 1-2 and future work from subsessions 4-5. This cognitive overhead increases as sessions grow.

**Thinking Externalization:** Agents working on complex problems need space to document their thought process, experiments, and dead ends. Currently this happens informally in worklog entries or not at all, making it harder to extract learnings later.

These limitations could lead to blocked sessions, dependency deadlocks, and inconsistent test coverage.

---

### 2. Proposed Change

We propose integrating the bd issue tracker (https://github.com/cristoslc/llm-beads) to provide structured subsession tracking while preserving the session protocol's git-centric, multi-agent coordination model.

**Core Changes:**

**Subsession Tracking:** Replace markdown task lists in `subsessions.md` with bd issues. Each subsession becomes a bd issue that serves as the parent for its constituent tasks. The tasks within a subsession are created as child bd issues, enabling fine-grained tracking with explicit dependencies, labels, and status updates for both subsessions and their tasks. The bd CLI provides `bd ready` to query unblocked work across subsessions and their tasks, and `bd status` to visualize dependency graphs. Cycle detection is automatic.

**TDD Enforcement:** For code-related subsessions, structure the child tasks as TDD phases (RED, GREEN, REFACTOR, QA), each as a separate child issue of the parent subsession issue. Dependencies between these child issues create blocking relationships: you cannot start GREEN until RED is complete, and so on. This makes TDD a structural requirement rather than a mere discipline, ensuring comprehensive coverage within the subsession's task hierarchy.

**Per-Subsession Scratchpads:** Create `scratchpads/` directory with one markdown file per subsession (`scratchpads/subsession-1-setup.md`). This replaces the session-wide `active-plan.md` with scoped thinking space. When working on subsession 3, you only see subsession 3's context.

**Session File Roles:** Clarify the purpose of each session file:
- `SESSION.md` becomes read-only during active work (made writeable only in drafting/ and completed/ states). It's the session contract - what we're building and why.
- `worklog.md` continues to capture WHEN and WHY decisions were made, complementing bd's WHAT (tasks/issues).
- `scratchpads/subsession-N.md` externalizes HOW agents are thinking through problems.

**Knowledge Extraction:** Combine scratchpad content with worklog entries to generate learnings at subsession completion. This creates a two-phase knowledge capture: subsession → learnings.md, session completion → kb-* merge sessions.

**Implementation:**

Modify session lifecycle scripts:
- `session-claim`: Initialize bd database, create scratchpads/ directory, set SESSION.md to read-only
- `subsession-start`: Create the parent subsession issue in bd, generate child task issues (structured by TDD for code subsessions or as defined otherwise), initialize scratchpad file
- `subsession-complete`: Archive scratchpad, extract learnings from scratchpad+worklog, close parent and child issues
- `work-ready`: Query bd for unblocked work (session-scoped, subsession-scoped, or all)
- `session-complete`: Validate bd state, create kb-* sessions from learnings, restore SESSION.md writeable

The bd database (`.beads/*.db`) is excluded from git, but JSONL exports (`.beads/*.jsonl`) are tracked for merge-friendliness. This gives us both queryability and git-friendly diffs.

The migration does not require changes to existing completed sessions - only new sessions will use bd tracking.

---

### 3. Alternatives Considered

**Status Quo (Markdown Task Lists):** Retaining the current approach minimizes learning curve and keeps the protocol simple. However, it perpetuates the dependency management and query problems that have caused session delays. The lack of TDD enforcement means we'll continue to see code-without-tests incidents.

**GitHub Issues API:** Using GitHub's native issue tracking would provide dependency management without adding new tools. However, it requires network connectivity (problematic for offline work), introduces latency on every query, and couples the protocol to GitHub specifically. Sessions are meant to be git-repository-centric, not forge-specific.

**Custom JSON/YAML Status Files:** We could create a `status.json` file with structured task tracking. This would be git-friendly and queryable. However, it requires building dependency resolution, cycle detection, and query logic from scratch. We'd essentially be reimplementing bd's core features with custom tooling that requires long-term maintenance.

**Separate TDD Script Without bd:** We could add a simple `tdd-phase` script that enforces phase progression without full dependency tracking. This would address TDD enforcement but leave subsession dependency problems unsolved. It also splits the concerns unnecessarily - having two systems (markdown for subsessions, script for TDD) creates cognitive overhead.

**bd Without TDD Enforcement:** We could use bd only for dependency tracking without structuring TDD phases as child issues. This simplifies the initial integration but misses the opportunity to address the code-without-tests problem structurally. Agents would still need to self-enforce TDD discipline.

---

### 4. Impact & Benefits

**For Agents:**
- "What's next?" becomes a single command: `bd ready` surfaces all unblocked work instantly
- No mental parsing of markdown to determine subsession status
- Scratchpads provide focused context - subsession 3 work doesn't compete with subsessions 1-2 notes
- TDD structure makes it clear what phase you're in and what's required before moving forward
- Explicit thinking space (scratchpads) reduces cognitive load and improves learning extraction

**For Session Quality:**
- Cycle detection prevents dependency deadlocks automatically
- TDD enforcement structurally guarantees test coverage (can't merge GREEN without passing RED tests)
- Better knowledge capture from scratchpad + worklog combination
- Clearer session contracts (read-only SESSION.md during active work means criteria don't drift)

**For Protocol Evolution:**
- bd's JSON API enables future tooling (dashboard, metrics, session analytics) without changing the protocol
- JSONL exports provide audit trail and enable post-mortem analysis of session patterns
- Proven tool (bd) means less maintenance burden than custom solutions

**Migration Effort:**
The change requires updates to seven scripts (session-claim, subsession-start, work-ready, subsession-complete, session-complete, session-abort, subsession-abort) and creation of bd knowledge base documentation. Existing completed sessions are unaffected. Active sessions can continue with markdown tracking until completion.

**Learning Curve:**
Agents need to learn bd CLI basics (create, status, dep, ready). The knowledge base will include comprehensive documentation. Initial sessions may take slightly longer as agents familiarize themselves with bd, but queries and dependency management should become more efficient over time.

---

### 5. Success Metrics

**Adoption Metrics:**
- New sessions successfully use bd tracking
- No sessions create undetected circular dependencies
- Agent "what's next" queries execute quickly (subsecond response)

**Quality Metrics:**
- Reduction in code-without-tests incidents (TDD structure enforcement)
- Session completion time remains reasonable (accounting for learning curve)
- Knowledge extraction improves (scratchpad + worklog combination)

**Experience Metrics:**
- Agents report clearer understanding of ready work
- No sessions blocked due to bd tool issues (corruption, performance, bugs)
- Scratchpads actively used for thinking externalization

**Technical Metrics:**
- All seven lifecycle scripts functional and tested
- bd knowledge base documentation complete (5 core documents)
- Test session completed successfully with multiple subsessions and TDD cycles

Metrics will be reviewed after initial adoption period to assess effectiveness.

---

### 6. Risks & Mitigations

**Risk:** bd not installed in agent environment creates immediate session-claim failure.
**Mitigation:** Update setup documentation to include bd installation. Add pre-flight check in session-claim that fails gracefully with installation instructions if bd is missing.

**Risk:** .beads/ directory merge conflicts when multiple agents work on same session (though rare with current protocol).
**Mitigation:** Rely on JSONL exports for merge-friendliness. Document conflict resolution: prefer "ours" for .beads/ directory, use bd export/import to rebuild from JSONL.

**Risk:** bd database corruption could lose subsession state.
**Mitigation:** JSONL exports (tracked in git) serve as backup. Document recovery procedure: `bd import sessions/active/{session}/.beads/export.jsonl`. Test recovery procedure in validation phase.

**Risk:** Learning curve delays adoption.
**Mitigation:** Create comprehensive bd knowledge base before rollout. Provide template session in `_templates/` showing complete bd workflow.

**Risk:** Script complexity increases, making maintenance harder.
**Mitigation:** Keep scripts as thin wrappers around direct bd commands. Document bd command usage in script comments. Avoid abstractions that hide bd's API.

**Risk:** Agents bypass bd tracking and use manual markdown anyway.
**Mitigation:** Scripts fail if bd database doesn't exist. Remove `active-plan.md` from session template. Document scratchpads as replacement. Make bd usage the path of least resistance.

---

### 7. Open Questions / Feedback Requested

**Optional vs Required:** Should bd be required for all new sessions, or offered as an opt-in enhancement? Current recommendation is required (with documented manual fallback if bd unavailable), but this impacts agents working in constrained environments.

**Scratchpad Lifecycle:** Should scratchpads be archived (kept in session directory) or deleted at subsession completion? Archival preserves full context but increases repository size. Current recommendation is archive for knowledge extraction value.

**TDD Granularity:** Should TDD phases (RED/GREEN/REFACTOR/QA) be mandatory structure, or offered as a pattern? Making them mandatory enforces discipline but reduces flexibility for non-TDD-appropriate work (documentation, configuration). Current recommendation is mandatory for code subsessions, optional for non-code subsessions (labeled appropriately).

**BD Version Pinning:** Should we pin to a specific bd version or document minimum version requirements? Pinning increases stability but requires update coordination. Minimum version provides flexibility but risks feature drift.

**Backward Compatibility:** Should we support sessions using both markdown and bd tracking during transition? This would ease migration but increases script complexity. Current recommendation is clean cutover for new sessions.

---

### 8. Next Steps / Decision Process

1. **Review and feedback**
   - Circulate RFC for review
   - Address open questions
   - Gather concerns about learning curve or tooling complexity

2. **Incorporate revisions**
   - Address feedback in updated RFC or session documentation
   - Finalize decisions on open questions
   - Confirm script update approach

3. **Implement Phase 1: Infrastructure**
   - Install bd and create setup documentation
   - Create .gitignore rules for .beads/ directory
   - Establish bd knowledge base structure

4. **Implement Phase 2: Scripts**
   - Update all seven lifecycle scripts
   - Create work-ready query script
   - Add bd usage documentation to script help text

5. **Validation & Testing**
   - Complete test session using full bd workflow
   - Verify TDD enforcement, dependency tracking, scratchpads
   - Confirm knowledge extraction produces quality learnings

6. **Documentation & Rollout**
   - Update SESSIONS-README.md and SESSIONS-REFERENCE.md
   - Create BD-INTEGRATION.md quick reference

7. **Retrospective**
   - Review success metrics after initial usage
   - Gather feedback
   - Decide on any protocol adjustments

**Decision Criteria:**
- All success metrics from validation phase met
- bd knowledge base complete
- No critical concerns unresolved
- Test session completed with all acceptance criteria satisfied

---

### Appendix

**Related Documents:**
- [SESSIONS-README.md](../SESSIONS-README.md) - Current session protocol overview
- [SESSIONS-REFERENCE.md](../SESSIONS-REFERENCE.md) - Current detailed implementation
- [2025-10-16-integrate-bd/SESSION.md](SESSION.md) - Full implementation specification

**Tool References:**
- bd issue tracker: https://github.com/cristoslc/llm-beads
- bd core concepts: dependency-aware task tracking, cycle detection, JSONL export
- bd CLI commands: create, status, dep, ready, list, export, import

**Example bd Workflow:**
```bash
# Start subsession with TDD structure
./scripts/subsession-start "Setup authentication"
# Creates:
#   bd issue: subsession-1-setup-authentication (parent)
#   bd issue: RED-write-failing-tests (child, unblocked)
#   bd issue: GREEN-implement-code (child, blocked by RED)
#   bd issue: REFACTOR-clean-code (child, blocked by GREEN)
#   bd issue: QA-validate (child, blocked by REFACTOR)

# Query what's ready
bd ready
# Output: RED-write-failing-tests

# Work on RED phase, update scratchpad
vim scratchpads/subsession-1-setup-authentication.md
# Document: "Need to test JWT generation, token expiration, invalid signatures"

# Complete RED phase
bd close RED-write-failing-tests
bd ready
# Output: GREEN-implement-code (now unblocked)
```

**Comparison Matrix:**

| Aspect | Current (Markdown) | Proposed (bd+TDD) |
|--------|-------------------|-------------------|
| Dependency tracking | Manual parsing | Automatic, queryable |
| Cycle detection | None | Built-in |
| "What's next?" query | Read multiple files | `bd ready` command |
| TDD enforcement | Discipline-based | Structural blocking |
| Context scoping | Session-wide (active-plan.md) | Per-subsession (scratchpads/) |
| Knowledge extraction | Manual worklog review | Scratchpad + worklog combination |
| Merge conflicts | Rare, easy to resolve | JSONL-based, documented recovery |
| Learning curve | Minimal | Medium (bd CLI + concepts) |
| Tooling dependency | None (markdown only) | Requires bd installed (Go) |
| Offline capability | Full | Full (bd is local-first) |

**Glossary:**
- **bd**: "Beads" - dependency-aware issue tracker, local-first, git-friendly
- **JSONL**: JSON Lines format - one JSON object per line, merge-friendly
- **TDD Phases**: RED (write failing tests), GREEN (make tests pass), REFACTOR (improve code), QA (validate quality)
- **Scratchpad**: Per-subsession markdown file for externalizing agent thinking
- **KB merge session**: Dedicated session (kb-* prefix) for integrating learnings into canonical knowledge base

