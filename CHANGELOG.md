# Changelog

All notable changes to this project will be documented in this file.

## [0.2.0] - 2026-03-06

### Removed

- Legacy beads formula infrastructure: `formulas/` (7 TOML files), `skills/meld/` (46 step-based skill files), `legacy/install.sh`, `legacy/uninstall.sh`
- Beads workflow documentation: `docs/beads-workflows.md`, `docs/migration-from-beads.md`
- Beads integration sections from README.md, CLAUDE.md, and `skills/using-meld/SKILL.md`

### Fixed

- README.md: Corrected quick-dev phase count from 7 to 6
- README.md: Added missing `meld-code-simplifier` and `meld-retrospective` to Methodology Skills table

## [0.1.0] - 2026-02-21

### Added

- Initial extraction from web-identity-kit into standalone repository
- 7 MELD formulas: analysis, planning, solutioning, implementation, quick-spec, quick-dev, full
- 8 agent persona files (Analyst, Architect, PM, Dev, QA, SM, UX Designer, Quick Flow Solo Dev)
- 39 skill files across 7 workflow phases
- 8 output document templates (PRD, architecture, epics, stories, tech-spec, product-brief, UX design, sprint-status)
- `install.sh` — install formulas and skills into any beads-enabled project
- `uninstall.sh` — clean removal of all MELD files
- Formula reference documentation (`docs/beads-workflows.md`)
