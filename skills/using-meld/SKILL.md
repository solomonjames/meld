---
name: using-meld
description: Use when starting any conversation - establishes when to use MELD methodology skills for structured product development, spec engineering, and adversarial review
---

# Using MELD

MELD (Methodology for Engineering Lifecycle & Development) provides a complete methodology for structured product development — from planning through execution to review. It includes:

- **Complexity routing** — Match approach depth to problem complexity
- **Spec engineering** — Structured Given/When/Then specifications with ready-for-dev standards
- **Adversarial review** — Information-asymmetric code review using subagents
- **Execution skills** — TDD, debugging, verification, parallel agents, worktrees

## When to Use MELD Skills

| Situation | MELD Skill | Pairs With |
|-----------|-----------|------------|
| User describes a feature or asks "build X" | `meld:meld-complexity-assessment` | — |
| Feature needs a spec before coding | `meld:meld-quick-spec` | — |
| Ready to implement (with or without spec) | `meld:meld-quick-dev` | TDD built-in (`meld:meld-tdd`) |
| Writing acceptance criteria | `meld:meld-spec-engineering` | — |
| Bug needs investigation | `meld:meld-debugging` | `meld:meld-tdd` |
| Claiming work is complete | `meld:meld-verification` | — |
| 2+ independent tasks | `meld:meld-parallel-agents` | — |
| Starting isolated work on a ticket | `meld:meld-worktrees` | Built into quick-dev |
| Code needs cleanup after implementation | `meld:meld-code-simplifier` | `meld:meld-verification` |
| Implementation complete, need review | `meld:meld-adversarial-review` | `meld:meld-verification` |
| Implementation complete, capture learnings | `meld:meld-retrospective` | Built into quick-dev |
| Need an output template (tech-spec, PRD, etc.) | `meld:meld-artifact-templates` | — |

## When NOT to Use MELD Skills

- **Simple bug fixes with obvious cause** — Go straight to TDD. If cause is NOT obvious, use `meld:meld-debugging`
- **One-line changes** — Just do it
- **Exploration/research** — Use exploration tools directly
- **Refactoring with no behavior change** — Execution skills suffice

## Available Skills

### Flow Skills (invoke these to run a workflow)
- **`meld-quick-spec`** — Conversational spec engineering: understand → investigate → generate → review. Produces a ready-for-dev tech spec.
- **`meld-quick-dev`** — Implementation flow: setup & mode detection → context gathering → execution → verify & self-check → adversarial review & resolution → retrospective capture.
- **`meld-complexity-assessment`** — Evaluate complexity signals and route to the right depth of planning.

### Methodology Skills (invoked by flow skills or standalone)
- **`meld-tdd`** — Test-driven development: Iron Law, Red-Green-Refactor cycle, rationalizations, red flags. Built into quick-dev's execution loop.
- **`meld-debugging`** — Systematic debugging: 4-phase root cause methodology (investigate → analyze → hypothesize → implement). Activated by quick-dev halt conditions. Includes 3 supporting docs (root-cause-tracing, defense-in-depth, condition-based-waiting).
- **`meld-code-simplifier`** — Code simplification pass: subagent reviews modified code for clarity, consistency, and maintainability. Can be invoked standalone or as part of a review workflow.
- **`meld-verification`** — Verification before completion: 5-step gate function requiring fresh evidence for every completion claim. Invoked at quick-dev Phases 4 and 5.
- **`meld-parallel-agents`** — Parallel agent dispatch: identify independent domains, craft focused prompts, dispatch via Task tool, review and integrate. Optional in quick-dev Phase 3.
- **`meld-worktrees`** — Git worktree creation with ticket-based branch naming. Auto-creates worktrees for beads tickets. Invoked at quick-dev Phases 1 and 5.
- **`meld-spec-engineering`** — Given/When/Then acceptance criteria format and ready-for-dev standards.
- **`meld-adversarial-review`** — Code review with information asymmetry using subagents.
- **`meld-retrospective`** — Post-implementation retrospective: categorizes findings patterns, measures spec accuracy, captures estimation signals, persists learnings to project memory. Built into quick-dev Phase 6.

### Reference Skills
- **`meld-artifact-templates`** — Index of 8 output templates (tech-spec, story, PRD, product brief, architecture decision, epics, UX design, sprint status).

## Slash Commands

- **`/quick-spec [ticket-id]`** — Start the quick-spec flow (optionally against a beads ticket)
- **`/quick-dev [ticket-id]`** — Start the quick-dev flow (optionally against a beads ticket)
- **`/assess-complexity`** — Run complexity assessment

## Self-Contained Methodology

MELD owns all its methodology and execution skills — from planning (spec engineering, complexity routing) through execution (TDD, debugging, verification, parallel agents, worktrees) to review (adversarial review). No external plugin dependencies are required.
