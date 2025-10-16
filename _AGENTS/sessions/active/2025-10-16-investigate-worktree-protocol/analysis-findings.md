# Worktree Protocol Investigation: Centering on Passive Restraint Strategy

## Executive Summary: The Critical Role of Passive Restraints

This investigation into worktree protocol violations during previous agent sessions reveals a fundamental gap: the absence of **passive restraint mechanisms**. While existing protocol documentation is clear, the agent's consistent failure in execution and decision-making highlights a critical need for always-on, context-aware guidance.

**Passive restraints** (e.g., `.roo/rules`, `.cursorrules`) are configuration files that AI agents automatically consult before taking actions. They act as persistent, project-specific guardrails, embodying "poka-yoke" (error-proofing) principles by making correct behavior easier and incorrect behavior harder. Their absence meant agents relied solely on transient system prompts or reactive validation, leading to:

-   Working outside the designated worktree.
-   Creating files in incorrect locations.
-   Operating on the wrong Git branch.

This document recontextualizes the root causes of protocol violations, emphasizing that **implementing robust passive restraints is the foundational solution** to ensure consistent adherence to the Agent Sessions Protocol.

## Remediation Strategies: First-Line Defenses for Protocol Adherence

To address the identified protocol violations, we must implement robust, non-agent-behavior-dependent remediation strategies as first-line defenses. These mechanisms act as "passive restraints" or "poka-yoke" (error-proofing) to prevent errors before they occur or make them immediately obvious.

### 1. Git Hooks (Pre-commit/Pre-push)

Git hooks are scripts that Git executes before or after events like commit or push. They are ideal for enforcing repository-level policies without relying on agent discretion.

*   **Pre-commit Hook**:
    *   **Purpose**: Prevent commits to non-session branches or from outside the worktree.
    *   **Implementation**: A script that checks `git rev-parse --abbrev-ref HEAD` against the expected session branch and `pwd` against the worktree path. If conditions are not met, the commit is aborted.
    *   **Example**:
        ```bash
        #!/bin/bash
        SESSION_BRANCH_PREFIX="session/"
        CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
        if [[ "$CURRENT_BRANCH" != "$SESSION_BRANCH_PREFIX"* ]]; then
            echo "ERROR: Cannot commit to non-session branch: $CURRENT_BRANCH"
            exit 1
        fi
        # Further checks for worktree path can be added here
        ```

*   **Pre-push Hook**:
    *   **Purpose**: Ensure all changes are pushed to the correct session branch.
    *   **Implementation**: A script that verifies the remote branch matches the local session branch.

### 2. Enhanced Shell Environment (Non-Agent Dependent)

The shell environment itself can provide continuous, passive feedback and enforce rules.

*   **Prompt Modification**:
    *   **Purpose**: Visually remind the agent of the current branch and worktree status.
    *   **Implementation**: Modify the `PS1` (prompt string 1) environment variable to display the current Git branch and an indicator if the current directory is within a worktree. This information is always visible, reducing "PWD Anchoring Loss."
    *   **Example**:
        ```bash
        # In .session-env or similar sourced file
        export PS1='[\u@\h \W $(git rev-parse --abbrev-ref HEAD 2>/dev/null)]$(pwd | grep -q ".worktrees" && echo " [WORKTREE]" || echo " [MAIN]")\$ '
        ```
    *   **Agent Awareness**: Agents must be explicitly instructed to parse and utilize this prompt information as a primary source of truth for their current operational context. They should be aware that if the `[WORKTREE]` indicator is missing or the branch name is incorrect, they need to source their session environment or change directories.

*   **Environment Variable Enforcement**:
    *   **Purpose**: Define critical session parameters that agents must adhere to.
    *   **Implementation**: Set `SESSION_SLUG`, `SESSION_BRANCH`, and `SESSION_DIR` variables upon sourcing the session environment. These variables can then be referenced by other scripts or agent logic.
    *   **Agent Awareness**: Agents should be trained to always check for the presence and correctness of these environment variables before performing session-critical actions. If not set, the agent should be instructed to source the session environment.

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
