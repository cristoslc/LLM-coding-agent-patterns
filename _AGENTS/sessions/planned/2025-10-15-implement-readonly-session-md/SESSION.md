# Session: Implement Read-Only SESSION.md Protection

## Context

The sessions protocol uses `SESSION.md` to define the original plan, context, and acceptance criteria for each session. However, during active work, agents may be tempted to modify SESSION.md to reflect scope changes, which defeats the purpose of tracking drift.

This session implements **read-only protection** for SESSION.md files when sessions are active, ensuring:
- Original plan/goals are preserved throughout the session
- Scope changes are tracked via worklog.md and subsessions.md
- Drift analysis is possible by comparing original plan to actual work
- Agents are guided to use the correct files for updates

This was identified as a requirement in session `2025-10-15-align-sessions-protocol` but not implemented.

## Acceptance Criteria

### Script Updates
- [ ] `claim-session` sets SESSION.md to read-only (chmod 444) after moving to active/
- [ ] `claim-session` outputs clear message about SESSION.md being read-only
- [ ] `complete-session` unlocks SESSION.md (chmod 644) before archiving
- [ ] `complete-session` allows final updates to SESSION.md if needed
- [ ] Both scripts handle file permission errors gracefully

### Documentation Updates
- [ ] SESSIONS-README.md explains read-only SESSION.md protection
- [ ] SESSIONS-README.md documents purpose: preserve original plan to track drift
- [ ] SESSIONS-README.md explains that updates go to worklog.md, active-plan.md, subsessions.md
- [ ] SESSIONS-REFERENCE.md includes read-only SESSION.md in workflow examples
- [ ] Troubleshooting section covers file permission issues
- [ ] Session workflow diagrams updated if needed

### Template Updates
- [ ] Session templates include note about SESSION.md being read-only
- [ ] Templates remind agents to use worklog/active-plan for updates
- [ ] KB merge template reflects this constraint if applicable

### Testing & Validation
- [ ] Test claim-session sets permissions correctly
- [ ] Test complete-session unlocks correctly
- [ ] Verify clear error when trying to edit read-only SESSION.md
- [ ] Test manual override path works (chmod 644 if really needed)
- [ ] Document override process in troubleshooting

### Edge Cases
- [ ] Handle sessions that existed before this feature
- [ ] Handle manual session moves (without script)
- [ ] Document when it's acceptable to override (rare cases)
- [ ] Provide escape hatch for emergencies

## Implementation Plan

### Phase 1: Update claim-session Script (30 minutes)

1. **Add permission change after move to active:**
   ```bash
   # After moving session to active and creating .session-env
   chmod 444 _AGENTS/sessions/active/$SESSION_SLUG/SESSION.md
   git add _AGENTS/sessions/active/$SESSION_SLUG/SESSION.md
   git commit -m "[$SESSION_SLUG] Set SESSION.md read-only"
   ```

2. **Add output message:**
   ```bash
   echo "📝 SESSION.md is now read-only to preserve original plan"
   echo "   Use worklog.md and active-plan.md for updates during session"
   ```

3. **Add error handling:**
   - Check if chmod succeeds
   - Provide helpful error if permission change fails
   - Continue even if chmod fails (not critical)

### Phase 2: Update complete-session Script (30 minutes)

1. **Add unlock before archiving:**
   ```bash
   # Before moving to completed
   chmod 644 _AGENTS/sessions/active/$SESSION_SLUG/SESSION.md
   echo "📝 SESSION.md unlocked for final updates"
   ```

2. **Optional: Prompt for final updates:**
   ```bash
   echo ""
   echo "SESSION.md is now writable. Add final notes if needed:"
   echo "  nano _AGENTS/sessions/active/$SESSION_SLUG/SESSION.md"
   echo ""
   read -p "Press Enter to continue with archival..."
   ```

3. **Set back to read-only in completed:**
   ```bash
   # After moving to completed
   chmod 444 _AGENTS/sessions/completed/$SESSION_SLUG/SESSION.md
   ```

### Phase 3: Update SESSIONS-README.md (45 minutes)

1. **Add new section: "SESSION.md Protection"**
   - Explain the purpose (drift tracking)
   - Document that SESSION.md becomes read-only in active/
   - List where updates should go instead
   - Explain unlock on completion

2. **Update manual workflow examples:**
   - Add chmod commands to claim process
   - Add chmod commands to completion process
   - Show proper error messages

3. **Update Quick Start:**
   - Mention SESSION.md is read-only
   - Point to worklog/active-plan for updates

### Phase 4: Update SESSIONS-REFERENCE.md (45 minutes)

1. **Update "Starting a Session" section:**
   - Include chmod in manual claim process
   - Show read-only confirmation

2. **Update "Completing a Session" section:**
   - Include unlock step
   - Show optional final update process

3. **Add troubleshooting section:**
   - "How to edit SESSION.md if really needed"
   - "Permission denied when trying to edit SESSION.md" (expected!)
   - Manual override: `chmod 644 SESSION.md` (with warnings)

### Phase 5: Update Templates (30 minutes)

1. **Add note to session templates:**
   ```markdown
   ## IMPORTANT: SESSION.md Read-Only Protection
   
   This file becomes **read-only** when the session moves to active/.
   
   - **DO NOT** modify this file during active work
   - **DO** use worklog.md for progress tracking
   - **DO** use active-plan.md for task updates
   - **DO** use subsessions.md for scope changes
   
   Purpose: Preserves original plan to enable drift analysis.
   ```

2. **Update session-env.template if needed:**
   - Add reminder message about read-only SESSION.md

3. **Check kb-merge template:**
   - Verify it follows same pattern

### Phase 6: Test End-to-End (1 hour)

1. **Test claim process:**
   - Run claim-session on test session
   - Verify SESSION.md is read-only (ls -la shows r--r--r--)
   - Try to edit SESSION.md (should fail)
   - Verify error message is clear

2. **Test completion process:**
   - Run complete-session on test session
   - Verify SESSION.md is unlocked before archival
   - Add test note to SESSION.md
   - Verify SESSION.md is read-only in completed/

3. **Test edge cases:**
   - Session moved manually (without script)
   - Pre-existing active sessions
   - Manual override (chmod 644)

4. **Document test results:**
   - What worked
   - What failed
   - What needs adjustment

### Phase 7: Update Documentation Examples (30 minutes)

1. **Review all code examples:**
   - Add chmod commands where needed
   - Update expected output
   - Show read-only markers in ls output

2. **Update flowcharts if needed:**
   - Add read-only indicator in diagrams

3. **Add FAQ entry:**
   - "Why is SESSION.md read-only?"
   - "How do I track scope changes?"

## Notes

### Purpose of Read-Only SESSION.md

**Drift Analysis:** By keeping SESSION.md unchanged during work:
- Can compare original plan vs. actual work done
- Can identify scope creep
- Can learn from planning inaccuracies
- Can improve future session planning

**Update Channels:**
- `worklog.md` - What happened, when, and why
- `active-plan.md` - Current tasks and next steps
- `subsessions.md` - Scope additions (creates new sessions)

### When to Override

**Acceptable reasons to override (rare):**
- Critical error in acceptance criteria (blocks completion)
- Major context error that invalidates the session
- Security issue in documented approach

**How to override:**
```bash
chmod 644 _AGENTS/sessions/active/{slug}/SESSION.md
# Make critical fix
git add SESSION.md
git commit -m "[{slug}] OVERRIDE: Fix critical SESSION.md error"
chmod 444 _AGENTS/sessions/active/{slug}/SESSION.md
# Document reason in worklog.md
```

### Expected Challenges

- Agents may initially try to edit SESSION.md (this is good - the error teaches them!)
- Some editors may not show clear read-only errors
- Need balance between protection and flexibility
- Edge cases with manual session management

### Success Criteria

At completion:
- SESSION.md becomes read-only in active/ sessions
- Scripts handle permissions correctly
- Documentation explains the why and how
- Troubleshooting covers common issues
- Tests prove it works
