# Session: Worktree Untracked Files Synchronization

**Session ID:** 2025-10-15-worktree-untracked-files
**Created:** 2025-10-15
**Status:** drafting
**Priority:** high
**Type:** infrastructure

## Context

The current session protocol uses git worktrees to provide isolated workspaces for concurrent sessions. However, git worktrees only contain tracked files from the git repository. Untracked files that are critical for development work (such as `.env` files, local configuration, build artifacts, etc.) are not automatically copied into the worktree.

This creates a problem where:
- Agents starting a session in a worktree may lack necessary configuration files
- Development environment may be incomplete or broken
- Manual copying of files is required, which is error-prone and not documented
- Different sessions may need different versions of these untracked files

## Goals

Implement a systematic solution to handle untracked files needed for session work:
1. Identify which untracked files are necessary for session work
2. Develop a mechanism to provision these files into worktrees
3. Support both shared files (copied) and session-specific files (templated/configured)
4. Integrate the solution into existing session lifecycle scripts
5. Document the approach for agents and users

Preserve:
- Git's exclusion of truly temporary/generated files
- Security (don't expose secrets unnecessarily)
- Simplicity of worktree workflow
- No changes to core git behavior

## Acceptance Criteria

### Analysis
- [ ] Identify categories of untracked files:
  - Configuration files (`.env`, `.envrc`, config files)
  - Build/cache directories
  - IDE settings
  - Local scripts/tools
  - Secrets/credentials
- [ ] Document which files should be:
  - Shared (same across all sessions)
  - Session-specific (different per session)
  - Never copied (truly temporary/generated)

### Solution Design
- [ ] Choose approach (evaluate options):
  - **Option A:** Copy manifest (list of files to copy on worktree creation)
  - **Option B:** Template system (generate files from templates)
  - **Option C:** Hybrid (copy some, template others)
  - **Option D:** Symlink approach (link to main workspace)
- [ ] Document trade-offs of chosen approach
- [ ] Define where configuration lives (`.session-templates/`, `.session-config/`, etc.)

### Implementation
- [ ] Update `claim-session` script to provision untracked files
- [ ] Create configuration/template storage location
- [ ] Add validation that required files exist in worktree
- [ ] Handle errors gracefully (missing templates, etc.)
- [ ] Support session-specific overrides where needed

### Documentation
- [ ] Document the untracked files mechanism in SESSIONS-README.md
- [ ] Add troubleshooting guide for common issues
- [ ] Document how to add new files to the provisioning system
- [ ] Provide examples of common patterns (env vars, config files)

### Testing
- [ ] Test with common scenarios:
  - Session with `.env` file requirements
  - Session needing IDE configuration
  - Multiple concurrent sessions with different configs
  - Missing template files (error handling)
- [ ] Verify security (no secrets leaked to git)
- [ ] Validate cleanup on session completion

## Technical Approach

### Phase 1: Discovery & Design
1. Audit current project for untracked files that sessions need
2. Research approaches used by similar tools (e.g., direnv, nix, docker)
3. Design solution that fits session protocol philosophy
4. Document design decisions

### Phase 2: Implementation Options

#### Option A: Copy Manifest
```bash
# .session-files.manifest
.env
.envrc
.tool-versions
local-config.yaml

# In claim-session:
while read -r file; do
  [ -f "$file" ] && cp "$file" ".worktrees/$SESSION_SLUG/$file"
done < .session-files.manifest
```

**Pros:**
- Simple to understand
- Easy to add files
- Files can be manually edited in main workspace

**Cons:**
- No session-specific customization
- Changes to main workspace affect all sessions
- May copy files that shouldn't be shared

#### Option B: Template System
```bash
# _AGENTS/sessions/_templates/env.template
DATABASE_URL=postgresql://localhost:5432/{{SESSION_SLUG}}
API_KEY={{API_KEY_FROM_SECURE_STORE}}
SESSION_ID={{SESSION_SLUG}}

# In claim-session:
for template in _AGENTS/sessions/_templates/*.template; do
  target=".worktrees/$SESSION_SLUG/.$(basename $template .template)"
  sed -e "s/{{SESSION_SLUG}}/$SESSION_SLUG/g" \
      -e "s/{{TIMESTAMP}}/$(date +%s)/g" \
      "$template" > "$target"
done
```

**Pros:**
- Session-specific configuration
- Can inject session context
- Templates tracked in git (documented)
- Secrets can be loaded from secure sources

**Cons:**
- More complex
- Requires template maintenance
- Learning curve for template syntax

#### Option C: Hybrid Approach
```bash
# _AGENTS/sessions/_config/session-files.yaml
copy:
  - .tool-versions
  - .editorconfig
template:
  - src: _templates/env.template
    dest: .env
  - src: _templates/session-config.template
    dest: .session-config
symlink:
  - node_modules
  - .cache
```

**Pros:**
- Best of both worlds
- Flexible for different file types
- Can optimize for performance (symlinks for large dirs)

**Cons:**
- Most complex
- Requires YAML parser (or JSON)
- Potentially overengineered

### Phase 3: Implementation

Recommended: **Hybrid Approach** (Option C) with minimal complexity

Structure:
```
_AGENTS/sessions/
  _templates/
    env.template
    session-config.template
  _config/
    untracked-files.conf  # Simple format
```

untracked-files.conf format:
```bash
# Format: ACTION:SOURCE:DEST
# Actions: copy, template, symlink, skip
copy:.tool-versions:.tool-versions
copy:.editorconfig:.editorconfig
template:_AGENTS/sessions/_templates/env.template:.env
symlink:node_modules:node_modules
```

### Phase 4: Integration
1. Update `claim-session`:
   - After worktree creation
   - Before session activation message
   - Run provisioning logic
2. Add validation check
3. Update documentation

### Phase 5: Documentation
- Document the conf file format
- Provide examples for common use cases
- Add troubleshooting section
- Document security considerations

## Out of Scope

- Dynamic file generation based on external services
- Encryption/decryption of secrets (use existing secret managers)
- Version control of untracked files (they're untracked for a reason)
- Migration of existing worktrees (manual if needed)
- Cross-platform compatibility beyond Linux/macOS

## Dependencies

- Existing session protocol and scripts
- Access to untracked files in main workspace
- Understanding of what files each session needs

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Secrets leaked to git | High | Clear documentation, .gitignore validation |
| File conflicts in worktree | Medium | Clear precedence rules, validation |
| Large files slow down session claim | Medium | Use symlinks for large directories |
| Template syntax complexity | Low | Keep simple, document well |
| Missing templates break claim | Medium | Validation with helpful error messages |

## Success Metrics

- Sessions can start without manual file copying
- Common untracked files provisioned automatically
- Clear documentation enables easy additions
- No secrets exposed in git history
- Performance impact < 2 seconds per session claim

## Open Questions

- [ ] Which files are actually needed for sessions in this project?
  - **Action:** Audit current project
- [ ] Should templates support complex logic or stay simple?
  - **Recommendation:** Start simple (variable substitution only)
- [ ] How to handle secrets securely?
  - **Recommendation:** Reference external secret manager, don't copy secrets
- [ ] Should existing worktrees be updated retroactively?
  - **Recommendation:** No, document manual process if needed

## Notes

- Keep solution simple and maintainable
- Prioritize common use cases over edge cases
- Document security considerations prominently
- Consider future CI/CD integration (may need different approach)
- Test with actual session scenarios before finalizing

## Proposed Subsessions

1. **Discovery & Audit**
   - Identify all untracked files in current project
   - Categorize by type and necessity
   - Document findings

2. **Design & Decision**
   - Evaluate approaches (A, B, C)
   - Choose solution based on project needs
   - Document design decisions and trade-offs

3. **Core Implementation**
   - Create configuration/template structure
   - Implement provisioning logic
   - Add to claim-session script
   - (Blocked by subsession 2)

4. **Validation & Error Handling**
   - Add validation checks
   - Implement error messages
   - Handle edge cases
   - (Blocked by subsession 3)

5. **Documentation**
   - Update SESSIONS-README.md
   - Add examples and troubleshooting
   - Document security considerations
   - (Blocked by subsession 4)

6. **Testing & Validation**
   - Test with real sessions
   - Validate security
   - Performance testing
   - (Blocked by subsession 5)

## Estimated Effort

- **Duration:** 1-2 days
- **Complexity:** Medium (requires design thinking and careful implementation)
- **Agent Capability:** Bash scripting, understanding of git worktrees, file system operations

