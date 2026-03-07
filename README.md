# MELD

MELD (Methodology for Engineering Lifecycle & Development) methodology as a **Claude Code plugin**. Adds structured product development — complexity routing, spec engineering, adversarial review — on top of your existing development workflow.

Self-contained methodology with built-in execution skills (TDD, debugging, verification, parallel agents, worktrees).

## Install

### As a Claude Code plugin (recommended)

```bash
# Add the marketplace
/plugin marketplace add https://github.com/solomonjames/meld

# Install the plugin
/plugin install meld
```

### From local clone

```bash
git clone https://github.com/solomonjames/meld.git
cd meld
/plugin marketplace add .
/plugin install meld
```

## Quick Start

### Slash commands

```
/quick-spec          # Conversational spec engineering — produces a ready-for-dev tech spec
/quick-dev           # Implementation flow — mode detection, execution, review
/assess-complexity   # Evaluate complexity and route to right depth of planning
```

### Direct skill invocation

Skills can also be invoked directly via the Skill tool:

```
meld:meld-quick-spec
meld:meld-quick-dev
meld:meld-complexity-assessment
meld:meld-spec-engineering
meld:meld-adversarial-review
meld:meld-artifact-templates
```

## How It Works

MELD provides **methodology skills** that structure how you approach development:

1. **Assess complexity** — Count complexity signals, route to direct execution, quick-spec, or full planning
2. **Spec engineering** — Conversational flow that produces a ready-for-dev tech spec with Given/When/Then acceptance criteria
3. **Implementation** — Per-task subagent loop: TDD implementer → code simplifier → spec reviewer → adversarial code reviewer, with auto-fix cycles
4. **Final verification** — Aggregate gate function across the full codebase
5. **Completion** — Summary, worktree handling, retrospective capture

## Skills Reference

### Flow Skills

| Skill | Command | Description |
|-------|---------|-------------|
| `meld-quick-spec` | `/quick-spec` | 4-phase spec engineering: understand, investigate, generate, review |
| `meld-quick-dev` | `/quick-dev` | 5-phase implementation: setup & mode detect, context, per-task subagent loop (TDD + review), final verification, completion |
| `meld-complexity-assessment` | `/assess-complexity` | Complexity signals → routing to right depth |

### Methodology Skills

| Skill | Description |
|-------|-------------|
| `meld-tdd` | Test-driven development (Red-Green-Refactor), built into quick-dev |
| `meld-debugging` | Systematic debugging: 4-phase root cause methodology |
| `meld-verification` | Verification before completion: 5-step gate function |
| `meld-parallel-agents` | Parallel agent dispatch and integration |
| `meld-worktrees` | Git worktree creation with ticket-based branch naming |
| `meld-spec-engineering` | Given/When/Then format, task format, ready-for-dev standards |
| `meld-adversarial-review` | Information-asymmetric code review via subagents |
| `meld-code-simplifier` | Code simplification pass for clarity, consistency, and maintainability |
| `meld-retrospective` | Post-implementation retrospective: findings patterns, spec accuracy, estimation signals |

### Reference Skills

| Skill | Description |
|-------|-------------|
| `meld-artifact-templates` | 8 output templates (tech-spec, story, PRD, architecture, etc.) |

## License

MIT
