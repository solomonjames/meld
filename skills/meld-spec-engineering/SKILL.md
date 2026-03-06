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

**Example:**
```
- [ ] Task 1: Add avatar attachment to User model
  - File: app/models/user.rb
  - Action: Add `has_one_attached :avatar` with content type validation (JPEG/PNG) and 5MB size limit. Add `avatar_thumbnail` and `avatar_profile` variant helper methods.
  - Notes: Requires `image_processing` gem. Use `resize_to_fill` for variants to handle non-square images without distortion.
```

**Task quality rules:**
- Each task is independently actionable by a developer
- File paths are specific (not "update the controller" — name the file)
- Dependency order is correct (migrations before models, models before controllers)
- No task depends on implicit knowledge from another task
- Each task should be small enough to implement in one sitting

## Acceptance Criteria Format

Use Given/When/Then for every significant behavior:

```
**AC-N: {title}**
- Given {precondition}
- When {action}
- Then {expected result}
```

**Example:**
```
**AC-3: Oversized file rejected**
- Given a signed-in user on the settings page
- When they attempt to upload a PNG file larger than 5MB
- Then the upload is rejected with the error "Avatar must be less than 5MB"
- And the existing avatar (if any) remains unchanged
```

The "Then" clause must describe an **observable, verifiable** outcome — what the user sees, what the API returns, or what state changes. Avoid vague outcomes like "the data is saved" — specify *how* you'd verify it (flash message, response body, database state).

## Coverage Requirements

Every spec must include acceptance criteria covering these categories:

**Happy path** — The primary success flow for each feature or endpoint.

**Error handling** — What happens when input is invalid, operations fail, or external services are unavailable. Cover validation errors, permission denials, and dependency failures.

**Edge cases** — Systematically consider:
- Empty/null/default states (first use, no data, blank inputs)
- Boundary values (exactly at limits, one above, one below)
- State transitions (replacing existing data, toggling between states)
- Concurrent access (two simultaneous requests for the same resource)
- Cleanup/rollback (what happens when an operation partially fails)

**Authorization** — Who can perform this action? Cover unauthenticated users, wrong-role users, and cross-tenant access where applicable.

**Commonly missed ACs** — Check whether these apply to your feature:
- Default/initial state for new users or empty collections
- Idempotency (what if the same action is performed twice?)
- Performance constraints (response time, batch size limits)
- Cascading effects (deleting a parent that has children)

## Evaluating an Existing Spec

When evaluating whether a spec is ready for development, assess each criterion individually:

1. **Walk through each task** — Does it have a specific file path and a concrete action? If any task says "update the relevant file" or "add appropriate fields" without specifics, it fails Actionable.
2. **Check task ordering** — Are migrations before models? Models before controllers? Controllers before views? If ordering is unclear or inverted, it fails Logical.
3. **Audit AC coverage** — Map each AC to the coverage categories above (happy path, error, edge case, authorization). If any category is empty, it fails Testable.
4. **Search for placeholders** — Grep for TBD, TODO, "to be determined", or vague phrases like "appropriate error." If found, it fails Complete.
5. **Fresh-eyes test** — Could someone with no context about this conversation implement the spec? If the spec references "the approach we discussed" or omits tech stack details, it fails Self-Contained.

Render a pass/fail verdict for each criterion with specific evidence.

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

When specifying testing strategy, be concrete about *what* to test, not just *that* to test:

- **Unit tests:** Name the specific file and scenarios. "User model validates avatar content type (JPEG passes, GIF rejected)" is actionable. "Write model tests" is not.
- **Feature/integration tests:** Describe the end-to-end flow being verified and what constitutes success. "Upload a valid JPEG via settings form, verify it renders in the nav bar as 100x100" is specific.
- **Manual testing:** UI flows that automated tests can't fully cover — visual rendering, responsive layout, third-party integrations.

Each test scenario should map back to at least one AC.

## Additional Context Structure

- **Dependencies:** External packages, internal modules, config changes needed
- **Notes:** Pre-mortem risks, limitations, trade-offs, future considerations
