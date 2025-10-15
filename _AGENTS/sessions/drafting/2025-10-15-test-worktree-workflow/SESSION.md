# Session: Test Worktree Workflow End-to-End

## Context

The sessions protocol was just migrated to use `git worktree` for session isolation (session `2025-10-15-align-sessions-protocol`). However, the updated `claim-session` and `complete-session` scripts have **never been tested** in a real end-to-end scenario.

This is a **CRITICAL** session because:
- The worktree workflow is the foundation of multi-agent concurrent work
- Untested scripts may have bugs that block agents from working
- Path references, environment variables, and commands need validation
- Error handling needs to be verified in real scenarios
- Session lock management needs to work correctly

## Acceptance Criteria

### Claim Process Testing
- [ ] `claim-session` script executes without errors
- [ ] Session moves from `planned/` to `active/` correctly
- [ ] `.session-env` file is created with correct variables
- [ ] Worktree is created at `.worktrees/{session-slug}/`
- [ ] Session branch is created correctly
- [ ] Session is added to `.agents/sessions.lock`
- [ ] All commits are made with proper messages
- [ ] Script output is clear and helpful
- [ ] Activation instructions are correct

### Worktree Environment Testing
- [ ] Can navigate to worktree directory
- [ ] Can source `.session-env` from worktree
- [ ] Environment variables are set correctly
- [ ] Can make code changes in worktree
- [ ] Can commit changes with session identity
- [ ] Changes are isolated from main repo
- [ ] Main repo stays on base branch (dev)

### Completion Process Testing
- [ ] `complete-session` script executes without errors
- [ ] Patch file is generated correctly
- [ ] Worktree is removed successfully
- [ ] Session merges to dev branch
- [ ] Session is removed from `.agents/sessions.lock`
- [ ] Session moves to `completed/` directory
- [ ] Session branch is deleted
- [ ] All artifacts are preserved correctly

### Error Handling Testing
- [ ] Graceful handling if worktree doesn't exist
- [ ] Graceful handling if session lock is malformed
- [ ] Clear error messages for common mistakes
- [ ] Recovery instructions provided when errors occur

### Documentation Validation
- [ ] All documented commands work as written
- [ ] All path references are correct
- [ ] Activation instructions match reality
- [ ] Examples in README work correctly
- [ ] Examples in REFERENCE work correctly

## Implementation Plan

### Phase 1: Setup Test Session (15 minutes)

1. **Create test session in planned/**
   - Simple test session with clear objectives
   - No actual code changes needed, just documentation updates
   - Minimal scope to focus on workflow validation

### Phase 2: Test Claim Process (30 minutes)

1. **Run claim-session script**
   ```bash
   ./_AGENTS/sessions/_bin/claim-session <test-session-slug>
   ```

2. **Validate each step:**
   - Check session lock file updated
   - Check session moved to active/
   - Check .session-env file created and contents correct
   - Check worktree created at correct path
   - Check session branch created
   - Check git log shows proper commits

3. **Document any issues:**
   - Note exact error messages
   - Note unexpected behavior
   - Note missing or incorrect output
   - Note path issues

4. **Fix issues immediately:**
   - Update claim-session script if bugs found
   - Update documentation if instructions wrong
   - Test fixes work correctly

### Phase 3: Test Worktree Environment (30 minutes)

1. **Navigate and activate:**
   ```bash
   cd .worktrees/<test-session-slug>
   source ../../_AGENTS/sessions/active/<test-session-slug>/.session-env
   ```

2. **Verify environment:**
   - Echo all environment variables
   - Verify git identity is correct
   - Check prompt shows session name

3. **Make test changes:**
   - Add a test file or edit documentation
   - Commit with session identity
   - Verify commit appears in git log with correct author

4. **Verify isolation:**
   - Check main repo is still on dev branch
   - Check changes only in worktree
   - Verify no files in main repo changed

5. **Document any issues:**
   - Path problems
   - Environment variable issues
   - Commit attribution problems

### Phase 4: Test Completion Process (45 minutes)

1. **Run complete-session script:**
   ```bash
   cd <repo-root>
   ./_AGENTS/sessions/_bin/complete-session <test-session-slug>
   ```

2. **Validate each step:**
   - Check patch file generated and valid
   - Check worktree removed
   - Check merge to dev successful
   - Check session removed from lock file
   - Check session in completed/ directory
   - Check session branch deleted
   - Check all artifacts present

3. **Document any issues:**
   - Script errors
   - Missing steps
   - Incorrect behavior
   - Path problems

4. **Fix issues immediately:**
   - Update complete-session script
   - Update documentation
   - Test fixes work

### Phase 5: Test Error Scenarios (30 minutes)

1. **Test missing worktree:**
   - Manually remove a worktree
   - Run complete-session
   - Verify graceful handling

2. **Test malformed lock file:**
   - Add invalid entry to sessions.lock
   - Try to claim session
   - Verify error handling

3. **Test common mistakes:**
   - Try to run claim-session when already claimed
   - Try to complete non-existent session
   - Verify error messages are helpful

### Phase 6: Validate Documentation (30 minutes)

1. **Test README examples:**
   - Follow Quick Start exactly as written
   - Run manual process examples
   - Verify all commands work

2. **Test REFERENCE examples:**
   - Try detailed implementation examples
   - Test manual claim process
   - Test manual completion process

3. **Document corrections needed:**
   - Note any incorrect commands
   - Note any wrong paths
   - Note any missing steps

### Phase 7: Document Findings (30 minutes)

1. **Create bug list:**
   - All issues found with severity
   - All fixes applied
   - All documentation corrections needed

2. **Update worklog:**
   - What was tested
   - What worked
   - What failed
   - What was fixed

3. **Create follow-up sessions if needed:**
   - For major bugs that need deeper fixes
   - For documentation improvements
   - For additional testing scenarios

## Notes

### Testing Strategy

- Use a real test session, not mock data
- Test in a clean state (no leftover worktrees)
- Document every step and observation
- Fix bugs immediately and re-test
- Don't skip error scenarios

### Expected Issues

Based on the implementation, potential issues:
- Path references may be incorrect (sessions/ vs _AGENTS/sessions/)
- Environment variable names may be inconsistent
- sed commands may not work on all systems (macOS vs Linux)
- Worktree removal may fail if uncommitted changes exist
- Lock file cleanup may fail if file permissions wrong

### Success Criteria

At the end of this session:
- A complete session has been claimed, worked on, and completed using new workflow
- All scripts work correctly
- All documentation is accurate
- Any bugs found have been fixed
- Confidence that other agents can use the workflow successfully

### Potential Challenges

- Finding bugs may require multiple iterations
- Documentation may need significant updates
- Scripts may need edge case handling
- Error messages may need improvement
