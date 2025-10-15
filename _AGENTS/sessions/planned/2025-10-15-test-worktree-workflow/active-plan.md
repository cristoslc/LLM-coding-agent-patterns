# Active Plan: Test Worktree Workflow

## Current Focus

Session is in **drafting** status. Ready to move to planned/ for agent to claim.

## Task Breakdown

### Phase 1: Setup Test Session
- [ ] Create simple test session in planned/
- [ ] Ensure clean state (no leftover worktrees)
- [ ] Document starting state

### Phase 2: Test Claim Process
- [ ] Run claim-session script
- [ ] Validate session moves to active/
- [ ] Validate .session-env created correctly
- [ ] Validate worktree created at correct path
- [ ] Validate session branch created
- [ ] Validate lock file updated
- [ ] Fix any bugs found
- [ ] Re-test after fixes

### Phase 3: Test Worktree Environment
- [ ] Navigate to worktree
- [ ] Source .session-env
- [ ] Verify environment variables
- [ ] Make test changes
- [ ] Commit with session identity
- [ ] Verify isolation from main repo
- [ ] Fix any issues found

### Phase 4: Test Completion Process
- [ ] Run complete-session script
- [ ] Validate patch generated
- [ ] Validate worktree removed
- [ ] Validate merge to dev
- [ ] Validate lock cleanup
- [ ] Validate session archived
- [ ] Validate branch deleted
- [ ] Fix any bugs found

### Phase 5: Test Error Scenarios
- [ ] Test missing worktree handling
- [ ] Test malformed lock file
- [ ] Test common user mistakes
- [ ] Verify error messages are helpful

### Phase 6: Validate Documentation
- [ ] Test all README examples
- [ ] Test all REFERENCE examples
- [ ] Note any corrections needed
- [ ] Update documentation if needed

### Phase 7: Document Findings
- [ ] Create comprehensive bug list
- [ ] Document all fixes applied
- [ ] Update worklog with results
- [ ] Create follow-up sessions if needed

## Blockers

None - ready to start.

## Next Actions

1. User reviews session and moves to planned/
2. Agent claims session
3. Agent creates simple test session for validation
4. Agent begins Phase 1

## Notes

This is a meta-session - using the workflow to test the workflow. Be methodical and document everything.
