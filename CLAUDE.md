# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MELD is a Claude Code plugin that implements the MELD (Methodology for Engineering Lifecycle & Development) methodology. It provides structured product development with complexity routing, spec engineering, adversarial review, and execution skills (TDD, debugging, verification, parallel agents, worktrees).

This is **not** a code library — it is a plugin/workflow system with no build step, no tests, and no package manager.

## Repository Structure

### Plugin Infrastructure
- `.claude-plugin/plugin.json` — Plugin manifest (name, version, metadata)
- `.claude-plugin/marketplace.json` — Self-hosted marketplace config
- `hooks/` — SessionStart hook that injects `using-meld` meta-skill into context
- `commands/` — Slash commands (`/quick-spec`, `/quick-dev`, `/assess-complexity`)

### Skills (Plugin Skills)
- `skills/using-meld/` — Meta-skill: routing table for when to use MELD vs other skills
- `skills/meld-quick-spec/` — Merged 4-phase spec engineering flow
- `skills/meld-quick-dev/` — Merged 6-phase implementation flow + adversarial reviewer prompt
- `skills/meld-complexity-assessment/` — Complexity signal evaluation and routing
- `skills/meld-tdd/` — Test-driven development methodology (Red-Green-Refactor), embedded in quick-dev
- `skills/meld-spec-engineering/` — Given/When/Then format, ready-for-dev standards
- `skills/meld-adversarial-review/` — Information-asymmetric code review + reviewer prompt
- `skills/meld-debugging/` — Systematic debugging: 4-phase root cause methodology + 3 supporting docs
- `skills/meld-code-simplifier/` — Code simplification pass for clarity, consistency, and maintainability
- `skills/meld-verification/` — Verification before completion: 5-step gate function
- `skills/meld-parallel-agents/` — Parallel agent dispatch and integration
- `skills/meld-worktrees/` — Git worktree creation with ticket-based branch naming
- `skills/meld-retrospective/` — Post-implementation retrospective: findings patterns, spec accuracy, estimation signals, project memory persistence
- `skills/meld-artifact-templates/` — 8 output templates + index

### Beads Integration (Legacy)
- `formulas/` — 7 TOML workflow definitions for beads users
- `skills/meld/` — Original step-by-step skill files for beads formulas
- `legacy/install.sh` / `legacy/uninstall.sh` — Copy formulas/skills into beads-enabled projects
- `docs/beads-workflows.md` — Beads workflow documentation

## How It Works

### As a Plugin
The SessionStart hook injects the `using-meld` meta-skill into every session. This provides a routing table so the agent knows when to invoke MELD skills. Users trigger flows via slash commands (`/quick-spec`) or direct skill invocation (`meld:meld-quick-spec`).

### As Beads Formulas
Formulas (TOML) define step sequences with dependencies. Each step references an agent persona and a skill file. The `legacy/install.sh` script copies formulas and skills into a beads-enabled project.

## Skill Authoring

Each plugin skill is a single `SKILL.md` file with:
- **YAML frontmatter:** `name` and `description` (used for skill discovery and auto-triggering)
- **Progress Tracking:** Tasks created via TaskCreate/TaskUpdate during execution
- **Phases:** Logical groupings of work within the skill
- **Sub-skill references:** `meld:skill-name` for other MELD skills

### Naming Convention
All skills are prefixed `meld-` to avoid collisions. Invoked as `meld:meld-quick-spec` (plugin:skill format).

### Style Instructions
Flow skills include a one-line `**Style:**` directive that sets behavioral expectations (e.g., "Be direct and efficient"). No external persona files are loaded.

## Key Patterns

- **Information asymmetry in reviews:** Adversarial review uses a subagent that sees ONLY the diff, not the spec or conversation.
- **Human gates:** Flow skills pause for human approval at key points (spec review, finding resolution).
- **WIP lifecycle:** Tech specs track progress via `stepsCompleted` in YAML frontmatter and transition through statuses: `in-progress` → `review` → `ready-for-dev` → `completed`.
- **Worktree-ticket binding:** When beads is active, worktrees use `feature/{ticket_id}` branch naming to bind isolation to the tracking ticket.
- **Supporting docs:** Some skills (e.g., `meld-debugging`) have supporting markdown files alongside SKILL.md. These are referenced by relative path within the skill, not as separate skills.

## Editing Guidelines

- **Plugin skills** (`skills/meld-*/SKILL.md`): Single file per skill with YAML frontmatter. Keep skills focused — one concept per file, practical over theoretical.
- **Commands** (`commands/*.md`): Lightweight wrappers with YAML frontmatter (`description`, `disable-model-invocation: true`) that invoke a skill.
- **Hooks** (`hooks/`): SessionStart injects meta-skill content. Uses JSON output compatible with both Cursor and Claude Code.
- **Templates** (`skills/meld-artifact-templates/*.md`): Use `{{variable}}` placeholders and YAML frontmatter for metadata.
- **Beads formulas** (`formulas/*.toml`): TOML with `[[step]]` arrays. Each step has `id`, `title`, `description`, `depends_on`.
