# Documentation Directory Structure

**Complete documentation repository structure (34 files across 4 categories):**

```
~/.claude/docs/
├── workflows/           (3 files - Process flows)
│   ├── tdd-cycle.md
│   ├── agent-collaboration.md
│   └── code-review-process.md
├── patterns/            (22 files - Domain-specific patterns)
│   ├── backend/         (4 files: api-design, database-design, database-integration, lambda-patterns)
│   ├── react/           (3 files: component-patterns, hooks, testing)
│   ├── typescript/      (5 files: schemas, strict-mode, type-vs-interface, branded-types, effect-ts)
│   ├── security/        (2 files: authentication, owasp-top-10)
│   ├── refactoring/     (3 files: common-patterns, dry-semantics, when-to-refactor)
│   └── performance/     (2 files: database-optimization, react-optimization)
├── references/          (8 files - Quick lookups)
│   ├── standards-checklist.md
│   ├── code-style.md
│   ├── agent-quick-ref.md
│   ├── working-with-claude.md
│   ├── http-status-codes.md
│   ├── severity-levels.md
│   ├── indexing-strategies.md
│   └── normalization.md
└── examples/            (4 files - Walkthroughs)
    ├── tdd-complete-cycle.md
    ├── schema-composition.md
    ├── factory-patterns.md
    └── refactoring-journey.md
```

All agents have access to these docs via the Read tool.

## How to Reference Documentation

**Pattern**: `@~/.claude/docs/[category]/[filename].md`

**Example Commands to Access Documentation:**

```bash
# Read complete TDD process
Read file: ~/.claude/docs/workflows/tdd-cycle.md

# Read agent collaboration workflows
Read file: ~/.claude/docs/workflows/agent-collaboration.md

# Read Zod schema patterns
Read file: ~/.claude/docs/patterns/typescript/schemas.md

# Read backend API design guide
Read file: ~/.claude/docs/patterns/backend/api-design.md

# Read React component patterns
Read file: ~/.claude/docs/patterns/react/component-patterns.md

# Read code style standards
Read file: ~/.claude/docs/references/code-style.md

# Read factory pattern examples
Read file: ~/.claude/docs/examples/factory-patterns.md
```

## Categories Quick Reference

- **workflows/** (3 files) - TDD cycle, agent collaboration, code review process
- **patterns/backend/** (4 files) - API design, database design, database integration, Lambda patterns
- **patterns/react/** (3 files) - Component patterns, hooks, testing
- **patterns/typescript/** (5 files) - Schemas, strict mode, type vs interface, branded types, Effect-TS
- **patterns/security/** (2 files) - Authentication, OWASP Top 10
- **patterns/refactoring/** (3 files) - Common patterns, DRY semantics, when to refactor
- **patterns/performance/** (2 files) - Database optimization, React optimization
- **references/** (8 files) - Standards checklist, code style, HTTP status codes, indexing strategies, normalization, severity levels, agent quick ref, working with Claude
- **examples/** (4 files) - TDD complete cycle, schema composition, factory patterns, refactoring journey
