# Active Plan: Implement Read-Only SESSION.md

## Current Focus

Session is in **planned** status. Ready for agent to claim.

## Task Breakdown

### Phase 1: Update claim-session Script
- [ ] Add chmod 444 after moving to active/
- [ ] Add output message about read-only status
- [ ] Add error handling for permission failures
- [ ] Test changes work correctly

### Phase 2: Update complete-session Script
- [ ] Add chmod 644 before archiving
- [ ] Add optional prompt for final updates
- [ ] Set back to read-only in completed/
- [ ] Test changes work correctly

### Phase 3: Update SESSIONS-README.md
- [ ] Add new section explaining read-only protection
- [ ] Update manual workflow examples with chmod
- [ ] Update Quick Start section
- [ ] Document purpose (drift tracking)

### Phase 4: Update SESSIONS-REFERENCE.md
- [ ] Update "Starting a Session" with chmod
- [ ] Update "Completing a Session" with unlock
- [ ] Add troubleshooting for permission issues
- [ ] Document manual override process

### Phase 5: Update Templates
- [ ] Add read-only note to session templates
- [ ] Update session-env.template if needed
- [ ] Check kb-merge template

### Phase 6: Test End-to-End
- [ ] Test claim-session sets permissions
- [ ] Test editing SESSION.md fails with clear error
- [ ] Test complete-session unlocks
- [ ] Test read-only in completed/
- [ ] Test manual override works
- [ ] Document test results

### Phase 7: Update Examples
- [ ] Add chmod to all code examples
- [ ] Update expected output in examples
- [ ] Add FAQ entry
- [ ] Update flowcharts if needed

## Blockers

None - ready to start.

## Next Actions

1. Agent claims session
3. Agent begins with script updates
4. Agent tests incrementally

## Notes

Focus on clear communication about WHY SESSION.md is read-only. Agents should understand it's for drift tracking, not arbitrary restriction.

Provide clear override path for rare cases where it's genuinely needed.
