
# Hub-and-Spoke Architecture with Shallow Clones

## Overview

This document explores a hub-and-spoke architecture where the main on-disk repository serves as the hub (connected to cloud), and ephemeral session clones are spokes (shallow clones using main as upstream). This design acknowledges that all sessions are temporary by definition.

## Architecture

```
                    ┌─────────────────┐
                    │  Cloud Remote   │
                    │ (GitHub/GitLab) │
                    └────────┬────────┘
                             │
                         push/pull
                         (only main)
                             │
                    ┌────────▼────────┐
                    │   Main Repo     │
                    │   (on-disk)     │
                    │  Single Source  │
                    └────────┬────────┘
                             │
                    shallow clone (local)
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   ┌────▼─────┐        ┌────▼─────┐        ┌────▼─────┐
   │ Session 1│        │ Session 2│        │ Session 3│
   │ (shallow)│        │ (shallow)│        │ (shallow)│
   │ephemeral │        │ephemeral │        │ephemeral │
   └──────────┘        └──────────┘        └──────────┘
```

## Core Principles

### 1. Sessions Are Ephemeral

All sessions are **temporary by definition**:
- Created for specific task
- Used for hours/days
- Deleted when complete
- Never intended to be permanent

**Implication**: Shallow clones are perfect—full history isn't needed for temporary work.

### 2. Main Repo is Hub

The main repository is the **single source of truth**:
- Only connection to cloud
- Permanent fixture
- Full history maintained
- Manages all remote sync

**Implication**: Only one repo needs cloud credentials and sync logic.

### 3. Sessions Use Main as Upstream

Session clones treat main repo as their origin:
- Clone from local main (fast)
- Push back to local main
- No direct cloud access
- Simple local workflow

**Implication**: Sessions don't need network access or cloud credentials.

## Implementation

### Session Creation

```bash
#!/bin/bash
# create-session-shallow-spoke.sh

SESSION_SLUG="$1"
MAIN_REPO="$(git rev-parse --show-toplevel)"
SESSION_DIR=".sessions/$SESSION_SLUG"

echo "Creating shallow session clone: $SESSION_SLUG"

# Shallow clone from main repo (local, fast)
git clone --depth 1 --single-branch --branch main "$MAIN_REPO" "$SESSION_DIR"

cd "$SESSION_DIR"

# Set main repo as origin (upstream)
git remote rename origin upstream

# Create session branch
git checkout -b "session/$SESSION_SLUG"

# Configure for pushing back to main
git config push.default current

# Session environment
cat > .session-env <<EOF
export SESSION_SLUG="$SESSION_SLUG"
export SESSION_BRANCH="session/$SESSION_SLUG"
export SESSION_DIR="\$(pwd)"
export SESSION_UPSTREAM="$MAIN_REPO"
EOF

echo "✓ Session created"
echo "  Source:   $MAIN_REPO (local)"
echo "  Location: $SESSION_DIR"
echo "  Branch:   session/$SESSION_SLUG"
echo "  Method:   Shallow clone (ephemeral)"
```

### Session Workflow

```bash
# In session directory
cd .sessions/my-feature
source .session-env

# Work normally
echo "new feature" > feature.txt
git add feature.txt
git commit -m "Add feature"

# Push to main repo (local)
git push upstream session/my-feature

# Main repo now has session branch
# Ready to sync to cloud when appropriate
```

### Session Completion

```bash
#!/bin/bash
# complete-session-spoke.sh

SESSION_SLUG="$1"
SESSION_DIR=".sessions/$SESSION_SLUG"
MAIN_REPO="$(git rev-parse --show-toplevel)"

cd "$SESSION_DIR"

# Push session branch to main repo
git push upstream "session/$SESSION_SLUG"

echo "✓ Session pushed to main repo"

# Back to main repo
cd "$MAIN_REPO"

# Optionally: Merge session branch
git merge "session/$SESSION_SLUG"

# Optionally: Push to cloud
git push origin main

# Delete session clone
rm -rf "$SESSION_DIR"

echo "✓ Session completed and cleaned"
```

### Main Repo Sync

```bash
#!/bin/bash
# sync-main-to-cloud.sh

# In main repo
cd "$(git rev-parse --show-toplevel)"

# Fetch from cloud
git fetch origin

# Merge or rebase as needed
git merge origin/main  # or git rebase origin/main

# Push session branches to cloud
git push origin --all

# Push tags
git push origin --tags

"✓ Main repo synced with cloud"
```

## Advantages of Hub-Spoke Architecture

### 1. Perfect for Ephemeral Sessions

Since sessions are temporary:
- ✅ Shallow clones have minimal history (sufficient for task)
- ✅ Fast creation (sessions start quickly)
- ✅ Small disk footprint (each session is lightweight)
- ✅ Easy cleanup (just delete directory)
- ✅ No history baggage (sessions don't need it)

### 2. Centralized Cloud Access

Only main repo touches cloud:
- ✅ Single point of credential management
- ✅ One location for network configuration
- ✅ Simplified auth (SSH keys, tokens in one place)
- ✅ Controlled sync (explicit push/pull to cloud)
- ✅ Reduced network traffic (sessions are local-only)

### 3. Network Independence

Sessions work offline:
- ✅ No cloud connection needed for session work
- ✅ No network delays during development
- ✅ Work anywhere (airplane, train, etc.)
- ✅ No rate limiting issues
- ✅ No remote timeouts

### 4. Simplified Mental Model

Clear hierarchy:
- Cloud ← Main → Sessions
- Sessions only know about main
- Main handles all cloud sync
- Simple, predictable workflow

### 5. Filesystem Independence

Unlike reference clones:
- ✅ Works across different filesystems
- ✅ Can be on network storage
- ✅ Can be in different volumes
- ✅ No hardlink requirements
- ✅ Platform-agnostic

### 6. No Dependency Fragility

Unlike reference clones:
- ✅ Sessions are independent copies
- ✅ Main repo can be moved/renamed
- ✅ No alternates file to break
- ✅ No GC coordination needed
- ✅ Robust and reliable

## Workflow Examples

### Example 1: Create Session, Work, Complete

```bash
# 1. Create session
./claim-session implement-feature-x

# 2. Work in session
cd .sessions/implement-feature-x
source .session-env

# Make changes
vim feature.py
git add feature.py
git commit -m "Implement feature X"

# 3. Push to main (local)
git push upstream session/implement-feature-x

# 4. Complete session
cd ../..
./complete-session implement-feature-x

# 5. Sync main to cloud (when ready)
git push origin session/implement-feature-x
```

### Example 2: Multiple Parallel Sessions

```bash
# Three developers, three features
./claim-session feature-a
./claim-session feature-b  
./claim-session feature-c

# Each works independently
cd .sessions/feature-a && <work>
cd .sessions/feature-b && <work>
cd .sessions/feature-c && <work>

# All push to main repo locally
# Main repo has three session branches
# Sync to cloud when appropriate
```

### Example 3: Stay in Sync with Main

```bash
# In session
cd .sessions/my-feature

# Fetch updates from main repo
git fetch upstream

# Merge main changes into session
git merge upstream/main

# Or rebase session on latest main
git rebase upstream/main

# Continue working with latest code
```

## Comparison with Other Approaches

### vs. Reference Clones

| Aspect | Hub-Spoke (Shallow) | Reference Clones |
|--------|---------------------|------------------|
| Filesystem | Any | Same only |
| Dependency | Independent | Dependent on main |
| Disk per session | ~50 MB | ~5 MB |
| Creation speed | Fast (2s) | Faster (0.5s) |
| Fragility | Robust | Can break |
| History | Limited | Full |
| Complexity | Low | Medium |

**Winner**: Hub-Spoke for **robustness and flexibility**

### vs. Worktrees

| Aspect | Hub-Spoke (Shallow) | Worktrees |
|--------|---------------------|-----------|
| Agent simplicity | High | Medium |
| Isolation | Complete | Shared state |
| Cleanup | Simple (rm -rf) | Special command |
| Hooks | Independent | Shared |
| Config | Independent | Shared |
| Violations | Harder | Easier |

**Winner**: Hub-Spoke for **isolation and simplicity**

### vs. Direct Cloud Clone

| Aspect | Hub-Spoke (Shallow) | Direct Cloud Clone |
|--------|---------------------|--------------------|
| Network needed | No | Yes |
| Creation speed | Fast (local) | Slow (network) |
| Offline work | Yes | No |
| Credentials | In main only | In every session |
| Network usage | Minimal | High |

**Winner**: Hub-Spoke for **speed and offline capability**

## Conclusion

The hub-and-spoke architecture with shallow clones is **ideal for the Agent Sessions Protocol** because:

1. **Aligns with Ephemeral Nature**: Sessions are temporary, shallow clones are temporary
2. **Simplifies Agent Understanding**: Standard Git workflow, no special concepts
3. **Maximizes Isolation**: Each session is completely independent
4. **Minimizes Risk**: Clear boundaries, hard to violate protocol
5. **Optimizes for Local Work**: Fast creation, offline capability, minimal network
6. **Centralizes Cloud Access**: Only main repo needs credentials/config
7. **Robust and Reliable**: No fragile dependencies, works anywhere

This architecture treats the main repository as the single source of truth and hub for cloud sync, while sessions are lightweight, ephemeral spokes that exist only as long as needed.

**Recommended as the default approach for the Agent Sessions Protocol.**
echo