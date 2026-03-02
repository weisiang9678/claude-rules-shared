# Project Context

Python monorepo using Polylith architecture. See Core Rules for concepts and conventions.

## Rules & Skills

- **Rules** (`.claude/rules/00-core.md`) — Always-loaded conventions
- **Skills** (`.claude/skills/`) — On-demand procedures, auto-discovered via YAML frontmatter

### Workflow Chain

```
/design
  → approved plan saved to plans/YYYY-MM-DD-<topic>.md
    → /implement
      → phase-scoped verification (verification-before-completion) at each boundary
```

Each skill knows what comes next. Planning hands off to execution. Execution uses verification at every boundary.
