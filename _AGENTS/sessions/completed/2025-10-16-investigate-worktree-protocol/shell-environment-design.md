# Shell Environment Enhancement Design

## Overview

This document defines how to enhance the shell environment to provide continuous visual feedback about the current Git branch and worktree status, integrated with the session environment activation process.

## Objective

Modify the `PS1` (prompt string 1) environment variable to display:
1. Current Git branch name
2. Worktree status indicator ([WORKTREE] or [MAIN])
3. Session context information

This information should be:
- Always visible in the terminal
- Automatically configured when sourcing the session environment
- Compatible with common shells (bash, zsh)

## PS1 Prompt Design

### Basic Components

```bash
# Format: [user@host dir branch] [STATUS]$
# Example: [user@host worktree session/2025-10-16-test] [WORKTREE]$
```

### Implementation

#### For Bash

```bash
# Enhanced PS1 with Git branch and worktree status
# To be added to .session-env file

# Function to get current git branch
__git_branch() {
    git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "no-git"
}

# Function to check if in worktree
__worktree_status() {
    if pwd | grep -q ".worktrees"; then
        echo " [WORKTREE]"
    else
        echo " [MAIN]"
    fi
}

# Color codes for visual distinction
__prompt_color() {
    local status=$(__worktree_status)
    if [[ "$status" == *"WORKTREE"* ]]; then
        echo "\[\033[0;32m\]"  # Green for worktree
    else
        echo "\[\033[0;33m\]"  # Yellow for main
    fi
}

# Reset color
__color_reset="\[\033[0m\]"

# Build the PS1
export PS1='[\u@\h \W $(__git_branch)]$(__worktree_status)\$ '

# Alternative with color
export PS1='$(__prompt_color)[\u@\h \W $(__git_branch)]$(__worktree_status)'$__color_reset'\$ '
```

#### For Zsh

```zsh
# Enhanced prompt with Git branch and worktree status
# To be added to .session-env file

# Enable parameter expansion in prompts
setopt PROMPT_SUBST

# Function to get current git branch
__git_branch() {
    git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "no-git"
}

# Function to check if in worktree
__worktree_status() {
    if pwd | grep -q ".worktrees"; then
        echo " [WORKTREE]"
    else
        echo " [MAIN]"
    fi
}

# Build the PS1 (zsh uses PROMPT instead of PS1)
export PROMPT='[%n@%m %1~ $(__git_branch)]$(__worktree_status)%# '

# Alternative with color
export PROMPT='%F{green}[%n@%m %1~ $(__git_branch)]$(__worktree_status)%f%# '
```

## Integration with Session Environment

### Updated .session-env Template

```bash
#!/bin/bash
# Session Environment for ${SESSION_SLUG}
# This file should be sourced when starting work in the session worktree

# ============================================================================
# SESSION VARIABLES
# ============================================================================

export SESSION_SLUG="${SESSION_SLUG}"
export SESSION_DIR="$PWD"
export SESSION_BRANCH="session/${SESSION_SLUG}"

# ============================================================================
# GIT IDENTITY (Optional)
# ============================================================================

export GIT_AUTHOR_NAME="Agent-${SESSION_SLUG}"
export GIT_AUTHOR_EMAIL="agent-${SESSION_SLUG}@sessions.local"
export GIT_COMMITTER_NAME="$GIT_AUTHOR_NAME"
export GIT_COMMITTER_EMAIL="$GIT_AUTHOR_EMAIL"

# ============================================================================
# ENHANCED PROMPT
# ============================================================================

# Save original PS1 if not already saved
if [ -z "$ORIGINAL_PS1" ]; then
    export ORIGINAL_PS1="$PS1"
fi

# Function to get current git branch
__git_branch() {
    git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "no-git"
}

# Function to check if in worktree
__worktree_status() {
    if pwd | grep -q ".worktrees"; then
        echo " [WORKTREE]"
    else
        echo " [MAIN]"
    fi
}

# Function to get prompt color based on context
__prompt_color() {
    local current_branch=$(__git_branch)
    local in_worktree=$(pwd | grep -q ".worktrees" && echo "yes" || echo "no")
    
    # Green if in worktree on session branch
    if [[ "$in_worktree" == "yes" ]] && [[ "$current_branch" == "session/"* ]]; then
        echo "\[\033[0;32m\]"  # Green - all good
    # Yellow if in worktree but wrong branch
    elif [[ "$in_worktree" == "yes" ]]; then
        echo "\[\033[0;33m\]"  # Yellow - warning
    # Red if not in worktree
    else
        echo "\[\033[0;31m\]"  # Red - danger
    fi
}

# Color reset
__color_reset="\[\033[0m\]"

# Set enhanced PS1
export PS1='$(__prompt_color)[\u@\h \W $(__git_branch)]$(__worktree_status)'$__color_reset'\$ '

# ============================================================================
# VALIDATION HELPERS
# ============================================================================

# Function to validate session environment
validate_session_env() {
    local errors=0
    
    echo "=== Session Environment Validation ==="
    
    # Check if we're on the correct branch
    local current_branch=$(__git_branch)
    if [[ "$current_branch" != "$SESSION_BRANCH" ]]; then
        echo "❌ Wrong branch: $current_branch (expected: $SESSION_BRANCH)"
        errors=$((errors + 1))
    else
        echo "✓ Correct branch: $current_branch"
    fi
    
    # Check if we're in the worktree
    if ! pwd | grep -q ".worktrees"; then
        echo "❌ Not in worktree directory"
        errors=$((errors + 1))
    else
        echo "✓ In worktree directory"
    fi
    
    # Check if SESSION_SLUG is set
    if [ -z "$SESSION_SLUG" ]; then
        echo "❌ SESSION_SLUG not set"
        errors=$((errors + 1))
    else
        echo "✓ SESSION_SLUG set: $SESSION_SLUG"
    fi
    
    if [ $errors -eq 0 ]; then
        echo "=== ✓ All checks passed ==="
        return 0
    else
        echo "=== ❌ $errors error(s) found ==="
        return 1
    fi
}

# Function to restore original prompt
restore_prompt() {
    if [ -n "$ORIGINAL_PS1" ]; then
        export PS1="$ORIGINAL_PS1"
        echo "✓ Original prompt restored"
    fi
}

# ============================================================================
# SESSION ACTIVATION MESSAGE
# ============================================================================

echo ""
echo "╔════════════════════════════════════════════════════════════╗"
echo "║  Session Environment Activated                              ║"
echo "╚════════════════════════════════════════════════════════════╝"
echo ""
echo "Session: $SESSION_SLUG"
echo "Branch:  $SESSION_BRANCH"
echo "Dir:     $SESSION_DIR"
echo ""
echo "Your prompt now shows:"
echo "  - Current Git branch"
echo "  - Worktree status indicator"
echo "  - Color coding (green=good, yellow=warning, red=danger)"
echo ""
echo "Helpful commands:"
echo "  validate_session_env  - Check session environment status"
echo "  restore_prompt        - Restore original prompt"
echo ""
```

## Shell Detection and Compatibility

To handle both bash and zsh:

```bash
# Detect shell type
if [ -n "$BASH_VERSION" ]; then
    # Bash-specific prompt
    export PS1='$(__prompt_color)[\u@\h \W $(__git_branch)]$(__worktree_status)'$__color_reset'\$ '
elif [ -n "$ZSH_VERSION" ]; then
    # Zsh-specific prompt
    setopt PROMPT_SUBST
    export PROMPT='%F{green}[%n@%m %1~ $(__git_branch)]$(__worktree_status)%f%# '
else
    # Fallback for other shells
    export PS1='[\u@\h \W] (\$(__git_branch))\$ '
fi
```

## Testing Strategy

### Test Script

```bash
#!/bin/bash
# test-prompt.sh - Test enhanced prompt functionality

echo "=== Testing Enhanced Prompt ==="

# Test 1: Source session environment
echo "Test 1: Sourcing session environment..."
source .session-env
if [[ "$PS1" == *"__git_branch"* ]]; then
    echo "✓ Prompt includes git branch function"
else
    echo "✗ Prompt missing git branch function"
fi

# Test 2: Check worktree detection
echo "Test 2: Testing worktree detection..."
cd /tmp
if [[ "$(__worktree_status)" == *"MAIN"* ]]; then
    echo "✓ Correctly detects non-worktree directory"
else
    echo "✗ Failed to detect non-worktree directory"
fi

# Test 3: Check git branch function
echo "Test 3: Testing git branch function..."
cd "$(git rev-parse --show-toplevel)"
if [[ -n "$(__git_branch)" ]]; then
    echo "✓ Git branch function works"
else
    echo "✗ Git branch function failed"
fi

# Test 4: Validate prompt colors
echo "Test 4: Testing prompt color function..."
if [[ -n "$(__prompt_color)" ]]; then
    echo "✓ Prompt color function works"
else
    echo "✗ Prompt color function failed"
fi

echo "=== Tests complete ==="
```

## Visual Examples

### Example Prompts

#### In worktree, correct branch (GREEN)
```
[user@host worktree session/2025-10-16-test] [WORKTREE]$
```

#### In worktree, wrong branch (YELLOW)
```
[user@host worktree main] [WORKTREE]$
```

#### Outside worktree (RED)
```
[user@host repo main] [MAIN]$
```

## Agent Awareness Integration

The enhanced prompt provides visual cues that agents should be trained to recognize:

1. **Branch Verification**: Agents should check that the branch shown in the prompt matches the expected session branch
2. **Worktree Status**: Agents should verify the `[WORKTREE]` indicator is present before performing session work
3. **Color Coding**: Agents should be aware of color meanings and take corrective action when seeing yellow/red

This will be detailed in the Agent Awareness Guidelines (Item 8).

## Rollback Procedure

To restore the original prompt:

```bash
# Call the restore function
restore_prompt

# Or manually
export PS1="$ORIGINAL_PS1"
```

## Integration Points

1. **Session Claiming**: The `claim-session` script should create the `.session-env` file with the enhanced prompt
2. **Session Completion**: The `complete-session` script should restore the original prompt
3. **Documentation**: Update `SESSIONS-README.md` to explain prompt indicators

## Future Enhancements

1. **Status Icons**: Add icons for different states (✓, ⚠, ✗)
2. **Customization**: Allow users to configure prompt format
3. **Additional Context**: Show number of uncommitted changes
4. **Session Timer**: Display time elapsed in session