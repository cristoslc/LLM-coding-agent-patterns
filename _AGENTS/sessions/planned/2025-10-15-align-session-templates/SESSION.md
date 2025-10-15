# Session: Align Session Templates with Worktree Workflow

## Context

The sessions protocol was recently migrated to use git worktrees for session isolation (session `2025-10-15-align-sessions-protocol`). The `claim-session` and `complete-session` scripts were updated to match this new workflow.

However, the **template files** in `_templates/` were never reviewed or updated:
- `kb-merge-SESSION.md` - Template for KB merge sessions
- `session-env.template` - Template for session environment files

These templates may contain:
- Outdated environment variable names (SESSION_BRANCH vs SESSION_ID)
- Incorrect workflow instructions
- No mention of worktree-based workflow
- No mention of read-only SESSION.md (when implemented)
- Outdated path references

This session ensures templates align with the current protocol and generate correct output when used by scripts.

## Acceptance Criteria

### Template Inventory
- [ ] Read both template files thoroughly
- [ ] Identify all variables used in templates
- [ ] List all instructions included in templates
- [ ] Note any workflow references
- [ ] Document current template usage by scripts

### Variable Alignment
- [ ] Remove any references to SESSION_BRANCH (deprecated)
- [ ] Ensure SESSION_ID is used consistently
- [ ] Ensure SESSION_SLUG is used consistently
- [ ] Verify GIT_AUTHOR_NAME format matches scripts
- [ ] Verify GIT_AUTHOR_EMAIL format matches scripts
- [ ] Verify GIT_COMMITTER_NAME format matches scripts
- [ ] Verify GIT_COMMITTER_EMAIL format matches scripts
- [ ] Add any missing variables used by scripts

### Workflow Alignment
- [ ] Update any checkout references to worktree references
- [ ] Update any path references to match worktree structure
- [ ] Add worktree activation instructions if needed
- [ ] Remove any agent-focused language (should be session-focused)
- [ ] Ensure instructions match current README/REFERENCE

### Script Integration
- [ ] Verify claim-session uses session-env.template correctly
- [ ] Verify complete-session uses kb-merge-SESSION.md correctly
- [ ] Ensure template variables match script substitutions
- [ ] Test that generated files are correct

### Content Quality
- [ ] Instructions are clear and accurate
- [ ] No outdated information remains
- [ ] Examples are realistic and helpful
- [ ] Comments explain template variables
- [ ] Templates are easy to understand and modify

### Future-Proofing
- [ ] Add note about read-only SESSION.md when implemented
- [ ] Document where updates should go (worklog, active-plan)
- [ ] Include worktree structure in templates
- [ ] Add references to README/REFERENCE for details

## Implementation Plan

### Phase 1: Inventory Templates (30 minutes)

1. **Read session-env.template:**
   - List all variables used
   - List all commands/instructions
   - Note any workflow references
   - Check for agent vs session language

2. **Read kb-merge-SESSION.md:**
   - List all template variables ({{VAR}})
   - Review structure and sections
   - Check acceptance criteria format
   - Check implementation plan format

3. **Check script usage:**
   - How does claim-session use session-env.template?
   - How does complete-session use kb-merge-SESSION.md?
   - What variables do scripts substitute?
   - Are there mismatches?

### Phase 2: Update session-env.template (45 minutes)

1. **Review current content:**
   ```bash
   cat _AGENTS/sessions/_templates/session-env.template
   ```

2. **Update variables:**
   - Ensure SESSION_ID is present
   - Remove SESSION_BRANCH if present
   - Update GIT_AUTHOR_NAME format if needed
   - Align all variables with claim-session script

3. **Update instructions:**
   - Add note about worktree location
   - Explain how to activate (from worktree)
   - Reference documentation for details

4. **Add comments:**
   - Explain each variable's purpose
   - Note which are required vs optional
   - Add examples of usage

5. **Test template:**
   - Manually substitute variables
   - Verify resulting file would work
   - Compare to actual .session-env from recent session

### Phase 3: Update kb-merge-SESSION.md (45 minutes)

1. **Review current content:**
   ```bash
   cat _AGENTS/sessions/_templates/kb-merge-SESSION.md
   ```

2. **Update template variables:**
   - Check {{SOURCE_SESSION}} is used correctly
   - Check {{TOPIC}} extraction makes sense
   - Add any missing variables from complete-session script

3. **Update structure:**
   - Ensure Context section is comprehensive
   - Update Acceptance Criteria to match current standards
   - Update Implementation Plan structure
   - Add Notes section if missing

4. **Add worktree references:**
   - Mention that KB merge sessions also use worktrees
   - Reference correct paths in examples

5. **Add read-only note when implemented:**
   - Placeholder for future read-only SESSION.md feature
   - Can be uncommented when that session completes

6. **Test template:**
   - Manually substitute variables
   - Verify resulting SESSION.md is clear
   - Compare to KB merge sessions that exist

### Phase 4: Verify Script Integration (30 minutes)

1. **Test claim-session template usage:**
   - Trace how script uses session-env.template
   - Verify all variables are substituted
   - Check if any hardcoded values should use template
   - Test with actual session claim

2. **Test complete-session template usage:**
   - Trace how script uses kb-merge-SESSION.md
   - Verify all variables are substituted correctly
   - Check sed commands work properly
   - Test with actual KB merge session creation

3. **Document findings:**
   - Any mismatches between scripts and templates
   - Any variables that need adding
   - Any substitution bugs

### Phase 5: Update Documentation References (30 minutes)

1. **Check if templates are documented:**
   - Are templates mentioned in README?
   - Are templates mentioned in REFERENCE?
   - Are template variables documented?

2. **Add documentation if needed:**
   - Document template purpose
   - Document template variables
   - Document how scripts use templates
   - Provide manual usage examples

3. **Update examples:**
   - If any examples reference templates, update them
   - Ensure examples match current template content

### Phase 6: Test End-to-End (45 minutes)

1. **Test session creation workflow:**
   - Claim a test session
   - Check .session-env matches template
   - Verify all variables correct
   - Verify activation works

2. **Test KB merge creation:**
   - Create session with KB learnings
   - Complete session
   - Check generated KB merge SESSION.md
   - Verify template substitution worked
   - Verify structure is correct

3. **Document test results:**
   - What worked correctly
   - What needs fixing
   - Edge cases discovered

### Phase 7: Final Review (30 minutes)

1. **Review all changes:**
   - Read updated templates
   - Check alignment with scripts
   - Verify all acceptance criteria met

2. **Update worklog:**
   - Document all changes made
   - Note any issues found
   - Record test results

3. **Create follow-up sessions if needed:**
   - If script bugs found
   - If documentation gaps found

## Notes

### Template Variables

**session-env.template variables:**
- `{{AGENT_ID}}` - May be deprecated, check usage
- `{{SESSION_SLUG}}` - Session identifier
- `{{USER_NAME}}` - Git user name
- `{{USER_EMAIL}}` - Git user email

**kb-merge-SESSION.md variables:**
- `{{SOURCE_SESSION}}` - Original session that created learnings
- `{{TOPIC}}` - Extracted from learnings file
- `{{AGENT_ID}}` - Agent that created source session (may be deprecated)
- `{{TIMESTAMP}}` - When KB merge session created

### Expected Issues

Based on the recent protocol changes:
- SESSION_BRANCH likely still referenced (should be SESSION_ID)
- Agent-focused language may remain (should be session-focused)
- Paths may reference old structure (before worktrees)
- No mention of worktree workflow
- No mention of read-only SESSION.md protection

### Success Criteria

At completion:
- Templates match current workflow exactly
- Scripts generate correct files from templates
- All variables align with script usage
- Documentation references templates appropriately
- No outdated information remains
- Templates are clear and well-commented

### Potential Challenges

- Templates may be heavily outdated
- Scripts may have drifted from templates
- Variable substitution may have bugs
- Need to balance completeness with simplicity
