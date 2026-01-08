# planspec

Design-first development workflow for Claude Code. Transform ideas into validated designs, then into actionable implementation plans with built-in review checkpoints.

## Installation

```
/plugin marketplace add elertan/planspec
/plugin install planspec@planspec
```

## Skills

| Skill | Description |
|-------|-------------|
| `/brainstorm` | Phase 0-5: Feasibility check → Design → Specification |
| `/impl-spec` | Generate phased implementation tasks from design specs |
| `/code-reviewer` | Quality and correctness review at checkpoints |
| `/security-reviewer` | Security-focused review for sensitive code |

## Workflow

```
User: "Let's build X"
         │
         ▼
    /brainstorm
    ├── Phase 0: Feasibility check
    ├── Phase 1: Understand problem
    ├── Phase 2: Define success
    ├── Phase 3: Explore approaches
    ├── Phase 4: Present design
    └── Phase 5: Write spec → planspec/designs/x.md
         │
         ▼
    /impl-spec planspec/designs/x.md
    └── Output: planspec/implementations/x.md
         │
         ▼
    Implementation with checkpoints
    ├── Phase 1: Tasks → /code-reviewer
    ├── Phase 2: Tasks → /code-reviewer
    └── Phase 3: Tasks → /code-reviewer + /security-reviewer
         │
         ▼
       Done
```

## Philosophy

- **Design before code**: Validate approach before investing implementation time
- **Feasibility first**: Quick viability check before full design
- **Implementation-ready specs**: Designs contain enough detail to generate concrete tasks
- **Review gates**: No forward progress with broken or insecure code
- **Meaningful tests**: Test to verify correctness, not for coverage theater

## License

MIT
