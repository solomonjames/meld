---
name: meld-quick-dev
description: Use when ready to implement a feature — 6-phase implementation flow with mode detection, TDD execution, verification, adversarial review, finding resolution, and retrospective capture
---

# Quick Dev

Implementation flow that handles both spec-driven (Mode A) and direct instruction (Mode B) development. Includes TDD execution, verification, adversarial review, finding resolution, and retrospective capture.

**Style:** Be direct and efficient — skip unnecessary ceremony, execute decisively, implement exactly what's specified without gold-plating.

## Why This Flow

Each phase exists for a specific reason:

1. **Setup & Mode Detection** — Establishes a baseline so you can diff later for review. Determines whether you have a spec (structured) or need to gather context (ad-hoc).
2. **Context Gathering** (Mode B only) — Without a spec, you need to understand the codebase before writing code. Skipped in Mode A because the spec IS the context.
3. **Execute** — TDD ensures every behavioral change is verified as you go, not after the fact. Continuous execution without pausing keeps momentum.
4. **Verify & Self-Check** — Fresh evidence for every completion claim. Models are prone to claiming "done" based on stale or assumed results. This phase forces honesty.
5. **Adversarial Review & Resolution** — A subagent reviews the diff with NO context about intent. This information asymmetry catches issues the implementing agent is blind to — you can't review your own work objectively.
6. **Retrospective Capture** — Categorizes findings, measures spec accuracy, captures estimation signals, and persists learnings to project memory so future specs avoid past mistakes.

## Beads Integration (Optional)

If `{ticket_id}` is set and beads is active, the ticket is the primary data source. When the ticket contains a ready-for-dev spec (from quick-spec), all spec content is loaded from ticket fields — no local files needed.

### Detection
1. Check if `{ticket_id}` is set (non-empty). If not, set `{beads_active}` to false.
2. Verify beads is installed: `which bd`. If not found, warn user ("beads not installed, proceeding without ticket tracking"), set `{beads_active}` to false.
3. Ensure working directory: Before running `bd` commands, verify you are within the project directory containing `.beads/`. If session context provides a beads project root, `cd` to it.
4. Load ticket: `bd show {ticket_id} --json`. If the ticket doesn't exist, warn and set `{beads_active}` to false.
5. Store the ticket's current `metadata` JSON for read-merge-write operations.

### Ticket-Driven Mode Detection
When `{beads_active}`, check ticket metadata:
- If `metadata.spec_status == "ready-for-dev"` → load spec from ticket fields: `design` (overview), `notes` (technical context), `acceptance_criteria` (ACs), sub-tickets (implementation tasks via `bd list --parent {ticket_id} --json`). Auto-select **Mode A**.
- If `metadata.meld_phase == "spec"` and `spec_status != "ready-for-dev"` → warn user that spec is incomplete, suggest running `/quick-spec {ticket_id}` first.
- Otherwise → fall through to normal Mode B detection.

### Beads Sync Pattern
Every metadata update follows read-merge-write:
1. Read current metadata: `bd show {ticket_id} --json` → extract `.metadata`
2. Merge new fields into the existing object (never discard existing keys)
3. Write full merged JSON: `bd update {ticket_id} --metadata '{...}'`

Sync at each phase boundary:
- Update metadata with current `meld_step`
- Add one `bd comment` summarizing the phase outcome
- Update sub-ticket statuses as tasks complete

## Progress Tracking

At skill start, create these tasks via TaskCreate. Mark `in_progress` when starting, `completed` when done.

| # | subject | activeForm | blockedBy |
|---|---------|------------|-----------|
| 1 | Setup and detect mode | Setting up and detecting mode | — |
| 2 | Gather context and confirm plan | Gathering context and confirming plan | 1 |
| 3 | Execute implementation | Executing implementation tasks | 1 or 2 |
| 4 | Verify and self-check | Running verification and self-check | 3 |
| 5 | Adversarial review and resolve | Running adversarial review and resolving findings | 4 |
| 6 | Write completion summary | Writing completion summary | 5 |
| 7 | Capture retrospective | Capturing retrospective and persisting learnings | 6 |

**Mode note:** Task 3 depends on task 1 in Mode A (skip context gathering) or task 2 in Mode B. If Mode A, mark task 2 `completed` immediately.

---

## Phase 1: Setup & Mode Detection

### Capture Baseline Commit
```bash
git rev-parse HEAD
```
Store as `baseline_commit`. If not in a git repo, set to "NO_GIT".

### Load Project Context
Read `project-context.md` if it exists. Note project conventions, patterns, constraints.

### Determine Execution Mode

**Mode A — Tech Spec (structured):**
- Triggers when user provides a tech-spec path, or when `{beads_active}` and ticket has `metadata.spec_status == "ready-for-dev"`
- **From ticket:** Load spec from `design` (overview), `notes` (technical context), `acceptance_criteria` (ACs). List sub-tickets as implementation tasks: `bd list --parent {ticket_id} --json`.
- **From file:** Load the tech-spec, verify it has `status: 'ready-for-dev'`, extract tasks and acceptance criteria.
- **Skip to Phase 3: Execute**

**Mode B — Direct Instructions (ad-hoc):**
- Triggers when no tech-spec is provided (and ticket has no ready spec)
- Parse user's feature description
- Proceed to escalation evaluation

### Escalation Evaluation (Mode B Only)
**REQUIRED SUB-SKILL:** Use `meld:meld-complexity-assessment` to evaluate complexity and route appropriately.

If user chooses to escalate:
- **[P] Plan first** → Suggest `meld:meld-quick-spec`, then return with tech spec
- **[W] Full MELD** → Guide to appropriate phase formula

If user chooses **[E] Execute directly** → Continue to Phase 2.

### Create Worktree (Optional)

**If `{beads_active}`:** Auto-create a worktree via `meld:meld-worktrees`. Branch name: `feature/{ticket_id}` (or `feature/{ticket_id}-{spec_slug}` if metadata has `spec_slug`). Re-capture `baseline_commit` from the worktree HEAD.

**If not beads-active:** Offer worktree creation as an option. If accepted, follow `meld:meld-worktrees` with kebab-case branch naming. Re-capture `baseline_commit` from worktree HEAD.

If worktree is declined or not applicable, note it and continue.

### Phase 1 Beads Sync
If `{beads_active}`:
1. `bd update {ticket_id} --status in_progress`
2. Merge metadata: `{"meld_phase": "dev", "meld_step": "mode-detect", "baseline_commit": "{baseline_commit}", "execution_mode": "{mode_a_or_mode_b}"}`
3. `bd comment {ticket_id} "MELD quick-dev Phase 1 complete — {Mode A/B} execution, baseline {baseline_commit}"`

---

## Phase 2: Context Gathering (Mode B Only)

Mode A skips this phase entirely — the tech spec IS the context.

### Identify Files to Modify
- Glob/grep for files related to the feature area
- Read key files to understand current implementation
- List all files that need changes and new files to create

### Find Relevant Patterns
- Code style and naming conventions
- Import/export patterns
- Error handling approach
- Test patterns (framework, helpers, factories)
- Database patterns (migrations, models, relationships)

### Note Dependencies
- External libraries needed (check if already installed)
- Internal modules to integrate with
- Config files that need updates

### Create Implementation Plan
- **Tasks:** Ordered list of discrete changes
- **Inferred ACs:** What "done" looks like for each task
- **Order of operations:** Dependency-correct sequence

### Present Plan for Confirmation
Show files, patterns, plan, and acceptance criteria. Ask: "Ready to execute? Adjust anything?"

**Wait for user confirmation before proceeding.** This is the one human gate before continuous execution — it's the last chance to correct course cheaply.

### Phase 2 Beads Sync
If `{beads_active}`:
1. `bd update {ticket_id} --notes "**Implementation Plan:**\n\n{task_list}\n\n**Files to modify:** {files_list}\n\n**Dependencies:** {dependencies}"`
2. `bd update {ticket_id} --acceptance-criteria "{inferred_acceptance_criteria}"`
3. Merge metadata: `{"meld_step": "context-gather"}`
4. `bd comment {ticket_id} "MELD quick-dev Phase 2 complete — {task_count} tasks planned, {file_count} files identified"`

---

## Phase 3: Execute

### TDD Methodology
This phase follows test-driven development. Read `meld:meld-tdd` for the full methodology — the core rule is **no production code without a failing test first.**

**Exception:** If the project has no test framework or the task is configuration-only, skip TDD steps and implement directly. Note the exception.

### Execution Loop
For each task in the plan/spec:

1. **Start task** — If `{beads_active}` and tasks are sub-tickets: `bd update {sub_id} --status in_progress`
2. **Load context** — Read relevant files, review patterns, identify test file locations and testing conventions
3. **RED** — Write one minimal failing test capturing expected behavior. Run it. Confirm it fails because the feature is missing (not errors or typos). If it passes immediately, you're testing existing behavior — fix the test.
4. **GREEN** — Write the simplest code to make the failing test pass. Run all tests. Confirm everything is green.
5. **REFACTOR** — Clean up while keeping tests green. Remove duplication, improve names. Do not add behavior.
6. **Mark complete** — If `{beads_active}` and tasks are sub-tickets: `bd close {sub_id}`
7. **Continue immediately** — Next task without pausing

### Critical Rules
- **Continuous execution** — Do NOT stop between tasks for approval. The human gate was Phase 2.
- **Follow patterns** — Match existing codebase conventions exactly
- **No gold-plating** — Implement exactly what's specified, nothing more

### Parallel Execution (Optional)
When the task list contains 2+ independent tasks (different files, no shared state), MAY dispatch via `meld:meld-parallel-agents`.

### Halt Conditions
ONLY halt for:
- 3 consecutive failures on the same task
- Tests failing with no obvious fix
- Blocking dependency (missing package, API, etc.)
- Ambiguity requiring user decision

When halted, follow `meld:meld-debugging` before attempting more fixes. Do NOT guess.

Do NOT halt for minor warnings, optional improvements, or deprecation notices.

### Phase 3 Beads Sync
If `{beads_active}` (after ALL tasks complete, not per-task):
1. Merge metadata: `{"meld_step": "execute"}`
2. `bd comment {ticket_id} "MELD quick-dev Phase 3 complete — {completed_count}/{total_count} tasks implemented"`

---

## Phase 4: Verify & Self-Check

This phase combines self-audit with fresh verification evidence. The point: models are prone to claiming "done" based on memory or assumptions. Every claim needs proof from THIS session, AFTER the latest change.

### Skip Conditions
If the diff is trivial (fewer than 20 lines changed AND fewer than 3 files modified via `git diff --stat {baseline_commit}`), skip the full audit and just run the verification gate.

### Self-Audit Checklist
Verify each item honestly:

**Tasks:**
1. All tasks marked complete
2. No tasks skipped without documented reason
3. Implementation matches task descriptions (no drift)

**Tests:**
4. All existing tests still pass
5. New tests written for new behavior
6. Test coverage covers happy path and error/edge cases

**Acceptance Criteria:**
7. Each AC is demonstrably satisfied
8. Edge cases from ACs are handled

**Patterns:**
9. Code follows existing codebase conventions
10. No code smells introduced (duplication, god objects, deep nesting)

### Verification Gate

**REQUIRED:** Invoke `meld:meld-verification` gate function. Run the full test suite, linter, and build fresh. Read complete output. Match evidence to every claim from the audit above.

Any claim without fresh evidence? Go back and gather it.

### Handle Failures
Fix immediately if possible. Re-run affected tests. If not fixable, document and flag for user.

### Present Summary
- Tasks completed: N/N
- Tests: X new, Y total passing
- Files modified/created
- Checklist: all items passing (or flagged)

### Phase 4 Beads Sync
If `{beads_active}`:
1. Merge metadata: `{"meld_step": "verify"}`
2. `bd comment {ticket_id} "MELD quick-dev Phase 4 complete — {pass_count}/10 checks passing, {new_test_count} new tests"`
3. Update spec artifact: mark tasks `[x]`, add implementation notes if approach deviated.

---

## Phase 5: Adversarial Review & Resolution

### Construct Diff
```bash
git diff {baseline_commit}
```
Include new files. Use NO_GIT fallback if needed.

### Invoke Review
**REQUIRED SUB-SKILL:** Use `meld:meld-adversarial-review` for the full procedure.

Use a subagent (Task tool) for true information asymmetry. The reviewer sees ONLY the diff and the reviewer prompt — NO spec, NO conversation history. This is the whole point: you can't objectively review code you just wrote, so a fresh agent with no context catches what you're blind to.

### Process and Present Findings
Number findings (F1, F2...) with severity and validity ratings. Order by severity. Flag zero findings as suspicious.

### Human Gate — Resolve Findings
Present options:
- **[W] Walk through** — Iterate each finding: Fix / Skip / Discuss
- **[F] Fix automatically** — Fix all "Real" findings, skip Noise/Undecided
- **[S] Skip** — Acknowledge and proceed

### Apply Resolution
- **Walk through:** Per-finding decision with test re-runs after fixes
- **Fix automatically:** Filter to "Real" only, apply, re-run all tests, report
- **Skip:** Note findings for future reference

### Final Verification
**REQUIRED:** Run `meld:meld-verification` gate function one last time. This catches regressions introduced during finding resolution.

### Worktree Completion
If in a worktree, present completion options via `meld:meld-worktrees`:
- **[P] Create Pull Request** — push branch and create PR
- **[M] Merge to main** — merge branch and clean up worktree
- **[K] Keep for later** — leave worktree in place
- **[D] Discard** — remove worktree and branch

**Wait for user decision.**

### Phase 5 Beads Sync
If `{beads_active}`:
1. Merge metadata: `{"meld_phase": "done", "meld_step": "resolve-findings", "finding_count": {total}, "findings_fixed": {fixed_count}, "findings_skipped": {skipped_count}, "spec_status": "completed"}`
2. `bd comment {ticket_id} "MELD quick-dev Phase 5 complete — {finding_count} findings ({fixed_count} fixed, {skipped_count} skipped). All tests passing."`
3. Ask user: "Implementation is complete. Close this ticket? (`bd close {ticket_id} --reason 'Implementation complete — {summary}'`)"

### Completion Summary
- Files modified/created
- Tests: all passing (count)
- Review: N findings, M fixed, K skipped
- Status: Complete

---

## Phase 6: Retrospective Capture

### Skip Conditions
If diff is trivial (<20 lines, <3 files) AND adversarial review produced zero findings, skip. Note: "Retrospective skipped — trivial change."

### Invoke Retrospective
**REQUIRED SUB-SKILL:** Use `meld:meld-retrospective`.
Pass: baseline_commit, findings list, execution mode, spec tasks (Mode A), executed tasks, feature slug/ticket ID.

### Phase 6 Beads Sync
If `{beads_active}`:
1. Merge metadata: `{"meld_step": "retrospective"}`
2. `bd comment {ticket_id} "MELD quick-dev Phase 6 complete — retrospective captured"`

---

Suggest: commit, run additional tests, or start a new workflow.
