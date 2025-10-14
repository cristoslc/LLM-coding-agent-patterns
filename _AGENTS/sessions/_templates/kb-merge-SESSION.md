# KB Merge Session: {{TOPIC}}

## Context

This session merges knowledge base learnings from a completed session.

- **Source Session**: {{SOURCE_SESSION}}
- **Source Agent**: {{AGENT_ID}}
- **Completed**: {{TIMESTAMP}}
- **Learnings Path**: `_AGENTS/knowledge/sessions/{{SOURCE_SESSION}}/learnings.md`

## Acceptance Criteria

- [ ] Review learnings for quality and accuracy
- [ ] Identify target location(s) in `knowledge/shared/`
- [ ] Merge without duplicating existing content
- [ ] Resolve conflicts with existing KB entries
- [ ] Update KB structure if needed (add sections, reorganize)
- [ ] Preserve source learnings file for reference
- [ ] Document merge decisions in worklog
- [ ] Update KB index/TOC if exists

## Original Implementation Plan

### Phase 1: Review
1. Read source learnings from completed session
2. Read existing KB files that may overlap
3. Identify conflicts, duplications, and gaps

### Phase 2: Merge Strategy
1. Determine merge approach:
   - **Augment**: Add to existing KB section
   - **Create**: Create new KB section
   - **Restructure**: Reorganize KB for better flow
2. Document strategy in worklog

### Phase 3: Execute Merge
1. Apply changes to `knowledge/shared/`
2. Test KB coherence:
   - No broken links
   - Consistent style and formatting
   - Logical organization
3. Update KB index/TOC

### Phase 4: Complete
1. Commit KB changes to session branch
2. Create PR to main with clear KB diff
3. Mark KB session complete

