# Per-Session Cloning Evaluation

## Overview

This document evaluates the feasibility and implications of dropping Git worktrees in favor of creating a fresh clone of the repository for each session to ensure complete isolation and prevent cross-session interference.

## Current Approach: Git Worktrees

### How It Works

```bash
# Create worktree for session
git worktree add -b session/my-feature .worktrees/my-feature HEAD

# Work in worktree
cd .worktrees/my-feature

# Cleanup
git worktree remove .worktrees/my-feature
```

### Advantages

1. **Shared Git Objects**: All worktrees share `.git` directory
2. **Disk Efficient**: No duplication of Git history
3. **Fast Creation**: Instant worktree creation
4. **Native Git Feature**: Well-supported, documented
5. **Easy Branch Management**: Branches shared across worktrees

### Disadvantages

1. **Complex for Agents**: Conceptually harder to understand
2. **Shared State**: Git config, hooks shared
3. **Potential Interference**: Operations in one worktree can affect others
4. **Directory Structure**: Requires understanding of worktree paths
5. **Cleanup Critical**: Orphaned worktrees can cause issues

## Proposed Alternative: Per-Session Cloning

### How It Would Work

```bash
# Clone repository for session
git clone /path/to/main/repo .sessions/my-feature
cd .sessions/my-feature

# Create and checkout session branch
git checkout -b session/my-feature

# Work normally
# ... make changes ...

# Push when done
git push origin session/my-feature

# Cleanup
cd ..
rm -rf .sessions/my-feature
```

### Advantages

1. **Complete Isolation**: Each session is truly independent
2. **Simpler Mental Model**: Just a normal Git repository
3. **No Shared State**: Separate Git config, hooks per session
4. **Familiar Workflow**: Standard Git operations
5. **Easy Cleanup**: Just delete the directory
6. **Parallel Safety**: No risk of interfering with other sessions
7. **Simpler for Agents**: No worktree concept to understand

### Disadvantages

1. **Disk Space**: Full copy of Git history per session
2. **Clone Time**: Initial clone takes time (depends on repo size)
3. **Network Usage**: If cloning from remote
4. **Object Duplication**: No sharing of Git objects
5. **Update Complexity**: Need to fetch/pull to stay in sync

## Detailed Comparison

### Disk Space

| Metric | Worktrees | Per-Session Clones |
|--------|-----------|-------------------|
| Working Files | 1x per worktree | 1x per clone |
| Git Objects | 1x shared | 1x per clone |
| Typical Session (50MB repo) | ~50MB | ~100-150MB |
| 5 Active Sessions | ~250MB | ~500-750MB |

**Winner**: Worktrees (more disk efficient)

### Creation Speed

| Repository Size | Worktree Create | Clone (Local) | Clone (Remote) |
|----------------|----------------|---------------|----------------|
| Small (10MB) | < 1 second | ~2 seconds | ~5 seconds |
| Medium (100MB) | < 1 second | ~5 seconds | ~30 seconds |
| Large (1GB) | < 1 second | ~30 seconds | ~5 minutes |

**Winner**: Worktrees (much faster creation)

### Conceptual Simplicity

| Aspect | Worktrees | Clones |
|--------|-----------|--------|
| Agent Understanding | Complex (shared .git) | Simple (independent repo) |
| Mental Model | Non-intuitive | Familiar |
| Directory Structure | Special (.worktrees) | Normal (any directory) |
| Git Operations | Shared context | Independent context |

**Winner**: Clones (simpler to understand)

### Isolation

| Aspect | Worktrees | Clones |
|--------|-----------|--------|
| File System | Isolated | Isolated |
| Git Config | Shared | Independent |
| Git Hooks | Shared | Independent |
| Branch Visibility | Shared | Independent |
| Stash | Shared | Independent |
| Reflog | Shared | Independent |

**Winner**: Clones (better isolation)

### Workflow Impact

| Operation | Worktrees | Clones |
|-----------|-----------|--------|
| Create Session | Instant | Slow (clone time) |
| Switch Sessions | cd command | cd command |
| Update from Main | git pull in main | git fetch + merge/rebase |
| Cleanup | git worktree remove | rm -rf |
| Parallel Sessions | Supported | Supported |

**Winner**: Tie (different trade-offs)

## Implementation Analysis

### Per-Session Cloning Implementation

#### Session Creation Script

```bash
#!/bin/bash
# claim-session-clone.sh

SESSION_SLUG="$1"
MAIN_REPO="$(git rev-parse --show-toplevel)"
SESSION_DIR=".sessions/$SESSION_SLUG"

echo "Creating session clone: $SESSION_SLUG"

# Clone the repository
git clone "$MAIN_REPO" "$SESSION_DIR"

# Enter the clone
cd "$SESSION_DIR"

# Create and checkout session branch
git checkout -b "session/$SESSION_SLUG"

# Set up session environment
cat > .session-env <<EOF
export SESSION_SLUG="$SESSION_SLUG"
export SESSION_BRANCH="session/$SESSION_SLUG"
export SESSION_DIR="\$(pwd)"
export SESSION_ACTIVE="true"
EOF

# Move session metadata to active
mv "$MAIN_REPO/_AGENTS/sessions/planned/$SESSION_SLUG" \
   "$MAIN_REPO/_AGENTS/sessions/active/$SESSION_SLUG"

echo "✓ Session ready: $SESSION_DIR"
echo "  cd $SESSION_DIR"
echo "  source .session-env"
```

#### Synchronization Strategy

To keep session clone in sync with main:

```bash
# Add main repo as remote
git remote add upstream "$MAIN_REPO"

# Fetch updates
git fetch upstream

# Merge main changes
git merge upstream/main

# Or rebase
git rebase upstream/main
```

#### Cleanup Script

```bash
#!/bin/bash
# complete-session-clone.sh

SESSION_SLUG="$1"
SESSION_DIR=".sessions/$SESSION_SLUG"

# Push session branch
cd "$SESSION_DIR"
git push origin "session/$SESSION_SLUG"

# Move metadata to completed
cd "$MAIN_REPO"
mv "_AGENTS/sessions/active/$SESSION_SLUG" \
   "_AGENTS/sessions/completed/$SESSION_SLUG"

# Remove clone
rm -rf "$SESSION_DIR"

echo "✓ Session completed and cleaned up"
```

### Hybrid Approach

Combine best of both:

```bash
# Use shallow clone to save space
git clone --depth 1 --single-branch "$MAIN_REPO" "$SESSION_DIR"

# Or use reference clone to share objects
git clone --reference "$MAIN_REPO" "$MAIN_REPO" "$SESSION_DIR"
```

#### Reference Clone

```bash
# Clone with reference to main repo (shares objects)
git clone --reference /path/to/main/repo /path/to/main/repo .sessions/my-feature

# Benefits:
# - Shares Git objects (disk efficient like worktrees)
# - Independent Git config and state (isolated like clones)
# - Fast creation (uses hardlinks)
```

**This could be the best of both worlds!**

## Use Case Analysis

### When Worktrees Are Better

1. **Frequent Session Creation**: Quick creation time matters
2. **Large Repositories**: Disk space is constrained
3. **Experienced Git Users**: Comfortable with worktree concept
4. **Shared Configuration**: Want same hooks/config across sessions
5. **Local-Only Work**: No need for remote sync

### When Clones Are Better

1. **Infrequent Sessions**: Clone time is acceptable
2. **Complete Isolation Needed**: Different configs per session
3. **Simpler Mental Model**: Easier for agents to understand
4. **Independent Lifecycles**: Sessions may live long
5. **Remote Collaboration**: Each session syncs independently

## Agent Perspective

### Worktrees from Agent View

```
Complexity: HIGH
- Must understand .git is shared
- Must work in .worktrees/ directory
- Must be careful with git operations
- Needs special awareness of worktree concept

Violation Risk: MEDIUM
- Can accidentally work outside worktree
- Shared hooks can be confusing
- Directory structure is non-obvious
```

### Clones from Agent View

```
Complexity: LOW
- Standard Git repository
- Work in any directory
- Normal git operations
- Familiar workflow

Violation Risk: LOW
- Self-contained repository
- Clear boundaries
- Independent state
```

## Recommendation

### Short-Term: KEEP WORKTREES with Enhanced Passive Restraints

**Rationale:**
1. Better disk efficiency
2. Faster session creation
3. Native Git feature
4. Passive restraints solve violation issues

**With Conditions:**
- Implement all designed passive restraints
- Add agent awareness guidelines
- Use Git hooks for enforcement
- Enhanced shell prompts

### Long-Term: OFFER BOTH OPTIONS

**Default: Worktrees** (for most users)
- Disk efficient
- Fast creation
- Good for frequent sessions

**Alternative: Reference Clones** (for isolation needs)
- Complete isolation
- Simpler mental model
- Better for complex sessions

### Implementation Strategy

```bash
# claim-session with option
claim-session my-feature --mode=worktree  # Default
claim-session my-feature --mode=clone     # Alternative
claim-session my-feature --mode=ref-clone # Best hybrid
```

## Reference Clone Deep Dive

### How Reference Clones Work

```bash
# Create clone that references main repo for objects
git clone --reference /path/to/main /path/to/main .sessions/my-feature

# Result:
# - Working files: Independent copy
# - Git objects: Hardlinks/references to main repo
# - Git config: Independent
# - Git hooks: Independent
```

### Benefits

- **Disk Efficient**: Like worktrees (shares objects)
- **Fast Creation**: Uses hardlinks, very quick
- **Complete Isolation**: Independent config, hooks, refs
- **Simple for Agents**: Just a normal repo
- **Safe Cleanup**: No special worktree removal needed

### Drawbacks

- **Object Dependency**: Deleting main repo breaks references
- **Less Known Feature**: Not as commonly used
- **Platform Limitations**: Hardlinks may not work on all filesystems

### Reference Clone Implementation

```bash
#!/bin/bash
# claim-session-refclone.sh

SESSION_SLUG="$1"
MAIN_REPO="$(git rev-parse --show-toplevel)"
SESSION_DIR=".sessions/$SESSION_SLUG"

# Create reference clone
git clone --reference "$MAIN_REPO" "$MAIN_REPO" "$SESSION_DIR"

cd "$SESSION_DIR"

# Create session branch
git checkout -b "session/$SESSION_SLUG"

# Configure independent hooks
cp -r "$MAIN_REPO/.git/hooks" ".git/hooks"

# Set up session environment
# ... (same as before)

echo "✓ Reference clone created"
echo "  Disk usage: $(du -sh . | cut -f1)"
```

## Migration Path

If we decide to switch:

### Phase 1: Support Both (Current + New)

```bash
# claim-session auto-detects or accepts flag
claim-session my-feature              # Uses current (worktree)
claim-session my-feature --clone      # Uses new (clone/ref-clone)
```

### Phase 2: Default to New (with Backward Compatibility)

```bash
# Default changes to clone
claim-session my-feature              # Uses clone
claim-session my-feature --worktree   # Uses old (worktree)
```

### Phase 3: Deprecate Old

```bash
# Only clone supported
claim-session my-feature              # Uses clone
# --worktree flag shows deprecation warning
```

## Decision Matrix

| Criterion | Weight | Worktrees | Clones | Ref-Clones |
|-----------|--------|-----------|--------|------------|
| Disk Efficiency | 3 | 9 | 3 | 9 |
| Creation Speed | 2 | 10 | 5 | 9 |
| Agent Simplicity | 4 | 4 | 10 | 10 |
| Isolation | 4 | 6 | 10 | 10 |
| Violation Risk | 5 | 5 | 9 | 9 |
| **Total** | - | **125** | **152** | **175** |

**Winner: Reference Clones** (best balance of all factors)

## Final Recommendation

### Recommended Approach: Reference Clones as Default

1. **Implement reference clone support** in claim-session script
2. **Make it the default** for new sessions
3. **Keep worktree support** as fallback/option
4. **Migrate existing sessions** gradually

### Implementation Priority

1. **Immediate**: Complete passive restraints for current worktrees
2. **Short-term** (1-2 weeks): Add reference clone option
3. **Medium-term** (1-2 months): Make reference clones default
4. **Long-term** (3+ months): Deprecate plain worktrees

### Why Reference Clones Win

```
✓ Disk efficient (shares objects like worktrees)
✓ Fast creation (hardlinks)
✓ Complete isolation (independent config/hooks)
✓ Simple for agents (standard repo)
✓ Low violation risk (clear boundaries)
✓ Easy cleanup (just rm -rf)
✓ Familiar workflow (normal Git)
```

## Conclusion

After thorough evaluation:

1. **Do NOT drop worktrees entirely** - they have valid use cases
2. **DO implement reference clones** - best of both worlds
3. **DO make reference clones the default** - better isolation and simplicity
4. **DO keep worktrees as option** - for specific needs or preferences

Reference clones provide the isolation and simplicity of clones with the efficiency of worktrees, making them ideal for the Agent Sessions Protocol while being much more intuitive for AI agents to work with.

## Next Steps

1. Complete Phase 2 (passive restraints) for current worktree system
2. Design and implement reference clone support
3. Test reference clones with existing workflow
4. Update documentation for both approaches
5. Gradually migrate to reference clones as default