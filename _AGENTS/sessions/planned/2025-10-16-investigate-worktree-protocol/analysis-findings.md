# Worktree Protocol Investigation: Findings and Analysis

## Current Protocol Documentation Analysis

### What the Documentation Actually Says

After reviewing both SESSIONS-README.md and SESSIONS-REFERENCE.md, the worktree protocol is actually **very clearly documented**:

#### From SESSIONS-README.md (Lines 107-117):
```bash
# 4. Create worktree with session branch (outside sessions/)
git worktree add -b session/2025-10-14-feature-x \
  .worktrees/2025-10-14-feature-x \
  HEAD

# 5. Activate session and start work
cd .worktrees/2025-10-14-feature-x
source ../../sessions/active/2025-10-14-feature-x/.session-env

# Now working in isolated worktree!
```

#### From SESSIONS-REFERENCE.md (Lines 434-476):
**Section: "Git Worktrees Setup"** - Dedicated 3-page section explaining:
- How worktrees work
- Creating session worktrees
- Multiple concurrent sessions
- Cleanup procedures
- Benefits and limitations

#### From SESSIONS-REFERENCE.md (Lines 264-270):
```bash
# Activate session environment (in worktree)
cd .worktrees/2025-10-14-auth-system
source ../../sessions/active/2025-10-14-auth-system/.session-env
```

### Key Protocol Points That Were Violated

1. **Worktree Creation Timing**: Protocol clearly states worktree should be created **immediately after claiming** (Line 108 in README, Line 45 in REFERENCE)

2. **Working Directory Requirement**: Protocol explicitly states to **"cd .worktrees/{session-slug}"** before activating session environment

3. **Isolation Principle**: Documentation emphasizes worktrees provide "isolated working directories" and "namespace isolation"

4. **Concurrent Session Support**: Protocol designed for "multiple sessions work concurrently" using worktrees

## Root Cause Analysis: Why Protocol Was Violated

### Primary Issue: **Agent Decision-Making Failure**
The protocol documentation is **crystal clear** - the failure was in **execution/decision-making**, not documentation clarity. The agent consistently failed to follow explicit instructions regarding worktree usage.

### Specific Decision Points Where Protocol Failed:

1.  **Session Start Decision Point**:
    -   **Should have**: Created worktree immediately when session started.
    -   **Actually did**: Worked in main directory, never created worktree initially.

2.  **File Creation Decision Points**:
    -   **Should have**: Created all session files within worktree directory.
    -   **Actually did**: Created files scattered across repository root.

3.  **Working Directory Decision Points**:
    -   **Should have**: Changed to worktree directory before any work.
    -   **Actually did**: Stayed in main repository directory.

4.  **Session Environment Activation**:
    -   **Should have**: Activated session environment from within worktree.
    -   **Actually did**: Attempted to source environment from wrong location.

## Contributing Factors

### 1. **PWD Anchoring Loss**
-   Agent lost track of current working directory.
-   No validation of current directory before operations.
-   No checks to ensure working in correct location.

### 2. **Missing Validation Steps**
-   No verification that worktree exists before proceeding.
-   No checks that files are being created in correct locations.
-   No confirmation of proper working directory.

### 3. **Protocol Adherence Assumption**
-   Agent assumed it was following protocol without verification.
-   No self-checking mechanisms in place.
-   Lack of protocol compliance validation.

### 4. **Complexity of Multi-Directory Operations**
-   Session requires coordination between multiple directories:
    -   Main repo (for session metadata)
    -   Worktree (for actual work)
    -   Sessions directory (for session files)
-   Agent got confused about which operations happen where.

### 5. **Agent Operating Outside Session Branch (New Guardrail Issue)**
-   During testing, the agent inadvertently made changes to the `main` branch, violating the "ONLY WORK IN THE SESSION BRANCH" constraint. This indicates a critical need for a mechanism to ensure the agent is always operating within the correct session branch while an active session is in progress.

### 6. **Missing Passive Restraint Mechanisms (Critical Discovery)**
-   The repository lacks passive constraint files (e.g., `.roo/rules`, `.cursorrules`) that would provide continuous guidance to AI agents.
-   **Passive restraints** are rule files that agents automatically consult before taking actions, similar to how `.editorconfig` enforces coding standards passively.
-   Without these passive restraints, agents rely solely on:
    -   System prompts (which can be forgotten or overridden)
    -   Active validation (which must be explicitly invoked)
    -   User intervention (reactive rather than preventive)
-   This absence likely contributed to protocol violations, as there was no persistent, always-available reference for correct behavior.
-   Passive restraints align with **poka-yoke principles**: they make correct behavior easier and incorrect behavior harder by providing constant guidance.

## Documentation Gaps Identified

While the protocol is documented, these **enhancements** could prevent future violations:

### 1. **Missing: Explicit Worktree Validation**
Current docs show **how** to use worktrees but lack **validation** that worktree is being used correctly.

**Needed**: "Verify you're in worktree: `pwd | grep -q '.worktrees/' || echo 'ERROR: Not in worktree directory'"`

### 2. **Missing: File Location Validation**
No guidance on ensuring files are created in correct locations.

**Needed**: "Check file location: `pwd | grep -q '.worktrees/' || echo 'ERROR: Creating file outside worktree'"`

### 3. **Missing: Protocol Compliance Checks**
No self-validation mechanisms mentioned.

**Needed**: Regular protocol compliance validation steps.

### 4. **Missing: Error Recovery Procedures**
No guidance on what to do if you realize you're not following protocol.

**Needed**: "If you realize you're working outside worktree: [recovery steps]"

### 5. **Missing: Active Session Branch Validation**
No explicit check to ensure the agent is operating within the correct session branch. While "Namespace Isolation" is mentioned, the documentation lacks explicit instructions or mechanisms to *enforce* that an agent remains on its session branch.

**Needed**: "Verify current branch: `git rev-parse --abbrev-ref HEAD | grep -q 'session/$SESSION_SLUG' || echo 'ERROR: Not in session branch'"`

## Strategies for Helping the Agent Adhere to the Session Branch

To address the newly identified guardrail issue and reinforce adherence to the session branch, the following strategies should be considered:

1.  **Pre-command Hooks/Aliases**: Implement shell functions or aliases that wrap common git commands (`git commit`, `git add`, `git push`) and automatically perform a `validate_in_session_branch` check before execution. If the check fails, the command should be aborted with a clear error message.
2.  **Environment Variable Enforcement**: Leverage session-specific environment variables (e.g., `SESSION_BRANCH`) to dynamically configure git to only operate on the designated branch, or to warn/error if an attempt is made to switch branches.
3.  **Enhanced Shell Prompt**: Modify the shell prompt (`PS1`) to prominently display the current branch and a clear indicator if it does not match the active session branch.
4.  **Automated Context Switching**: Explore mechanisms that automatically switch the agent's working directory and git context to the correct worktree/branch upon session activation, and revert upon deactivation.
5.  **Pre-commit/Pre-push Hooks**: Implement client-side git hooks that prevent commits or pushes from occurring if the agent is not on the correct session branch.
6.  **Clearer Documentation and Training**: While the protocol is clear, emphasize the "ONLY WORK IN THE SESSION BRANCH" rule with dedicated sections, examples, and warnings in `SESSIONS-README.md` and `SESSIONS-REFERENCE.md`.

## Conclusion

**The protocol documentation is actually excellent and comprehensive.** The failure was in **execution and decision-making**, not documentation clarity. The solution is to:

1.  **Add validation mechanisms** to prevent protocol violations.
2.  **Enhance documentation** with explicit validation steps.
3.  **Create safeguards** that make protocol violations impossible or easily detectable.
4.  **Implement recovery procedures** for when violations are detected.
5.  **Specifically enforce adherence to the session branch** through technical and procedural safeguards.

The core protocol is sound - we just need better **enforcement and validation** mechanisms, with a particular focus on ensuring the agent operates exclusively within its assigned session branch.