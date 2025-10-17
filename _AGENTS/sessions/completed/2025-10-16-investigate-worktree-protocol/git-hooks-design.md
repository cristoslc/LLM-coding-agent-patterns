# Git Hooks Design for Agent Sessions Protocol

## Overview

This document defines the design for Git hooks that enforce session branch adherence and worktree usage, along with a safe installation method that integrates with existing hooks.

## Hook 1: Pre-commit Hook

### Purpose
Prevent commits to non-session branches and ensure commits only occur from within worktrees.

### Implementation

```bash
#!/bin/bash
# Pre-commit hook for Agent Sessions Protocol
# This hook enforces session branch and worktree requirements

# Check if we're on a session branch
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null)
SESSION_BRANCH_PREFIX="session/"

if [[ "$CURRENT_BRANCH" != "$SESSION_BRANCH_PREFIX"* ]]; then
    echo "ERROR: Cannot commit to non-session branch: $CURRENT_BRANCH"
    echo "You must be on a session branch (starting with 'session/')"
    echo "Current branch: $CURRENT_BRANCH"
    exit 1
fi

# Check if we're in a worktree
CURRENT_DIR=$(pwd)
if [[ ! "$CURRENT_DIR" =~ \.worktrees/ ]]; then
    echo "ERROR: Not committing from within a worktree directory"
    echo "Current directory: $CURRENT_DIR"
    echo "Expected to be in: .worktrees/<session-slug>/"
    exit 1
fi

# Optional: Verify session environment is sourced
if [[ -z "$SESSION_SLUG" ]]; then
    echo "WARNING: SESSION_SLUG not set. Have you sourced the session environment?"
    echo "Run: source ../../_AGENTS/sessions/active/<session-slug>/.session-env"
    # Non-blocking warning for now
fi

exit 0
```

## Hook 2: Pre-push Hook

### Purpose
Ensure pushes only occur for session branches to prevent accidentally pushing to main/dev.

### Implementation

```bash
#!/bin/bash
# Pre-push hook for Agent Sessions Protocol
# This hook enforces session branch push requirements

# Read stdin for refs being pushed
while read local_ref local_sha remote_ref remote_sha
do
    # Extract branch name from ref
    if [[ "$local_ref" =~ refs/heads/(.+) ]]; then
        BRANCH_NAME="${BASH_REMATCH[1]}"
        
        # Check if it's a session branch
        if [[ "$BRANCH_NAME" != session/* ]]; then
            echo "ERROR: Cannot push non-session branch: $BRANCH_NAME"
            echo "Only branches starting with 'session/' can be pushed"
            echo "Attempted to push: $BRANCH_NAME"
            exit 1
        fi
    fi
done

exit 0
```

## Hook Installation Method

### Challenges
- Existing hooks may already be in place
- Hooks are single files, not directories
- Must not overwrite existing functionality
- Must work across platforms (macOS, Linux, Windows)

### Solution: Hook Wrapper Pattern

Instead of directly installing hooks, we:
1. Create our protocol hooks as separate scripts
2. Create/modify the main hook file to source all hook scripts in a hooks directory
3. This allows multiple hooks to coexist

### Directory Structure
```
.git/
  hooks/
    pre-commit              # Main hook file (wrapper)
    pre-push               # Main hook file (wrapper)
    hooks.d/               # Directory for hook scripts
      00-sessions-protocol-precommit.sh
      00-sessions-protocol-prepush.sh
```

### Wrapper Hook Template (pre-commit)

```bash
#!/bin/bash
# Git hook wrapper - sources all scripts in hooks.d/
# This allows multiple hooks to coexist

HOOK_NAME="pre-commit"
HOOKS_DIR="$(dirname "$0")/hooks.d"

# Exit on first failure
set -e

# Run all executable scripts in hooks.d/ for this hook
if [ -d "$HOOKS_DIR" ]; then
    for hook in "$HOOKS_DIR"/*-$HOOK_NAME.sh; do
        if [ -x "$hook" ]; then
            "$hook" "$@"
        elif [ -f "$hook" ]; then
            # Make executable if not already
            chmod +x "$hook"
            "$hook" "$@"
        fi
    done
fi

exit 0
```

### Installation Script

```bash
#!/bin/bash
# install-session-hooks.sh
# Safely installs Agent Sessions Protocol Git hooks

set -e

GIT_DIR=$(git rev-parse --git-dir)
HOOKS_DIR="$GIT_DIR/hooks"
HOOKS_D_DIR="$HOOKS_DIR/hooks.d"

echo "Installing Agent Sessions Protocol Git hooks..."

# Create hooks.d directory if it doesn't exist
mkdir -p "$HOOKS_D_DIR"

# Copy protocol hook scripts
echo "Installing hook scripts..."
cp "$(dirname "$0")/hooks/sessions-protocol-precommit.sh" \
   "$HOOKS_D_DIR/00-sessions-protocol-precommit.sh"
cp "$(dirname "$0")/hooks/sessions-protocol-prepush.sh" \
   "$HOOKS_D_DIR/00-sessions-protocol-prepush.sh"

# Make executable
chmod +x "$HOOKS_D_DIR/00-sessions-protocol-precommit.sh"
chmod +x "$HOOKS_D_DIR/00-sessions-protocol-prepush.sh"

# Install or update wrapper hooks
for HOOK_NAME in pre-commit pre-push; do
    HOOK_FILE="$HOOKS_DIR/$HOOK_NAME"
    
    # Check if hook already exists
    if [ -f "$HOOK_FILE" ]; then
        # Check if it's already our wrapper
        if grep -q "hooks.d" "$HOOK_FILE"; then
            echo "Wrapper hook already exists for $HOOK_NAME, skipping"
            continue
        fi
        
        # Backup existing hook
        echo "Backing up existing $HOOK_NAME hook..."
        BACKUP_NAME="$HOOK_FILE.backup-$(date +%Y%m%d-%H%M%S)"
        mv "$HOOK_FILE" "$BACKUP_NAME"
        
        # Convert old hook to hooks.d script
        echo "Converting old hook to hooks.d format..."
        mv "$BACKUP_NAME" "$HOOKS_D_DIR/99-legacy-$HOOK_NAME.sh"
        chmod +x "$HOOKS_D_DIR/99-legacy-$HOOK_NAME.sh"
    fi
    
    # Install wrapper
    echo "Installing wrapper for $HOOK_NAME..."
    cat > "$HOOK_FILE" << 'EOF'
#!/bin/bash
# Git hook wrapper - sources all scripts in hooks.d/
# This allows multiple hooks to coexist

HOOK_NAME="$(basename "$0")"
HOOKS_DIR="$(dirname "$0")/hooks.d"

# Exit on first failure
set -e

# Run all executable scripts in hooks.d/ for this hook
if [ -d "$HOOKS_DIR" ]; then
    for hook in "$HOOKS_DIR"/*-$HOOK_NAME.sh; do
        if [ -x "$hook" ]; then
            "$hook" "$@"
        elif [ -f "$hook" ]; then
            # Make executable if not already
            chmod +x "$hook"
            "$hook" "$@"
        fi
    done
fi

exit 0
EOF
    
    chmod +x "$HOOK_FILE"
done

echo "✓ Git hooks installed successfully!"
echo ""
echo "Installed hooks:"
echo "  - pre-commit: Enforces session branch and worktree usage"
echo "  - pre-push: Prevents pushing non-session branches"
echo ""
echo "Hook scripts location: $HOOKS_D_DIR"
```

## Testing Strategy

### Test Cases

1. **Test pre-commit on non-session branch**
   - Expected: Commit blocked with error message
   
2. **Test pre-commit outside worktree**
   - Expected: Commit blocked with error message
   
3. **Test pre-commit on session branch in worktree**
   - Expected: Commit succeeds
   
4. **Test pre-push for session branch**
   - Expected: Push succeeds
   
5. **Test pre-push for non-session branch**
   - Expected: Push blocked with error message

6. **Test installation with existing hooks**
   - Expected: Existing hooks preserved and converted to hooks.d format

### Manual Test Script

```bash
#!/bin/bash
# test-hooks.sh - Manual test script for Git hooks

echo "=== Testing Git Hooks ==="

# Test 1: Pre-commit on non-session branch
echo "Test 1: Attempting commit on main branch..."
git checkout main 2>/dev/null || git checkout -b main
echo "test" > test.txt
git add test.txt
if git commit -m "test" 2>&1 | grep -q "ERROR.*non-session branch"; then
    echo "✓ Test 1 passed: Pre-commit correctly blocks non-session branch"
else
    echo "✗ Test 1 failed"
fi
git reset HEAD~1 --soft 2>/dev/null

# Test 2: Pre-commit outside worktree (on session branch)
echo "Test 2: Attempting commit on session branch outside worktree..."
git checkout -b session/test-hook 2>/dev/null || git checkout session/test-hook
if git commit -m "test" 2>&1 | grep -q "ERROR.*worktree"; then
    echo "✓ Test 2 passed: Pre-commit correctly blocks commits outside worktree"
else
    echo "✗ Test 2 failed"
fi

# Test 3: Pre-commit in worktree (simulated)
echo "Test 3: Would succeed in worktree (manual verification needed)"

# Cleanup
git checkout main 2>/dev/null
git branch -D session/test-hook 2>/dev/null
rm -f test.txt

echo "=== Tests complete ==="
```

## Rollback Procedure

If hooks need to be removed:

```bash
#!/bin/bash
# uninstall-session-hooks.sh

GIT_DIR=$(git rev-parse --git-dir)
HOOKS_DIR="$GIT_DIR/hooks"
HOOKS_D_DIR="$HOOKS_DIR/hooks.d"

echo "Removing Agent Sessions Protocol hooks..."

# Remove protocol hook scripts
rm -f "$HOOKS_D_DIR/00-sessions-protocol-precommit.sh"
rm -f "$HOOKS_D_DIR/00-sessions-protocol-prepush.sh"

# Restore legacy hooks if they exist
for HOOK_NAME in pre-commit pre-push; do
    LEGACY_HOOK="$HOOKS_D_DIR/99-legacy-$HOOK_NAME.sh"
    if [ -f "$LEGACY_HOOK" ]; then
        echo "Restoring legacy $HOOK_NAME hook..."
        mv "$LEGACY_HOOK" "$HOOKS_DIR/$HOOK_NAME"
    else
        # Remove wrapper if no other hooks exist
        if [ -z "$(ls -A "$HOOKS_D_DIR" 2>/dev/null)" ]; then
            rm -f "$HOOKS_DIR/$HOOK_NAME"
        fi
    fi
done

# Remove hooks.d if empty
if [ -z "$(ls -A "$HOOKS_D_DIR" 2>/dev/null)" ]; then
    rmdir "$HOOKS_D_DIR"
fi

echo "✓ Hooks uninstalled"
```

## Integration Points

1. **Session Claiming**: The `claim-session` script should run `install-session-hooks.sh` after creating the worktree
2. **Session Environment**: The `.session-env` file should set `SESSION_SLUG`, `SESSION_BRANCH`, and `SESSION_DIR`
3. **Documentation**: Update `SESSIONS-REFERENCE.md` to document hook behavior and installation

## Future Enhancements

1. **Configurable strictness**: Allow users to configure warnings vs errors
2. **Bypass mechanism**: Add `--no-verify` documentation for emergency commits
3. **Remote validation**: Add server-side hooks to enforce protocol on push
4. **Metrics**: Track hook executions and violations for analysis