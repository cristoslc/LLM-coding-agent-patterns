# RFC: Comparing Session/Subsession Process with BD Integration

**Status:** Draft
**Created:** 2025-10-15
**Updated:** 2025-10-15

## Summary

This RFC proposes a comparison and potential integration between the current session-based development process and Behavior Driven Development (BDD) methodologies. The analysis examines how BDD practices can enhance the existing session workflow while maintaining the benefits of git-based coordination and agent collaboration.

## Problem Statement

The current session process provides excellent isolation and traceability but may benefit from more structured specification and testing approaches. BDD offers systematic ways to capture requirements and validate implementations, but needs adaptation to work within the distributed, git-coordinated session model.

## Current Implementation

### Session Process Overview

The current system uses a four-state workflow:

1. **Drafting** - Session definition and planning phase
2. **Planned** - Ready for agent claiming via git atomic operations
3. **Active** - Isolated worktrees with session-scoped git identity
4. **Completed** - Knowledge base merging and main branch integration

### Key Features

- **Git Coordination**: Atomic session claiming via `.agents/sessions.lock`
- **Namespace Isolation**: Separate worktrees and branches per session
- **Agent Identity**: Session-scoped git author/committer attribution
- **Knowledge Management**: Two-phase KB capture (session-scoped → canonical)
- **Traceability**: Full git history with session-prefixed commits

## Proposed Solution

### BD Integration Points

Integrate BDD practices into the session workflow:

1. **Specification Phase**: Extend drafting state with BDD scenario capture
2. **Development Phase**: BDD scenario validation within active sessions
3. **Verification Phase**: Automated BDD test execution in worktrees
4. **Documentation Phase**: BDD scenarios as living documentation in KB

## Comparison Analysis

### Current Session Process vs BD Integration

| Aspect | Current Session Process | BD Integration | Hybrid Approach |
|--------|------------------------|-----------------|-----------------|
| **Requirements** | SESSION.md (narrative) | BDD scenarios (structured) | SESSION.md + BDD scenarios |
| **Planning** | active-plan.md (manual) | Scenario outlines (executable) | Scenario-driven active-plan.md |
| **Development** | Code + worklog updates | TDD with BDD context | Session-scoped BDD execution |
| **Testing** | Manual verification | Automated scenario testing | Git-coordinated BDD runs |
| **Documentation** | KB learnings (post-hoc) | Living specifications (BDD) | BDD scenarios in KB merge |
| **Coordination** | Git-based session locking | Scenario state tracking | Session-scoped scenario status |
| **Traceability** | Session-prefixed commits | Scenario-to-commit mapping | Session + scenario attribution |

### Detailed Comparison

#### Requirements Capture

**Current Process:**
```markdown
## Acceptance Criteria

- [ ] User can authenticate with JWT
- [ ] Password validation works
- [ ] Refresh tokens implemented
```

**BD Integration:**
```gherkin
Feature: User Authentication
  Scenario: Successful login
    Given user provides valid credentials
    When login request is made
    Then JWT token is returned
    And refresh token is provided
```

**Hybrid:**
- SESSION.md contains narrative context + BDD scenarios
- Scenarios serve as executable acceptance criteria
- Worklog tracks scenario implementation status

#### Development Workflow

**Current Process:**
- Agent claims session atomically via git
- Works in isolated worktree
- Commits with `[session-id]` prefix
- Updates worklog.md manually

**BD Integration:**
- BDD scenarios guide implementation
- TDD cycle within session context
- Scenario steps drive code structure
- Automated test execution in worktree

**Hybrid:**
- Session activation includes BDD tooling setup
- Scenario validation integrated with session workflow
- KB learnings include BDD pattern insights

#### Knowledge Management

**Current Process:**
- Session-scoped learnings in `_AGENTS/knowledge/sessions/{session}/`
- KB merge sessions for canonical documentation
- Manual knowledge organization

**BD Integration:**
- BDD scenarios as living documentation
- Automated scenario-to-implementation mapping
- Specification by example methodology

**Hybrid:**
- BDD scenarios included in session learnings
- KB merge sessions incorporate scenario patterns
- Automated scenario maintenance in canonical KB

## Implementation Plan

### Phase 1: Foundation (Week 1-2)

- [ ] Install BDD tooling (Cucumber/Gherkin parser)
- [ ] Create BDD scenario templates for sessions
- [ ] Update session creation scripts with BDD support
- [ ] Document BDD scenario format for agents

### Phase 2: Integration (Week 3-4)

- [ ] Modify session activation to include BDD environment
- [ ] Update worktree setup for BDD test execution
- [ ] Integrate scenario validation in session workflow
- [ ] Create BDD scenario status tracking

### Phase 3: Enhancement (Week 5-6)

- [ ] Add automated scenario validation in CI/CD
- [ ] Implement scenario-to-commit mapping
- [ ] Create BDD reporting in session completion
- [ ] Update KB merge process for BDD scenarios

### Phase 4: Optimization (Week 7-8)

- [ ] Performance optimization for BDD execution
- [ ] Cross-session scenario dependency management
- [ ] BDD scenario maintenance automation
- [ ] Agent training and documentation updates

## Migration Strategy

### Backward Compatibility

- Existing sessions continue to work unchanged
- BDD features opt-in during session creation
- Gradual rollout starting with new sessions
- KB migration handles legacy vs BDD content

### Risk Mitigation

- Start with pilot sessions using BD integration
- Monitor performance impact of BDD tooling
- Maintain fallback to current process if needed
- Agent feedback drives iterative improvements

## Success Criteria

### Technical Metrics

- [ ] BDD scenario execution time < 5 minutes per session
- [ ] Scenario success rate > 95% for completed sessions
- [ ] Zero breaking changes to existing session workflow
- [ ] BDD scenarios included in 80%+ of new sessions

### User Experience

- [ ] Agent onboarding time reduced by 30%
- [ ] Session handoff clarity improved via BDD scenarios
- [ ] KB quality enhanced with executable specifications
- [ ] Cross-agent collaboration improved

## Risk Assessment

### Technical Risks

**High Risk:**
- BDD tooling performance impact on large codebases
- Git worktree complexity with BDD test environments
- Scenario maintenance overhead in fast-moving projects

**Medium Risk:**
- Learning curve for agents adopting BDD practices
- Scenario format standardization across different domains
- Integration complexity with existing CI/CD pipelines

**Mitigation Strategies:**
- Performance benchmarking before full rollout
- Gradual feature introduction with opt-in approach
- Comprehensive documentation and training materials

### Process Risks

**Medium Risk:**
- Over-specification leading to reduced development velocity
- BDD scenarios becoming outdated maintenance burden
- Agent resistance to structured specification approach

**Mitigation Strategies:**
- Clear guidelines for when BDD scenarios add value
- Automated scenario maintenance tooling
- Regular retrospective reviews of BDD adoption

## Open Questions

1. **BDD Tool Selection**: Which BDD framework best integrates with the session model? (Cucumber.js, Jest BDD, custom solution?)

2. **Scenario Scope**: What granularity of scenarios works best within session timeframes? (epic-level vs task-level)

3. **Cross-Session Dependencies**: How to handle BDD scenarios that span multiple sessions?

4. **Performance Impact**: What's the computational overhead of BDD execution in worktrees?

5. **Agent Adoption**: How to encourage consistent BDD scenario writing across different agent types?

6. **KB Integration**: How should BDD scenarios be represented in the canonical knowledge base?

## References

- [SESSIONS-README.md](./_AGENTS/sessions/SESSIONS-README.md) - Current session protocol documentation
- [Behavior Driven Development Overview](https://cucumber.io/docs/bdd/) - BDD methodology fundamentals
- [Git Worktrees Documentation](https://git-scm.com/docs/git-worktree) - Git worktree isolation mechanism
- [Session-Based Test Management](https://en.wikipedia.org/wiki/Session-based_testing) - Testing methodology parallels