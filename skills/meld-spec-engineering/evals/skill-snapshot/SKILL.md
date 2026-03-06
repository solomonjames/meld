---
name: meld-spec-engineering
description: Use when writing acceptance criteria, implementation tasks, or evaluating whether a spec is ready for development — provides Given/When/Then format and ready-for-dev standards
---

# Spec Engineering

This skill defines the standard for **implementation-ready specifications**. Use it when writing acceptance criteria, structuring implementation tasks, or evaluating spec quality.

## Task Format

Every implementation task must be independently actionable:

```
- [ ] Task N: {description}
  - File: {specific file path}
  - Action: {what to change/create}
  - Notes: {implementation details, edge cases}
```

**Task quality rules:**
- Each task is independently actionable by a developer
- File paths are specific (not "update the controller" — name the file)
- Dependency order is correct (migrations before models, models before controllers)
- No task depends on implicit knowledge from another task

## Acceptance Criteria Format

Use Given/When/Then for every significant behavior:

```
**AC-N: {title}**
- Given {precondition}
- When {action}
- Then {expected result}
```

## Coverage Requirements

Every spec must include acceptance criteria covering:

- **Happy path** for every feature
- **Error handling** / validation failures
- **Edge cases** (empty state, boundary values, concurrent access)
- **Authorization checks** where applicable

## Ready-for-Dev Standard

A spec is ready for development when it passes ALL five criteria:

| Criterion | Question |
|-----------|----------|
| **Actionable** | Does every task have a clear file path and specific action? |
| **Logical** | Are tasks ordered by dependency (lowest level first)? |
| **Testable** | Do all ACs follow Given/When/Then and cover happy path + edge cases? |
| **Complete** | Are all investigation results inlined? No placeholders or TBD? |
| **Self-Contained** | Can a fresh agent implement without reading workflow history? |

If any criterion fails, the spec is not ready. Fix the gaps before proceeding.

## Testing Strategy Structure

When specifying testing strategy, include:

- **Unit tests:** What to test in isolation
- **Feature/integration tests:** What end-to-end flows to verify
- **Manual testing:** Any UI flows that need manual verification

## Additional Context Structure

- **Dependencies:** External packages, internal modules, config changes needed
- **Notes:** Pre-mortem risks, limitations, trade-offs, future considerations
