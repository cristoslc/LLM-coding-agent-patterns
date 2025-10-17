# Environment Variable Enforcement Design

## Overview

This document defines the critical session environment variables (`SESSION_SLUG`, `SESSION_BRANCH`, `SESSION_DIR`) and their enforcement mechanisms to ensure agents operate within the correct session context.

## Core Environment Variables

### 1. SESSION_SLUG

**Purpose**: Unique identifier for the current session

**Format**: `YYYY-MM-DD-description`

**Example**: `2025-10-16-investigate-worktree-protocol`

**Usage**:
- Identifies which session is active
- Used in file paths
- Used in branch names
- Referenced by scripts

**Setting**:
```bash
export SESSION_SLUG="2025-10-16-investigate-worktree-protocol"
```

### 2. SESSION_BRANCH

**Purpose**: Git branch name for the current session

**Format**: `session/${SESSION_SLUG}`

**Example**: `session/2025-10-16-investigate-worktree-protocol`

**Usage**:
- Validates commits are on correct branch
- Used by Git hooks
- Displayed in prompts

**Setting**:
```bash
export SESSION_BRANCH="session/${SESSION_SLUG}"
```

### 3. SESSION_DIR

**Purpose**: Absolute path to the session worktree directory

**Format**: `/absolute/path/to/.worktrees/${SESSION_SLUG}`

**Example**: `/home/user/project/.worktrees/2025-10-16-investigate-worktree-protocol`

**Usage**:
- Validates file operations are in correct directory
- Used for relative path calculations
- Referenced by scripts

**Setting**:
```bash
export SESSION_DIR="$(pwd)"  # When sourced from worktree
# Or explicitly:
export SESSION_DIR="/home/user/project/.worktrees/${SESSION_SLUG}"
```

## Additional Supporting Variables

### 4. SESSION_ACTIVE

**Purpose**: Boolean flag indicating if a session environment is active

**Values**: `true` or `false` (or unset)

**Usage**:
- Quick check if session environment is sourced
- Prevents double-sourcing

**Setting**:
```bash
export SESSION_ACTIVE="true"
```

### 5. SESSION_START_TIME

**Purpose**: Timestamp when session was activated

**Format**: Unix timestamp or ISO 8601

**Usage**:
- Track session duration
- Logging and analytics

**Setting**:
```bash
export SESSION_START_TIME=$(date +%s)
# Or ISO format:
export SESSION_START_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
```

### 6. ORIGINAL_PS1

**Purpose**: Preserve original prompt before modification

**Usage**:
- Restore prompt when session ends
- Prevent nested prompt modifications

**Setting**:
```bash
if [ -z "$ORIGINAL_PS1" ]; then
    export ORIGINAL_PS1="$PS1"
fi
```

## Enforcement Mechanisms

### 1. Validation Functions

```bash
# Validate all required session variables are set
validate_session_vars() {
    local missing=()
    
    [ -z "$SESSION_SLUG" ] && missing+=("SESSION_SLUG")
    [ -z "$SESSION_BRANCH" ] && missing+=("SESSION_BRANCH")
    [ -z "$SESSION_DIR" ] && missing+=("SESSION_DIR")
    
    if [ ${#missing[@]} -gt 0 ]; then
        echo "ERROR: Missing required session variables:"
        printf '  - %s\n' "${missing[@]}"
        return 1
    fi
    
    return 0
}

# Validate session variables have correct values
validate_session_values() {
    local errors=0
    
    # Check SESSION_BRANCH matches SESSION_SLUG
    if [[ "$SESSION_BRANCH" != "session/$SESSION_SLUG" ]]; then
        echo "ERROR: SESSION_BRANCH mismatch"
        echo "  Expected: session/$SESSION_SLUG"
        echo "  Actual:   $SESSION_BRANCH"
        errors=$((errors + 1))
    fi
    
    # Check SESSION_DIR exists and is a directory
    if [ ! -d "$SESSION_DIR" ]; then
        echo "ERROR: SESSION_DIR does not exist or is not a directory"
        echo "  Path: $SESSION_DIR"
        errors=$((errors + 1))
    fi
    
    # Check SESSION_DIR is in .worktrees
    if [[ "$SESSION_DIR" != *".worktrees"* ]]; then
        echo "ERROR: SESSION_DIR is not in .worktrees directory"
        echo "  Path: $SESSION_DIR"
        errors=$((errors + 1))
    fi
    
    # Check current directory matches SESSION_DIR
    if [[ "$(pwd)" != "$SESSION_DIR"* ]]; then
        echo "WARNING: Current directory is not within SESSION_DIR"
        echo "  Current: $(pwd)"
        echo "  Expected: $SESSION_DIR"
    fi
    
    return $errors
}
```

### 2. Automatic Validation on Sourcing

```bash
# In .session-env, add automatic validation
if ! validate_session_vars; then
    echo "ERROR: Session environment validation failed"
    return 1
fi

if ! validate_session_values; then
    echo "WARNING: Session environment has incorrect values"
fi
```

### 3. Pre-command Validation (Optional)

For stricter enforcement, validate before critical operations:

```bash
# Wrapper function for git commands
git() {
    # Validate session environment before git operations
    if [ "$1" = "commit" ] || [ "$1" = "push" ]; then
        if ! validate_session_vars &>/dev/null; then
            echo "ERROR: Session environment not properly configured"
            echo "Run: source .session-env"
            return 1
        fi
    fi
    
    command git "$@"
}
```

### 4. Directory Change Monitoring

Monitor directory changes to warn if leaving session directory:

```bash
# Add to .session-env
cd() {
    builtin cd "$@"
    
    # Warn if changing out of session directory
    if [[ "$(pwd)" != "$SESSION_DIR"* ]] && [ -n "$SESSION_DIR" ]; then
        echo "⚠️  Warning: You have left the session directory"
        echo "   Session: $SESSION_DIR"
        echo "   Current: $(pwd)"
    fi
}
```

## Complete .session-env Template

```bash
#!/bin/bash
# Session Environment for ${SESSION_SLUG}
# Source this file when starting work in the session worktree
# Usage: source .session-env

# ============================================================================
# PREVENT DOUBLE-SOURCING
# ============================================================================

if [ "$SESSION_ACTIVE" = "true" ] && [ "$SESSION_SLUG" = "${SESSION_SLUG}" ]; then
    echo "⚠️  Session environment already active for: $SESSION_SLUG"
    return 0
fi

# ============================================================================
# CORE SESSION VARIABLES
# ============================================================================

export SESSION_SLUG="${SESSION_SLUG}"
export SESSION_BRANCH="session/${SESSION_SLUG}"
export SESSION_DIR="$(pwd)"
export SESSION_ACTIVE="true"
export SESSION_START_TIME=$(date +%s)

# ============================================================================
# VALIDATION FUNCTIONS
# ============================================================================

validate_session_vars() {
    local missing=()
    
    [ -z "$SESSION_SLUG" ] && missing+=("SESSION_SLUG")
    [ -z "$SESSION_BRANCH" ] && missing+=("SESSION_BRANCH")
    [ -z "$SESSION_DIR" ] && missing+=("SESSION_DIR")
    
    if [ ${#missing[@]} -gt 0 ]; then
        echo "❌ ERROR: Missing required session variables:"
        printf '  - %s\n' "${missing[@]}"
        return 1
    fi
    
    return 0
}

validate_session_values() {
    local errors=0
    
    # Check SESSION_BRANCH matches SESSION_SLUG
    if [[ "$SESSION_BRANCH" != "session/$SESSION_SLUG" ]]; then
        echo "❌ ERROR: SESSION_BRANCH mismatch"
        echo "  Expected: session/$SESSION_SLUG"
        echo "  Actual:   $SESSION_BRANCH"
        errors=$((errors + 1))
    fi
    
    # Check SESSION_DIR exists
    if [ ! -d "$SESSION_DIR" ]; then
        echo "❌ ERROR: SESSION_DIR does not exist"
        echo "  Path: $SESSION_DIR"
        errors=$((errors + 1))
    fi
    
    # Check SESSION_DIR is in .worktrees
    if [[ "$SESSION_DIR" != *".worktrees"* ]]; then
        echo "❌ ERROR: SESSION_DIR is not in .worktrees"
        echo "  Path: $SESSION_DIR"
        errors=$((errors + 1))
    fi
    
    # Check current branch
    local current_branch=$(git rev-parse --abbrev-ref HEAD 2>/dev/null)
    if [[ "$current_branch" != "$SESSION_BRANCH" ]]; then
        echo "⚠️  WARNING: Not on session branch"
        echo "  Expected: $SESSION_BRANCH"
        echo "  Actual:   $current_branch"
    fi
    
    return $errors
}

# ============================================================================
# RUN INITIAL VALIDATION
# ============================================================================

if ! validate_session_vars; then
    echo ""
    echo "Session environment setup failed!"
    return 1
fi

if ! validate_session_values; then
    echo ""
    echo "⚠️  Session environment has issues (see warnings above)"
fi

# ============================================================================
# ENHANCED PROMPT (from shell-environment-design.md)
# ============================================================================

# Save original PS1
if [ -z "$ORIGINAL_PS1" ]; then
    export ORIGINAL_PS1="$PS1"
fi

# Prompt functions
__git_branch() {
    git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "no-git"
}

__worktree_status() {
    if pwd | grep -q ".worktrees"; then
        echo " [WORKTREE]"
    else
        echo " [MAIN]"
    fi
}

__prompt_color() {
    local current_branch=$(__git_branch)
    local in_worktree=$(pwd | grep -q ".worktrees" && echo "yes" || echo "no")
    
    if [[ "$in_worktree" == "yes" ]] && [[ "$current_branch" == "session/"* ]]; then
        echo "\[\033[0;32m\]"  # Green
    elif [[ "$in_worktree" == "yes" ]]; then
        echo "\[\033[0;33m\]"  # Yellow
    else
        echo "\[\033[0;31m\]"  # Red
    fi
}

__color_reset="\[\033[0m\]"

export PS1='$(__prompt_color)[\u@\h \W $(__git_branch)]$(__worktree_status)'$__color_reset'\$ '

# ============================================================================
# HELPER FUNCTIONS
# ============================================================================

# Deactivate session environment
deactivate_session() {
    echo "Deactivating session: $SESSION_SLUG"
    
    # Restore original prompt
    if [ -n "$ORIGINAL_PS1" ]; then
        export PS1="$ORIGINAL_PS1"
    fi
    
    # Clear session variables
    unset SESSION_SLUG
    unset SESSION_BRANCH
    unset SESSION_DIR
    unset SESSION_ACTIVE
    unset SESSION_START_TIME
    
    # Unset functions
    unset -f validate_session_vars
    unset -f validate_session_values
    unset -f deactivate_session
    unset -f __git_branch
    unset -f __worktree_status
    unset -f __prompt_color
    
    echo "✓ Session deactivated"
}

# Show session info
session_info() {
    echo "╔════════════════════════════════════════════════════════════╗"
    echo "║  Session Information                                        ║"
    echo "╚════════════════════════════════════════════════════════════╝"
    echo ""
    echo "Slug:    $SESSION_SLUG"
    echo "Branch:  $SESSION_BRANCH"
    echo "Dir:     $SESSION_DIR"
    echo "Active:  $SESSION_ACTIVE"
    
    if [ -n "$SESSION_START_TIME" ]; then
        local elapsed=$(($(date +%s) - SESSION_START_TIME))
        echo "Elapsed: ${elapsed}s"
    fi
    
    echo ""
    validate_session_vars && validate_session_values
}

# ============================================================================
# ACTIVATION MESSAGE
# ============================================================================

echo ""
echo "╔════════════════════════════════════════════════════════════╗"
echo "║  ✓ Session Environment Activated                           ║"
echo "╚════════════════════════════════════════════════════════════╝"
echo ""
echo "Session: $SESSION_SLUG"
echo "Branch:  $SESSION_BRANCH"
echo "Dir:     $SESSION_DIR"
echo ""
echo "Available commands:"
echo "  session_info        - Show session information"
echo "  deactivate_session  - Deactivate this session environment"
echo ""
```

## Integration with claim-session Script

The `claim-session` script should generate the `.session-env` file with proper values:

```bash
#!/bin/bash
# claim-session script snippet

SESSION_SLUG="$1"
WORKTREE_DIR=".worktrees/$SESSION_SLUG"

# Create .session-env file
cat > "_AGENTS/sessions/active/$SESSION_SLUG/.session-env" <<EOF
#!/bin/bash
# Generated session environment for $SESSION_SLUG
# Do not edit manually - regenerate with claim-session

export SESSION_SLUG="$SESSION_SLUG"
export SESSION_BRANCH="session/$SESSION_SLUG"
export SESSION_DIR="\$(cd \$(dirname "\${BASH_SOURCE[0]}")/../../../.worktrees/$SESSION_SLUG && pwd)"
export SESSION_ACTIVE="true"
export SESSION_START_TIME=\$(date +%s)

# ... (include all functions and prompt setup)
EOF

chmod +x "_AGENTS/sessions/active/$SESSION_SLUG/.session-env"
```

## Testing Strategy

```bash
#!/bin/bash
# test-env-vars.sh

echo "=== Testing Environment Variable Enforcement ==="

# Test 1: Source environment
source .session-env
if [ "$SESSION_ACTIVE" = "true" ]; then
    echo "✓ Session activated"
else
    echo "✗ Session activation failed"
fi

# Test 2: Validate variables are set
if validate_session_vars; then
    echo "✓ All required variables set"
else
    echo "✗ Missing required variables"
fi

# Test 3: Validate variable values
if validate_session_values; then
    echo "✓ All variable values correct"
else
    echo "✗ Some variable values incorrect"
fi

# Test 4: Deactivate and verify cleanup
deactivate_session
if [ -z "$SESSION_ACTIVE" ]; then
    echo "✓ Session deactivated successfully"
else
    echo "✗ Session deactivation failed"
fi

echo "=== Tests complete ==="
```

## Security Considerations

1. **Path Injection**: SESSION_DIR should be validated to prevent path traversal
2. **Variable Tampering**: Variables should not be modifiable after initial setup
3. **Isolation**: Each session should have independent variable scope

## Future Enhancements

1. **Session Locking**: Prevent multiple sessions with same slug
2. **Session History**: Track previous session activations
3. **Resource Limits**: Enforce limits on session duration or file operations
4. **Audit Logging**: Log all session variable changes