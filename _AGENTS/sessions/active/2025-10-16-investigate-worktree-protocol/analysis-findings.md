# Worktree Protocol Investigation: Hub-Spoke Architecture Transition

## Executive Summary: From Worktrees to Hub-Spoke Architecture

This investigation into worktree protocol violations during previous agent sessions revealed fundamental issues with the worktree approach itself. While passive restraint mechanisms are valuable, the core problem is that **Git worktrees are conceptually too complex for reliable agent execution**.

**Critical Discovery**: A **hub-and-spoke architecture with shallow clones** is superior for agent sessions because:

- **Simpler Mental Model**: Standard Git workflow, no special concepts
- **Complete Isolation**: Each session is truly independent
- **Ephemeral by Nature**: Sessions are temporary, shallow clones are temporary
- **Network Independence**: Sessions work offline, only main repo connects to cloud
- **Robust and Reliable**: No fragile dependencies, works across filesystems

This document presents the investigation findings and the architectural decision to **transition from worktrees to a hub-spoke methodology** with shallow clones as the foundation for the Agent Sessions Protocol.

## Investigation Findings: Root Causes and Solutions

### Root Cause Analysis
The investigation identified multiple contributing factors to worktree protocol violations:

1. **Conceptual Complexity**: Worktrees introduce cognitive overhead that agents struggle with
2. **Shared State Confusion**: Worktrees share Git state, making boundaries unclear
3. **Missing Passive Restraints**: Lack of automatic enforcement mechanisms
4. **Decision-Making Failure**: Agents consistently failed to follow explicit worktree instructions
5. **Workflow Friction**: Non-standard Git operations that require special knowledge

### Architectural Decision: Hub-Spoke with Shallow Clones

After comprehensive evaluation of alternatives (reference clones, shallow clones, full clones), the **hub-and-spoke architecture with shallow clones** emerges as the optimal solution:

```
Cloud (GitHub)
    ↓
Main Repo (on-disk, full history)
    ↓
Session Clones (shallow, ephemeral)
```

### Why Hub-Spoke Solves Core Problems

1. **Eliminates Conceptual Complexity**: Standard Git repository workflow
2. **Provides Complete Isolation**: Each session is independent
3. **Aligns with Ephemeral Nature**: Sessions are temporary by definition
4. **Simplifies Agent Understanding**: No special concepts to learn
5. **Maximizes Reliability**: No fragile dependencies or shared state

## Updated Remediation Strategies: Hub-Spoke Implementation

The transition to hub-spoke architecture requires implementing new remediation strategies tailored for shallow clones:

### 1. Git Hooks for Session Clones

Git hooks are adapted for the hub-spoke architecture to enforce session isolation.

*   **Pre-commit Hook**:
    *   **Purpose**: Prevent commits to non-session branches or from outside the session directory.
    *   **Implementation**: A script that checks `git rev-parse --abbrev-ref HEAD` against the expected session branch and `pwd` against the session directory.
    *   **Example**:
        ```bash
        #!/bin/bash
        SESSION_BRANCH_PREFIX="session/"
        CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
        if [[ "$CURRENT_BRANCH" != "$SESSION_BRANCH_PREFIX"* ]]; then
            echo "ERROR: Cannot commit to non-session branch: $CURRENT_BRANCH"
            exit 1
        fi
        # Check we're in session directory
        if [[ ! "$PWD" == *"/.sessions/"* ]]; then
            echo "ERROR: Must commit from within session directory"
            exit 1
        fi
        ```

*   **Pre-push Hook**:
    *   **Purpose**: Ensure pushes go to the correct upstream (main repo) and branch.
    *   **Implementation**: A script that verifies the remote configuration and session branch.

### 2. Enhanced Shell Environment for Session Clones

The shell environment is adapted for session clones with clear visual feedback.

*   **Prompt Modification**:
    *   **Purpose**: Visually remind the agent of the current branch and session status.
    *   **Implementation**: Modify the `PS1` (prompt string 1) environment variable to display the current Git branch and an indicator if the current directory is within a session clone.
    *   **Example**:
        ```bash
        # In .session-env for session clones
        export PS1='[\u@\h \W $(git rev-parse --abbrev-ref HEAD 2>/dev/null)]$(pwd | grep -q ".sessions" && echo " [SESSION]" || echo " [MAIN]")\$ '
        ```
    *   **Agent Awareness**: Agents parse the prompt to verify they're in the correct session directory and on the right branch.

*   **Environment Variable Enforcement**:
    *   **Purpose**: Define critical session parameters for shallow clones.
    *   **Implementation**: Set `SESSION_SLUG`, `SESSION_BRANCH`, and `SESSION_DIR` variables upon sourcing the session environment. These variables reference the session clone directory rather than worktree.
    *   **Agent Awareness**: Agents check these variables to ensure they're operating in the correct session context.

### 3. File System Guardrails

*   **Directory Structure Enforcement**:
    *   **Purpose**: Prevent file creation in incorrect locations.
    *   **Implementation**: While not a "hook," the agent's internal logic can be guided by rules that explicitly state where files *must* be created (e.g., `_AGENTS/sessions/active/{session-slug}/` or `.worktrees/{session-slug}/`).
    *   **Passive Restraint Integration**: A `.roo/rules` file can contain regex patterns or explicit paths for allowed file creation locations.

## Current Protocol Documentation Analysis

The existing documentation (`SESSIONS-README.md` and `SESSIONS-REFERENCE.md`) provides clear instructions for worktree usage:

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

Despite clear documentation, the following protocol points were consistently violated:

1.  **Worktree Creation Timing**: Worktree was not created immediately after claiming the session.
2.  **Working Directory Requirement**: Agent failed to `cd .worktrees/{session-slug}` before starting work.
3.  **Isolation Principle**: Agent did not maintain an isolated working directory, leading to scattered files.
4.  **Concurrent Session Support**: The agent's actions undermined the ability for multiple concurrent sessions.

## Root Cause Analysis: The Absence of Passive Restraints

The primary issue was **Agent Decision-Making Failure**, but this failure was critically exacerbated by the **absence of passive restraint mechanisms**. The protocol documentation is clear, but without persistent, automatic guardrails, the agent consistently failed to follow explicit instructions.

### Specific Decision Points Where Protocol Failed (Exacerbated by Missing Passive Restraints):

1.  **Session Start Decision Point**:
    -   **Should have**: Created worktree immediately.
    -   **Actual**: Worked in main directory, never created worktree initially.
    -   **Passive Restraint Impact**: A `.roo/rules` file could have enforced worktree creation as a prerequisite for any session-related action.

2.  **File Creation Decision Points**:
    -   **Should have**: Created all session files within the worktree directory.
    -   **Actual**: Created files scattered across the repository root.
    -   **Passive Restraint Impact**: Rules could have prevented file creation outside the designated worktree path.

3.  **Working Directory Decision Points**:
    -   **Should have**: Changed to worktree directory before any work.
    -   **Actual**: Stayed in the main repository directory.
    -   **Passive Restraint Impact**: Rules could have validated the current working directory before allowing operations.

4.  **Session Environment Activation**:
    -   **Should have**: Activated session environment from within the worktree.
    -   **Actual**: Attempted to source environment from the wrong location.
    -   **Passive Restraint Impact**: Rules could have enforced environment sourcing as a prerequisite for session activity.

### Contributing Factors (Addressed by Passive Restraints)

1.  **PWD Anchoring Loss**:
    -   Agent lost track of the current working directory.
    -   **Passive Restraint Solution**: Rules can enforce `pwd` validation before operations.

2.  **Missing Validation Steps**:
    -   No verification that worktree exists or files are created in correct locations.
    -   **Passive Restraint Solution**: Rules can define mandatory validation checks for worktree presence, file paths, and branch adherence.

3.  **Protocol Adherence Assumption**:
    -   Agent assumed compliance without verification.
    -   **Passive Restraint Solution**: Rules provide continuous, automatic self-checking mechanisms.

4.  **Complexity of Multi-Directory Operations**:
    -   Agent got confused about which operations happen where.
    -   **Passive Restraint Solution**: Rules simplify decision-making by providing clear, always-available context-specific instructions.

5.  **Agent Operating Outside Session Branch (New Guardrail Issue)**:
    -   Agent inadvertently made changes to the `main` branch.
    -   **Passive Restraint Solution**: Rules can explicitly prohibit operations on non-session branches, acting as a strong preventive measure.


## Conclusion: Prioritizing First-Line Defenses

The core Agent Sessions Protocol is sound, but the failure was in **execution and decision-making** due to a lack of **continuous, automatic enforcement**. The solution lies in the proactive implementation of **first-line defense mechanisms** that do not rely on agent behavior.

By prioritizing remediation strategies like Git hooks, enhanced shell environments, and file system guardrails, we can:

1.  **Embed Protocol Rules**: Make session workflow requirements an inherent part of the agent's operational environment.
2.  **Prevent Violations**: Leverage poka-yoke principles to make incorrect actions impossible or immediately obvious.
3.  **Ensure Consistent Adherence**: Provide always-on guidance that transcends individual session prompts.
4.  **Reduce Cognitive Load**: Simplify decision-making by providing clear, context-specific rules.
5.  **Enable Self-Correction**: Equip agents with the means to validate their own actions against established protocols.

The next phase of this session must prioritize the creation and integration of these first-line defenses to establish a robust, error-proof foundation for agent operations.
