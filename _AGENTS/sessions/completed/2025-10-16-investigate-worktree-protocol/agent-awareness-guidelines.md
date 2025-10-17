# Agent Awareness Guidelines for Hub-Spoke Architecture

## Overview

This document provides explicit instructions for AI agents on how to check the `PS1` prompt, environment variables, and other contextual indicators to ensure they are operating within the correct session context in the **hub-and-spoke architecture with shallow clones**. These guidelines serve as the "agent behavior" component that complements the passive restraint mechanisms.

## Core Principle

**Before performing any session-critical operation, agents MUST verify their operational context using available environmental indicators.**

## Hub-Spoke Architecture Context

In the hub-and-spoke model:
- **Main Repo**: Serves as the hub with full Git history
- **Session Clones**: Independent shallow clones in `.sessions/{session-slug}/` directories
- **Standard Git Workflow**: No special worktree concepts, just regular Git operations
- **Clear Boundaries**: Each session is a completely separate repository clone

## Environmental Indicators Hierarchy

Agents should check indicators in this order:

1. **PS1 Prompt** (Most immediately visible)
2. **Environment Variables** (Most reliable)
3. **Current Working Directory** (Foundational)
4. **Git Branch** (Repository state)

## 1. PS1 Prompt Interpretation for Session Clones

### What to Look For

The enhanced PS1 prompt provides real-time visual feedback for session clones:

```
[user@host dir branch-name] [STATUS]$
```

### Prompt Components

| Component | Example | Meaning |
|-----------|---------|---------|
| `branch-name` | `session/2025-10-16-test` | Current Git branch |
| `[SESSION]` | Green indicator | In session clone directory |
| `[MAIN]` | Yellow/Red indicator | NOT in session clone directory |
| Color | Green/Yellow/Red | Overall context status |

### Color Coding for Hub-Spoke

- **Green**: ✅ All good - in session clone on session branch
- **Yellow**: ⚠️ Warning - in session clone but wrong branch
- **Red**: ❌ Danger - not in session clone directory

### Agent Actions Based on Prompt

#### If Prompt Shows `[MAIN]` (Red)
```
Action Required: STOP and fix environment
Steps:
1. Identify the session slug from the task context
2. cd .sessions/{session-slug}
3. source .session-env
4. Verify prompt now shows [SESSION] in green
```

#### If Prompt Shows Wrong Branch (Yellow)
```
Action Required: Switch to correct branch
Steps:
1. Verify the expected session branch name
2. git checkout session/{session-slug}
3. Verify prompt now shows correct branch in green
```

#### If Prompt Shows `[SESSION]` + Correct Branch (Green)
```
Status: SAFE to proceed with session operations
```

### Prompt Verification Command

Agents should mentally execute this check:

```bash
# Pseudo-code for agent reasoning
if prompt_shows("[MAIN]") or not prompt_shows("[WORKTREE]"):
    stop_and_fix_environment()
elif prompt_branch != expected_session_branch:
    switch_to_correct_branch()
else:
    safe_to_proceed()
```

## 2. Environment Variable Checks

### Required Variables

Before any session operation, verify these variables are set:

```bash
echo $SESSION_SLUG
echo $SESSION_BRANCH  
echo $SESSION_DIR
echo $SESSION_ACTIVE
```

### Verification Procedure

```bash
# Agent should conceptually execute:
if [ -z "$SESSION_SLUG" ]; then
    echo "ERROR: Session environment not sourced"
    echo "Run: source _AGENTS/sessions/active/{session-slug}/.session-env"
    exit 1
fi
```

### Agent Decision Tree

```
Is SESSION_SLUG set?
├─ NO → Source session environment
│       └─ source .session-env
│       └─ Verify variables now set
└─ YES → Validate values
         ├─ Does SESSION_BRANCH match expected?
         │  ├─ NO → Fix branch discrepancy
         │  └─ YES → Continue
         └─ Does SESSION_DIR match pwd?
            ├─ NO → Change to SESSION_DIR
            └─ YES → Safe to proceed
```

## 3. Working Directory Awareness

### Critical Rule

**File operations MUST only occur within the session worktree directory.**

### Directory Verification

Before creating or modifying files:

```bash
# Agent reasoning:
current_dir=$(pwd)
if [[ "$current_dir" != *".worktrees/"* ]]; then
    echo "ERROR: Not in worktree directory"
    echo "Current: $current_dir"
    echo "Expected: .worktrees/{session-slug}/"
    # STOP - do not proceed with file operation
fi
```

### Safe File Operations

```bash
# Before writing files, verify:
1. Check: pwd contains ".worktrees/"
2. Check: pwd matches $SESSION_DIR
3. Only then: Proceed with file operation
```

### Example Agent Workflow

```
User: "Create a new file called analysis.md"

Agent Internal Checklist:
1. ✓ Check PS1 prompt → Shows [WORKTREE] in green
2. ✓ Check SESSION_SLUG → Set to "2025-10-16-test"
3. ✓ Check pwd → In .worktrees/2025-10-16-test
4. ✓ Safe to create file

Action: Create analysis.md in current directory
```

## 4. Git Branch Validation

### Before ANY Git Operation

```bash
# Agent must verify:
current_branch=$(git rev-parse --abbrev-ref HEAD)
expected_branch="session/${SESSION_SLUG}"

if [[ "$current_branch" != "$expected_branch" ]]; then
    echo "ERROR: On wrong branch"
    echo "Current: $current_branch"
    echo "Expected: $expected_branch"
    # Offer to switch or abort
fi
```

### Git Operation Safety Matrix

| Operation | Required Check | Action if Check Fails |
|-----------|---------------|----------------------|
| `git add` | In worktree | Stop, cd to worktree |
| `git commit` | On session branch | Stop, checkout branch |
| `git push` | On session branch | Stop, verify branch |
| `git checkout` | N/A | Allowed (for fixing) |

## 5. Session Environment Activation

### When to Source

Source the session environment in these scenarios:

1. **At session start**: Always source before beginning work
2. **After directory change**: If you cd out of worktree and back
3. **If variables are unset**: If $SESSION_SLUG is empty
4. **If prompt shows [MAIN]**: Indicates environment not active

### Sourcing Procedure

```bash
# Agent command sequence:
cd .worktrees/{session-slug}
source ../../_AGENTS/sessions/active/{session-slug}/.session-env

# Verify activation:
- Check SESSION_SLUG is set
- Check prompt shows [WORKTREE]
- Check pwd is in worktree
```

### Re-sourcing Safety

The session environment is designed to prevent harmful double-sourcing:

```bash
# Safe to source multiple times
source .session-env  # First time
source .session-env  # Second time - will detect and skip
```

## 6. Operational Safety Checklist

### Before File Creation

- [ ] PS1 prompt shows `[WORKTREE]` in green
- [ ] `$SESSION_DIR` is set
- [ ] `pwd` is within `.worktrees/`
- [ ] `pwd` matches `$SESSION_DIR`

### Before Git Commit

- [ ] PS1 prompt shows correct branch
- [ ] `$SESSION_BRANCH` is set
- [ ] Current branch matches `$SESSION_BRANCH`
- [ ] In worktree directory

### Before Git Push

- [ ] All "Before Git Commit" checks pass
- [ ] Remote branch is session branch
- [ ] No commits to main/dev

### At Session Start

- [ ] Session claimed (in `active/` directory)
- [ ] Worktree exists
- [ ] Changed to worktree directory
- [ ] Sourced session environment
- [ ] All variables set correctly

## 7. Error Recovery Procedures

### Error: Not in Worktree

```
Symptom: Prompt shows [MAIN]
Cause: Working in main repository directory

Recovery:
1. Identify session slug from context
2. cd .worktrees/{session-slug}
3. source ../../_AGENTS/sessions/active/{session-slug}/.session-env
4. Verify prompt now green with [WORKTREE]
```

### Error: Wrong Branch

```
Symptom: Prompt shows different branch than expected
Cause: On wrong Git branch

Recovery:
1. Check expected branch: echo $SESSION_BRANCH
2. Switch: git checkout $SESSION_BRANCH
3. Verify prompt now shows correct branch
```

### Error: Environment Not Sourced

```
Symptom: $SESSION_SLUG is empty
Cause: Session environment not activated

Recovery:
1. Ensure in worktree: cd .worktrees/{session-slug}
2. Source environment: source ../../_AGENTS/sessions/active/{session-slug}/.session-env
3. Verify: echo $SESSION_SLUG
```

### Error: Files in Wrong Location

```
Symptom: Created files outside worktree
Cause: Worked in wrong directory

Recovery:
1. Identify misplaced files
2. Move to correct location:
   mv /wrong/location/file .worktrees/{session-slug}/
3. Update Git tracking if needed
```

## 8. Agent Self-Validation Commands

Agents should periodically validate their context:

### Quick Status Check

```bash
# Mental execution before operations:
echo "Branch: $(git rev-parse --abbrev-ref HEAD)"
echo "Dir: $(pwd)"
echo "Session: $SESSION_SLUG"
```

### Full Validation

```bash
# Run the built-in validation:
validate_session_vars && validate_session_values
# Or call the session info function:
session_info
```

## 9. Integration with .roo/rules

These agent awareness guidelines complement the `.roo/rules` passive restraints:

**`.roo/rules` provides**: Persistent, always-available protocol rules

**Agent Awareness provides**: Behavioral instructions for interpreting and acting on those rules

### Synergy

```
.roo/rules says: "MUST work in worktree"
Agent Awareness says: "HOW to verify you're in worktree (check prompt/pwd)"

.roo/rules says: "MUST be on session branch"
Agent Awareness says: "HOW to verify branch (check $SESSION_BRANCH)"
```

## 10. Training Scenarios

### Scenario 1: Starting a Session

```
Context: User says "work on session 2025-10-16-test"

Agent Actions:
1. Check if session exists in active/
2. cd .worktrees/2025-10-16-test
3. source ../../_AGENTS/sessions/active/2025-10-16-test/.session-env
4. Verify prompt shows [WORKTREE] and correct branch
5. Ready to proceed

Checkpoints:
✓ Directory changed
✓ Environment sourced
✓ Prompt verified
✓ Variables set
```

### Scenario 2: Creating a File

```
Context: User says "create analysis.md with findings"

Agent Pre-Flight:
1. Check prompt → Green [WORKTREE]
2. Check pwd → In .worktrees/2025-10-16-test
3. Check SESSION_DIR → Matches pwd
4. Safe to create file

Agent Action:
Create analysis.md in current directory

Post-Action:
File created in correct location
```

### Scenario 3: Committing Changes

```
Context: User says "commit the changes"

Agent Pre-Flight:
1. Check prompt → Shows session/2025-10-16-test
2. Check $SESSION_BRANCH → Matches current branch
3. Check pwd → In worktree
4. Safe to commit

Agent Action:
git add .
git commit -m "message"

Post-Action:
Commit on correct branch, in worktree
```

## 11. Anti-Patterns to Avoid

### ❌ Don't: Assume Environment is Correct

```
Bad: Just start creating files
Good: Check prompt and variables first
```

### ❌ Don't: Ignore Prompt Colors

```
Bad: See red prompt, proceed anyway
Good: See red prompt, stop and fix environment
```

### ❌ Don't: Work from Main Directory

```
Bad: cd to repo root and create files
Good: Always work from within worktree
```

### ❌ Don't: Skip Environment Sourcing

```
Bad: "I'll just remember the session name"
Good: Always source .session-env to set variables
```

### ❌ Don't: Commit to Main Branch

```
Bad: Commit to main during a session
Good: Verify on session branch before commit
```

## 12. Success Indicators

An agent is correctly following these guidelines when:

- ✅ Prompt is always green when working
- ✅ All operations occur in worktree
- ✅ All commits are on session branch
- ✅ Variables are always set
- ✅ No files created in wrong locations
- ✅ No protocol violation errors

## 13. Quick Reference Card

```
┌─────────────────────────────────────────────────────────┐
│ AGENT QUICK REFERENCE                                   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ Before ANY operation:                                   │
│  1. Check PS1 → Must show [WORKTREE] in green          │
│  2. Check $SESSION_SLUG → Must be set                  │
│  3. Check pwd → Must be in .worktrees/                 │
│                                                         │
│ If ANY check fails:                                     │
│  1. cd .worktrees/{session-slug}                       │
│  2. source .session-env                                 │
│  3. Re-verify all checks                                │
│                                                         │
│ Red prompt = DANGER - Fix immediately                   │
│ Yellow prompt = WARNING - Verify branch                 │
│ Green prompt = SAFE - Proceed                           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## 14. Implementation Notes

These guidelines should be:

1. **Integrated into agent training**: Core knowledge for all agents
2. **Referenced in `.roo/rules`**: Linked from passive restraints
3. **Included in session docs**: Part of SESSIONS-REFERENCE.md
4. **Validated in testing**: Verify agents follow guidelines
5. **Updated as needed**: Refine based on real-world usage

## Conclusion

By consistently following these agent awareness guidelines, AI agents can effectively:

- Verify their operational context
- Prevent protocol violations
- Recover from errors quickly
- Maintain session isolation
- Work safely within the Agent Sessions Protocol

These guidelines transform passive restraints from static rules into actionable, contextual intelligence that guides agent behavior in real-time.