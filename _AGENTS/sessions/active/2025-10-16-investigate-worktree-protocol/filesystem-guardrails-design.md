# File System Guardrails Design

## Overview

This document defines strategies and rules for controlling where files can be created during active sessions, ensuring all session-related files remain within the designated worktree directory.

## Objective

Prevent file creation outside the session worktree by:
1. Defining allowed file creation paths
2. Implementing validation mechanisms
3. Providing clear error messages
4. Integrating with passive restraint systems

## Allowed File Creation Paths

### Primary Rule

**All session work files MUST be created within the worktree directory.**

```
Allowed:    .worktrees/{session-slug}/**/*
Forbidden:  Everything else
```

### Path Patterns

#### Allowed Patterns (Regex)

```regex
# Any file within the worktree for this session
^\.worktrees/[^/]+/.*$

# More specific - must match session slug
^\.worktrees/${SESSION_SLUG}/.*$
```

#### Forbidden Patterns (Regex)

```regex
# Repository root
^(?!\.worktrees/).*$

# Other worktrees
^\.worktrees/(?!${SESSION_SLUG}/).*$

# Parent directories
^\.\./.*$

# Absolute paths outside workspace
^/(?!.*\.worktrees/${SESSION_SLUG}).*$
```

### Exception Paths

Some operations may legitimately need to access files outside the worktree:

```
Read-only Access Allowed:
- _AGENTS/sessions/active/{session-slug}/  (session metadata)
- _AGENTS/sessions/_templates/              (templates)
- .git/                                     (Git metadata, read-only)
- README.md, CONTRIBUTING.md                (documentation, read-only)

Write Access Forbidden Everywhere Except:
- .worktrees/{session-slug}/**/*
```

## Validation Mechanisms

### 1. Path Validation Function

```bash
#!/bin/bash
# validate_file_path.sh
# Validates that a file path is within the session worktree

validate_file_path() {
    local file_path="$1"
    local operation="${2:-write}"  # write, read, or execute
    
    # Normalize path to absolute
    local abs_path=$(realpath -m "$file_path" 2>/dev/null || echo "$file_path")
    
    # Get session worktree path
    local worktree_path="${SESSION_DIR:-}"
    
    if [ -z "$worktree_path" ]; then
        echo "ERROR: SESSION_DIR not set. Source session environment first."
        return 1
    fi
    
    # Normalize worktree path
    worktree_path=$(realpath -m "$worktree_path")
    
    # Check if file is within worktree
    case "$abs_path" in
        "$worktree_path"/*)
            # File is within worktree - allowed
            return 0
            ;;
        *)
            # File is outside worktree
            if [ "$operation" = "read" ]; then
                # Read operations may be allowed for certain paths
                case "$abs_path" in
                    *"/_AGENTS/sessions/active/"*)
                        echo "INFO: Read access to session metadata allowed"
                        return 0
                        ;;
                    *"/_AGENTS/sessions/_templates/"*)
                        echo "INFO: Read access to templates allowed"
                        return 0
                        ;;
                    *)
                        echo "WARNING: Reading file outside worktree: $abs_path"
                        return 2  # Warning, not error
                        ;;
                esac
            else
                # Write/execute operations outside worktree are forbidden
                echo "ERROR: Cannot $operation file outside worktree"
                echo "  File:     $abs_path"
                echo "  Worktree: $worktree_path"
                return 1
            fi
            ;;
    esac
}

# Wrapper for common file operations
safe_touch() {
    validate_file_path "$1" "write" || return 1
    touch "$1"
}

safe_mkdir() {
    validate_file_path "$1" "write" || return 1
    mkdir -p "$1"
}

safe_write() {
    validate_file_path "$1" "write" || return 1
    cat > "$1"
}
```

### 2. Integration with Shell Functions

Override common file creation commands:

```bash
# Add to .session-env

# Override touch
touch() {
    local file="$1"
    if ! validate_file_path "$file" "write"; then
        echo "Blocked: touch $file"
        return 1
    fi
    command touch "$@"
}

# Override mkdir
mkdir() {
    local dir="$1"
    if ! validate_file_path "$dir" "write"; then
        echo "Blocked: mkdir $dir"
        return 1
    fi
    command mkdir "$@"
}

# Override file redirection helper
safe_redirect() {
    local file="$1"
    if ! validate_file_path "$file" "write"; then
        echo "Blocked: redirect to $file"
        return 1
    fi
    cat > "$file"
}
```

### 3. .roo/rules Integration

Define file system rules in `.roo/rules`:

```markdown
# File System Guardrails for Agent Sessions

## CRITICAL: Session File Isolation

When an active session exists in `_AGENTS/sessions/active/`:
- ALL file write operations MUST occur within `.worktrees/{session-slug}/`
- NO files may be created in the repository root
- NO files may be created in other worktrees

## File Path Validation

Before creating or modifying any file:
1. Verify `pwd` is within `.worktrees/{session-slug}/`
2. Verify target file path is within current worktree
3. If validation fails, STOP and correct directory

## Allowed File Operations

### Write Operations (Allowed)
- `.worktrees/{session-slug}/**/*` - All files within session worktree

### Read Operations (Allowed)
- `.worktrees/{session-slug}/**/*` - Session worktree files
- `_AGENTS/sessions/active/{session-slug}/**/*` - Session metadata (read-only)
- `_AGENTS/sessions/_templates/**/*` - Templates (read-only)
- Repository documentation (README.md, etc.) - Read-only

### Forbidden Operations
- ❌ Creating files in repository root
- ❌ Creating files in other worktrees
- ❌ Modifying files outside worktree (except via Git)
- ❌ Creating files in `_AGENTS/sessions/` directory structure

## Validation Commands

Check current working directory:
```bash
pwd | grep -q "\.worktrees/${SESSION_SLUG}" || echo "ERROR: Not in worktree"
```

Check file path before creation:
```bash
realpath -m "$filepath" | grep -q "\.worktrees/${SESSION_SLUG}" || echo "ERROR: File outside worktree"
```
```

## Error Messages

### Clear, Actionable Error Messages

When a file system guardrail is triggered:

```bash
echo "╔════════════════════════════════════════════════════════════╗"
echo "║  FILE SYSTEM GUARDRAIL VIOLATION                           ║"
echo "╚════════════════════════════════════════════════════════════╝"
echo ""
echo "❌ ERROR: Attempted to create file outside worktree"
echo ""
echo "File:     $attempted_path"
echo "Worktree: $SESSION_DIR"
echo ""
echo "Required Actions:"
echo "  1. Ensure you are in the worktree directory:"
echo "     cd .worktrees/${SESSION_SLUG}"
echo ""
echo "  2. Verify session environment is active:"
echo "     echo \$SESSION_DIR"
echo ""
echo "  3. Create files only within the worktree"
echo ""
echo "For help, run: session_info"
```

## Implementation Strategies

### Strategy 1: Preventive (Recommended)

Prevent file creation outside worktree using shell function overrides:

**Pros:**
- Catches errors before they happen
- No cleanup needed
- Clear immediate feedback

**Cons:**
- May be circumvented if function is not used
- Requires sourcing session environment

**Implementation:** Included in `.session-env` as function overrides

### Strategy 2: Detective

Monitor file creation and alert when violations occur:

**Pros:**
- Doesn't interfere with normal operations
- Can catch any creation method

**Cons:**
- Violations may occur before detection
- Requires cleanup after the fact

**Implementation:** Git pre-commit hook checks file locations

### Strategy 3: Corrective

Automatically move misplaced files to worktree:

**Pros:**
- Transparent to user
- Files always end up in correct place

**Cons:**
- May be confusing
- Could mask underlying issues

**Implementation:** Post-creation hook or cleanup script

### Recommended Approach: Layered Defense

Use all three strategies:

1. **Preventive** (Primary): Shell function overrides in `.session-env`
2. **Detective** (Secondary): Pre-commit hooks validate file locations
3. **Corrective** (Fallback): Manual cleanup tools for edge cases

## File Location Audit

### Audit Script

```bash
#!/bin/bash
# audit-file-locations.sh
# Audits files created during session to ensure proper location

SESSION_SLUG="${1:-$SESSION_SLUG}"
WORKTREE_DIR=".worktrees/$SESSION_SLUG"

if [ -z "$SESSION_SLUG" ]; then
    echo "ERROR: SESSION_SLUG not provided"
    exit 1
fi

echo "=== File Location Audit for $SESSION_SLUG ==="
echo ""

# Get session branch
SESSION_BRANCH="session/$SESSION_SLUG"

# Get list of files added in session
FILES=$(git diff --name-only "$SESSION_BRANCH" $(git merge-base main "$SESSION_BRANCH"))

# Check each file
VIOLATIONS=()
CORRECT=()

while IFS= read -r file; do
    if [[ "$file" == "$WORKTREE_DIR/"* ]]; then
        CORRECT+=("$file")
    else
        VIOLATIONS+=("$file")
    fi
done <<< "$FILES"

# Report results
if [ ${#VIOLATIONS[@]} -eq 0 ]; then
    echo "✅ All files in correct location"
    echo "   Total files: ${#CORRECT[@]}"
else
    echo "❌ File location violations found!"
    echo ""
    echo "Files in wrong location:"
    printf '  - %s\n' "${VIOLATIONS[@]}"
    echo ""
    echo "Correct files: ${#CORRECT[@]}"
    echo "Violations:    ${#VIOLATIONS[@]}"
    echo ""
    echo "Run cleanup script to fix: cleanup-misplaced-files.sh"
fi
```

### Cleanup Script

```bash
#!/bin/bash
# cleanup-misplaced-files.sh
# Moves misplaced files to correct worktree location

SESSION_SLUG="${1:-$SESSION_SLUG}"
WORKTREE_DIR=".worktrees/$SESSION_SLUG"

echo "=== Cleaning Up Misplaced Files ==="
echo ""

# Get misplaced files
FILES=$(git diff --name-only "session/$SESSION_SLUG" $(git merge-base main "session/$SESSION_SLUG") | grep -v "^$WORKTREE_DIR/")

if [ -z "$FILES" ]; then
    echo "✅ No misplaced files found"
    exit 0
fi

echo "Moving files to worktree:"
while IFS= read -r file; do
    if [ -f "$file" ]; then
        # Determine target path
        target="$WORKTREE_DIR/$file"
        
        # Create directory if needed
        mkdir -p "$(dirname "$target")"
        
        # Move file
        echo "  $file -> $target"
        git mv "$file" "$target"
    fi
done <<< "$FILES"

echo ""
echo "✅ Cleanup complete"
echo "Files moved to: $WORKTREE_DIR"
echo ""
echo "Commit these changes:"
echo "  git commit -m 'fix: Move files to correct worktree location'"
```

## Pre-commit Hook Integration

Add file location check to pre-commit hook:

```bash
#!/bin/bash
# Part of pre-commit hook

# Check if committing files outside worktree
if [ -n "$SESSION_SLUG" ]; then
    WORKTREE_DIR=".worktrees/$SESSION_SLUG"
    
    # Get staged files
    STAGED=$(git diff --cached --name-only)
    
    # Check each staged file
    OUTSIDE_WORKTREE=()
    while IFS= read -r file; do
        if [[ "$file" != "$WORKTREE_DIR/"* ]]; then
            OUTSIDE_WORKTREE+=("$file")
        fi
    done <<< "$STAGED"
    
    # Block commit if files outside worktree
    if [ ${#OUTSIDE_WORKTREE[@]} -gt 0 ]; then
        echo "ERROR: Cannot commit files outside worktree"
        echo ""
        echo "Files outside worktree:"
        printf '  - %s\n' "${OUTSIDE_WORKTREE[@]}"
        echo ""
        echo "Expected location: $WORKTREE_DIR/"
        echo ""
        echo "To fix, move files to worktree:"
        echo "  ./cleanup-misplaced-files.sh"
        exit 1
    fi
fi
```

## Testing Strategy

### Test Cases

```bash
#!/bin/bash
# test-filesystem-guardrails.sh

echo "=== Testing File System Guardrails ==="

# Setup
SESSION_SLUG="test-guardrails"
export SESSION_DIR=".worktrees/$SESSION_SLUG"
mkdir -p "$SESSION_DIR"
cd "$SESSION_DIR"
source ../../_AGENTS/sessions/active/$SESSION_SLUG/.session-env

# Test 1: File creation in worktree (should succeed)
echo "Test 1: Create file in worktree..."
if touch test-file.txt 2>/dev/null; then
    echo "✓ Test 1 passed"
else
    echo "✗ Test 1 failed"
fi

# Test 2: File creation outside worktree (should fail)
echo "Test 2: Attempt file creation outside worktree..."
if touch ../../bad-file.txt 2>&1 | grep -q "ERROR"; then
    echo "✓ Test 2 passed - creation blocked"
else
    echo "✗ Test 2 failed - creation allowed!"
fi

# Test 3: Directory creation in worktree (should succeed)
echo "Test 3: Create directory in worktree..."
if mkdir test-dir 2>/dev/null; then
    echo "✓ Test 3 passed"
else
    echo "✗ Test 3 failed"
fi

# Test 4: Directory creation outside worktree (should fail)
echo "Test 4: Attempt directory creation outside worktree..."
if mkdir ../../bad-dir 2>&1 | grep -q "ERROR"; then
    echo "✓ Test 4 passed - creation blocked"
else
    echo "✗ Test 4 failed - creation allowed!"
fi

# Cleanup
cd ../..
rm -rf ".worktrees/$SESSION_SLUG"

echo "=== Tests complete ==="
```

## Documentation Integration

### Update SESSIONS-REFERENCE.md

Add section on file system guardrails:

```markdown
## File System Guardrails

When working in a session, all file operations must occur within the session worktree.

### Allowed Operations

✅ Create files in `.worktrees/{session-slug}/`
✅ Read session metadata from `_AGENTS/sessions/active/{session-slug}/`
✅ Read templates from `_AGENTS/sessions/_templates/`

### Forbidden Operations

❌ Create files in repository root
❌ Create files in other worktrees
❌ Modify files outside worktree

### Validation

Your session environment includes automatic validation:
- File creation commands check path before executing
- Clear error messages if path is invalid
- Pre-commit hooks prevent committing misplaced files
```

## Future Enhancements

1. **Symbolic Link Handling**: Prevent symlinks from bypassing guardrails
2. **Temporary File Management**: Handle `/tmp` files appropriately
3. **Binary File Restrictions**: Additional checks for binary files
4. **Size Limits**: Enforce maximum file sizes
5. **File Type Restrictions**: Block certain file types if needed

## Conclusion

File system guardrails provide multiple layers of protection:

1. **Preventive**: Function overrides stop violations before they occur
2. **Detective**: Audit scripts identify violations
3. **Corrective**: Cleanup tools fix violations

Combined with Git hooks and agent awareness, these guardrails ensure session isolation and prevent cross-session interference.