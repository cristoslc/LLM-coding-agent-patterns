# LLM Coding Agent Patterns

A comprehensive framework for organizing and managing LLM-powered coding agent sessions, featuring structured workflows, persona-driven development, and trunk-based development practices.

## Overview

This project provides a systematic approach to managing coding agent sessions, ensuring consistent documentation, progress tracking, and knowledge capture. It's designed to help developers and AI agents work together more effectively by providing clear patterns and workflows.

## Quickstart

1. Clone or download+copy the `_AGENTS` directory into your project.
2. Prompt your local agent (cursor, Roo Code, Continue.dev, aider, Claude code, codex, etc.): `Create a new session folder based on @_AGENTS/sessions/SESSIONS-README.md with purpose: {{your-purpose}}`
3. Edit the session file like you would a Jira ticket for a junior dev.
4. Tell your local agent `Implement @YYYY-MM-DD-session-slug/SESSION.md.`

_No muss, no fuss, no install script to run._

**Example:**
> _User Prompt:_ Create a new session folder based on @SESSIONS-README.md with purpose: resolve errors identified in the attached error.log
>
> _Agent:_ # creates folder `_AGENTS/sessions/2025-11-01-resolve-logged-errors`
>
> _User Prompt:_ Confirm your understanding of `@_AGENTS/sessions/2025-11-01-resolve-logged-errors/SESSION.md` and begin implementing it.


## Key Features

### 🎯 Structured Session Management
- **State-based workflow**: Sessions move through planned → active → completed/abandoned states
- **Comprehensive documentation**: Each session includes context, acceptance criteria, and implementation plans
- **Progress tracking**: Detailed worklogs and dynamic implementation plans

### 👥 Persona-Driven Development
- **Dana**: Data Architect focused on data strategy and impact
- **Oscar**: Software Architect with Python expertise, advocate for elegant solutions
- **Argo**: Corporate perspective that identifies risks and assumptions

### 🔄 Trunk-Based Development Integration
- **Session branches**: Each session gets its own branch (`session/YYYY-MM-DD-descriptive-slug`)
- **Frequent merges**: Sub-sessions are merged to main regularly
- **Clean history**: Squash merges maintain logical units of work

### 📋 Sub-session Orchestration
- **Epic breakdown**: Large sessions are broken into manageable sub-sessions
- **TDD integration**: Red-Green-Refactor cycles with proper documentation
- **Acceptance criteria**: Clear success metrics for each sub-session

## Project Structure

```
_AGENTS/
├── personas/
│   └── personas.md          # Development personas for different perspectives
└── sessions/
    ├── active/              # Currently active sessions
    ├── completed/           # Finished sessions
    ├── planned/             # Future sessions
    ├── abandoned/           # Cancelled/incomplete sessions
    ├── _templates/          # Jinja2 templates for session documents
    └── SESSIONS-README.md   # Detailed session management documentation
```

## Session Lifecycle

```mermaid
flowchart TD
    Start["Session Request"] --> CheckExisting["Check Existing Sessions"]
    CheckExisting --> PlannedState["Planned State"]
    PlannedState --> ActiveState["Active State"]
    ActiveState --> CompletedState["Completed State"]
    ActiveState --> AbandonedState["Abandoned State"]
```

## Templates System

The `_templates/` directory contains Jinja2 templates for generating consistent session documents. These templates use variable substitution to create personalized documents for different session types.

### Available Templates

- **`rfc.md.j2`**: Request for Comments (RFC) document template
- **`kb-merge-SESSION.md.j2`**: Knowledge base merge session template
- **`session-env.j2`**: Session environment configuration template

### Using Templates

Templates use Jinja2 syntax with variables enclosed in double curly braces:
```jinja2
{{ VARIABLE_NAME }}
```

Comments and documentation are enclosed in Jinja2 comment blocks:
```jinja2
{# This is a template comment #}
```

### Template Variables

Each template includes comprehensive documentation of:
- **Purpose**: What the template is used for
- **Variables**: Required and optional variables
- **Usage**: How to use the template
- **Examples**: Sample values for variables

### Rendering Templates

To render a template, you need:
1. A Jinja2 template engine (Python's `jinja2` package)
2. A context dictionary with variable values
3. Template rendering logic

Example Python code:
```python
from jinja2 import Template

with open('_templates/rfc.md.j2', 'r') as f:
    template = Template(f.read())

context = {
    'TITLE': 'Unifying Access Control',
    'AUTHORS': 'Jordan Lee (Platform Engineering)',
    'DATE': 'October 14, 2025',
    # ... other variables
}

rendered = template.render(**context)
```

## Getting Started

1. **Create a new session**: Follow the naming convention `YYYY-MM-DD-descriptive-slug`
2. **Document context**: Define what the session is about and success criteria
3. **Break into sub-sessions**: Identify manageable chunks of work
4. **Track progress**: Use worklogs and dynamic implementation plans
5. **Apply personas**: Leverage Dana, Oscar, and Argo for different perspectives

## Session Documentation

Each session includes:
- **SESSION.md**: Core documentation with context and acceptance criteria
- **worklog.md**: Progress tracking with timestamps and decisions
- **active-plan.md**: Dynamic implementation plan with task lists
- **subsessions.md**: Sub-session organization and status
- **patch file**: Final patch generated after completion

## Best Practices

- **Update frequently**: Keep documentation current with work progress
- **Document decisions**: Provide context for future agents and developers
- **Be honest**: Record failures and lessons learned
- **Clean up**: Remove temporary files when sessions complete
- **Use specific commits**: Avoid `git add .` - be explicit about what you're committing

## Contributing

This framework is designed to evolve with your needs. Feel free to:
- Add new personas for different perspectives
- Extend session documentation templates
- Improve workflow patterns
- Share successful session examples

## License

This project is open source and available under the [MIT License](LICENSE).
