# Temporary User with Restricted Permissions Investigation

## Overview

This document investigates the feasibility of creating a temporary, platform-agnostic (macOS, Linux, Windows) user with permissions restricted to only the worktree directory, and read-only access to other repository files, to prevent unauthorized edits.

## Objective

Create a user context that:
1. Can only write to `.worktrees/{session-slug}/`
2. Has read-only access to repository files
3. Works across macOS, Linux, and Windows
4. Can be created/destroyed automatically
5. Integrates with session workflow

## Platform Analysis

### Linux

#### User Creation
```bash
# Create temporary user
sudo useradd -M -s /bin/bash session-user-{slug}

# Set password (or use passwordless sudo)
sudo passwd session-user-{slug}
```

#### Permission Control
```bash
# Grant read-only to repository
sudo chmod -R u+r,g+r,o+r /path/to/repo

# Grant write access to worktree only
sudo chown -R session-user-{slug}:session-user-{slug} /path/to/repo/.worktrees/{slug}
sudo chmod -R u+rwx /path/to/repo/.worktrees/{slug}

# Use ACLs for finer control
sudo setfacl -R -m u:session-user-{slug}:r /path/to/repo
sudo setfacl -R -m u:session-user-{slug}:rwx /path/to/repo/.worktrees/{slug}
```

#### Cleanup
```bash
# Remove user when session ends
sudo userdel session-user-{slug}
```

**Pros:**
- Fine-grained permission control via ACLs
- Well-established user management
- Can use sudo for elevation

**Cons:**
- Requires sudo/root access
- Complex permission management
- May interfere with existing permissions

### macOS

#### User Creation
```bash
# Create user via dscl
sudo dscl . -create /Users/session-user-{slug}
sudo dscl . -create /Users/session-user-{slug} UserShell /bin/bash
sudo dscl . -create /Users/session-user-{slug} UniqueID 5000
sudo dscl . -create /Users/session-user-{slug} PrimaryGroupID 5000
sudo dscl . -passwd /Users/session-user-{slug} password
```

#### Permission Control
```bash
# Similar to Linux, using chmod and chown
sudo chmod -R +a "session-user-{slug} allow read" /path/to/repo
sudo chmod -R +a "session-user-{slug} allow read,write,delete" /path/to/repo/.worktrees/{slug}
```

#### Cleanup
```bash
# Remove user
sudo dscl . -delete /Users/session-user-{slug}
```

**Pros:**
- Similar to Linux
- Native ACL support

**Cons:**
- Requires admin privileges
- More complex than Linux
- Different syntax than Linux

### Windows

#### User Creation
```powershell
# Create local user
net user session-user-{slug} /add

# Or via PowerShell
New-LocalUser -Name "session-user-{slug}" -NoPassword
```

#### Permission Control
```powershell
# Set read-only on repository
icacls "C:\path\to\repo" /grant "session-user-{slug}:(R)"

# Set read/write on worktree
icacls "C:\path\to\repo\.worktrees\{slug}" /grant "session-user-{slug}:(OI)(CI)(F)"
```

#### Cleanup
```powershell
# Remove user
net user session-user-{slug} /delete
```

**Pros:**
- Native permission system
- Granular access control

**Cons:**
- Requires administrator privileges
- Very different from Unix systems
- UAC complications

## Cross-Platform Challenges

### 1. Privilege Elevation

**Problem:** Creating users requires root/admin access

**Solutions:**
- Pre-create users during system setup
- Use containerization instead
- Abandon user-based approach

### 2. Permission Inheritance

**Problem:** New files may not inherit correct permissions

**Solutions:**
- Set default ACLs on directories
- Use umask to control defaults
- Monitor and adjust permissions

### 3. Process Execution

**Problem:** Switching user context mid-session is complex

**Solutions:**
- Run entire session as temporary user
- Use `su` (Linux/macOS) or `runas` (Windows)
- Containerize the session

### 4. Development Tool Integration

**Problem:** VSCode, Git, etc. may not work well with user switching

**Solutions:**
- Configure tools for multi-user
- Use SSH or remote contexts
- Accept limitations

## Alternative Approaches

### Alternative 1: Docker/Container-Based Isolation

Instead of OS users, use containers:

```dockerfile
# Session container
FROM ubuntu:latest

# Create non-root user
RUN useradd -m -s /bin/bash session-user

# Copy repository (read-only)
COPY --chown=root:root /repo /repo
RUN chmod -R 444 /repo

# Create writable worktree
RUN mkdir -p /repo/.worktrees/{slug}
RUN chown -R session-user:session-user /repo/.worktrees/{slug}

USER session-user
WORKDIR /repo/.worktrees/{slug}
```

**Pros:**
- Perfect isolation
- Cross-platform (via Docker)
- Easy to create/destroy
- No system-level permission changes

**Cons:**
- Requires Docker
- Performance overhead
- Complexity in setup
- File sync challenges

### Alternative 2: Virtual Environments with Restricted Paths

Use language-specific virtual environments with path restrictions:

```python
# Python example
import os
import sys

class RestrictedPath:
    def __init__(self, allowed_path):
        self.allowed = os.path.abspath(allowed_path)
    
    def validate(self, path):
        abs_path = os.path.abspath(path)
        if not abs_path.startswith(self.allowed):
            raise PermissionError(f"Access denied: {path}")
        return abs_path

# Use in agent code
worktree = RestrictedPath(".worktrees/session-slug")
file_path = worktree.validate("some/file.txt")
```

**Pros:**
- No system-level changes
- Cross-platform
- Programmatic control

**Cons:**
- Only works within application
- Can be bypassed
- Not OS-enforced

### Alternative 3: Git Worktree + Read-Only Main Checkout

Instead of user permissions, make main checkout read-only:

```bash
# Make main repository read-only
chmod -R a-w .git
chmod -R a-w * (except .worktrees)

# Worktrees remain writable
chmod -R u+w .worktrees/{slug}
```

**Pros:**
- No user management
- Simple to implement
- Cross-platform

**Cons:**
- Can be easily changed back
- Not secure against determined user
- May interfere with Git operations

## Feasibility Assessment

### Technical Feasibility: ⚠️ MODERATE

| Aspect | Feasibility | Notes |
|--------|-------------|-------|
| Linux | ✅ High | Well-supported, ACLs work well |
| macOS | ⚠️ Medium | Possible but complex |
| Windows | ⚠️ Medium | Very different implementation |
| Cross-platform | ❌ Low | Would need 3 different implementations |
| Automation | ⚠️ Medium | Requires elevated privileges |
| Integration | ❌ Low | Complex with dev tools |

### Practical Feasibility: ❌ LOW

**Major Blockers:**

1. **Privilege Requirements**
   - Creating users requires root/admin
   - Automating sudo/admin access is security risk
   - May not have privileges in all environments

2. **Tool Compatibility**
   - VSCode may not handle user switching well
   - Git operations may be disrupted
   - SSH keys, credentials tied to original user

3. **Workflow Disruption**
   - Switching users mid-session is jarring
   - Environment variables don't transfer
   - Open files, terminals would be lost

4. **Maintenance Complexity**
   - Three separate implementations
   - Permission debugging is difficult
   - Cleanup on session failure

## Recommended Approach

### ❌ Do NOT Implement OS-Level User Restrictions

**Reasons:**
1. Too complex for the benefit
2. Requires elevated privileges
3. Platform-specific implementations
4. Tool compatibility issues
5. Workflow disruption

### ✅ Alternative: Multi-Layered Soft Restrictions

Instead, use a combination of:

1. **Git Hooks** (already designed)
   - Prevent commits outside worktree
   - Block wrong branch operations

2. **Shell Function Overrides** (already designed)
   - Catch file operations
   - Validate paths before execution

3. **Environment-Based Restrictions** (already designed)
   - Validate $SESSION_DIR
   - Check prompt indicators

4. **Agent Training** (already designed)
   - Explicit guidelines
   - Awareness of restrictions

5. **Read-Only Main Checkout** (new)
   - Simple file permissions
   - No user management needed

```bash
# Simple read-only implementation
make_repo_readonly() {
    # Make everything except .worktrees read-only
    find . -maxdepth 1 -not -name '.worktrees' -not -name '.' -exec chmod -R a-w {} \;
    
    # Keep .worktrees writable
    chmod -R u+w .worktrees
}

make_repo_writable() {
    # Restore write permissions
    chmod -R u+w .
}
```

## Docker-Based Isolation (Optional Advanced Feature)

For users who want maximum isolation:

### Session Container Template

```dockerfile
FROM ubuntu:22.04

# Install dependencies
RUN apt-get update && apt-get install -y git

# Create session user
RUN useradd -m -s /bin/bash agent

# Repository will be mounted as volume
VOLUME /repo

# Worktree will be writable
VOLUME /repo/.worktrees

USER agent
WORKDIR /repo/.worktrees

ENTRYPOINT ["/bin/bash"]
```

### Usage

```bash
# Start session container
docker run -it --rm \
  -v /path/to/repo:/repo:ro \
  -v /path/to/repo/.worktrees/{slug}:/repo/.worktrees/{slug}:rw \
  session-container

# Inside container, repository is read-only except worktree
```

**When to Use:**
- High-security requirements
- Untrusted code execution
- Complete isolation needed
- Docker already in use

**When NOT to Use:**
- Normal development workflow
- Performance-critical operations
- Simple personal projects

## Implementation Recommendation

### Phase 1: Implement Soft Restrictions (Priority)

Focus on passive restraints already designed:
1. Git hooks
2. Shell overrides
3. Environment variables
4. Agent guidelines
5. File system guardrails

### Phase 2: Optional Read-Only Enhancement

Add simple file permission restrictions:
1. Make main checkout read-only
2. Keep worktrees writable
3. Provide toggle commands

### Phase 3: Optional Docker Support

For advanced users:
1. Create session container template
2. Document Docker workflow
3. Provide convenience scripts

## Conclusion

Creating temporary OS-level users with restricted permissions is:

- **Technically possible** but with significant platform-specific complexity
- **Practically infeasible** due to privilege requirements and tool compatibility
- **Not recommended** for the Agent Sessions Protocol

Instead, the multi-layered approach of passive restraints (hooks, overrides, validation) provides sufficient protection without the complexity and limitations of user-based isolation.

For cases requiring maximum isolation, Docker containers are a better solution than temporary users.

## Decision

**Do NOT pursue temporary user creation.**

**Rationale:**
1. Complexity far exceeds benefit
2. Requires privileges we can't assume
3. Breaks development workflows
4. Platform-specific nightmares
5. Soft restrictions are sufficient

**Action:**
Proceed with designed passive restraints and consider Docker as optional advanced feature for users who need it.