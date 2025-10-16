# Worktree Protocol Investigation: Centering on Passive Restraint Strategy

## Executive Summary: The Critical Role of Passive Restraints

This investigation into worktree protocol violations during previous agent sessions reveals a fundamental gap: the absence of **passive restraint mechanisms**. While existing protocol documentation is clear, the agent's consistent failure in execution and decision-making highlights a critical need for always-on, context-aware guidance.

**Passive restraints** (e.g., `.roo/rules`, `.cursorrules`) are configuration files that AI agents automatically consult before taking actions. They act as persistent, project-specific guardrails, embodying "poka-yoke" (error-proofing) principles by making correct behavior easier and incorrect behavior harder. Their absence meant agents relied solely on transient system prompts or reactive validation, leading to:

-   Working outside the designated worktree.
-   Creating files in incorrect locations.
-   Operating on the wrong Git branch.

This document recontextualizes the root causes of protocol violations, emphasizing that **implementing robust passive restraints is the foundational solution** to ensure consistent adherence to the Agent Sessions Protocol.

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

## Documentation Gaps Identified (Addressed by Passive Restraints)

The identified documentation "gaps" are not necessarily a lack of *information*, but a lack of *enforcement*. Passive restraints directly address these by embedding the "needed" validations and checks into the agent's operational environment.

1.  **Missing: Explicit Worktree Validation**:
    -   **Needed**: "Verify you're in worktree: `pwd | grep -q '.worktrees/' || echo 'ERROR: Not in worktree directory'"`
    -   **Passive Restraint Integration**: This check can be a rule that automatically runs before file operations.

2.  **Missing: File Location Validation**:
    -   **Needed**: "Check file location: `pwd | grep -q '.worktrees/' || echo 'ERROR: Creating file outside worktree'"`
    -   **Passive Restraint Integration**: A rule can prevent file creation outside the worktree.

3.  **Missing: Protocol Compliance Checks**:
    -   **Needed**: Regular protocol compliance validation steps.
    -   **Passive Restraint Integration**: Rules define these as automatic, continuous checks.

4.  **Missing: Error Recovery Procedures**:
    -   **Needed**: "If you realize you're working outside worktree: [recovery steps]"
    -   **Passive Restraint Integration**: Rules can include explicit recovery instructions for common violations.

5.  **Missing: Active Session Branch Validation**:
    -   **Needed**: "Verify current branch: `git rev-parse --abbrev-ref HEAD | grep -q 'session/$SESSION_SLUG' || echo 'ERROR: Not in session branch'"`
    -   **Passive Restraint Integration**: A rule can enforce this check before any Git commit/push.

## Strategies for Helping the Agent Adhere to the Session Branch (Re-framed by Passive Restraints)

The previously identified strategies are all mechanisms that can be *implemented through* or *reinforced by* passive restraints.

1.  **Pre-command Hooks/Aliases**: Can be defined and enforced by passive restraint rules.
2.  **Environment Variable Enforcement**: Passive restraints can ensure correct environment variable setup and usage.
3.  **Enhanced Shell Prompt**: Passive restraints can guide the agent to configure a prompt that provides critical context.
4.  **Automated Context Switching**: Passive restraints can define the conditions and actions for automatic context switching.
5.  **Pre-commit/Pre-push Hooks**: Passive restraints can specify and even generate these Git hooks.
6.  **Clearer Documentation and Training**: Passive restraints serve as a living, executable form of this documentation.

## Conclusion: Passive Restraints as the Foundational Solution

The core Agent Sessions Protocol is sound, but the failure was in **execution and decision-making** due to a lack of **continuous, automatic enforcement**. The solution is not merely more documentation or reactive validation, but the proactive implementation of **passive restraint mechanisms**.

By centering our strategy on passive restraints, we can:

1.  **Embed Protocol Rules**: Make session workflow requirements an inherent part of the agent's operational environment.
2.  **Prevent Violations**: Leverage poka-yoke principles to make incorrect actions impossible or immediately obvious.
3.  **Ensure Consistent Adherence**: Provide always-on guidance that transcends individual session prompts.
4.  **Reduce Cognitive Load**: Simplify decision-making by providing clear, context-specific rules.
5.  **Enable Self-Correction**: Equip agents with the means to validate their own actions against established protocols.

The next phase of this session must prioritize the creation and integration of passive restraint files (e.g., `.roo/rules`) to establish a robust, error-proof foundation for agent operations.