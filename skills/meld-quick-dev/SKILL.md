---
name: meld-quick-dev
description: Use when ready to implement a feature — 5-phase implementation flow with mode detection, per-task subagent loop (TDD + review), final verification, and completion
---

# Quick Dev

Implementation flow that handles both spec-driven (Mode A) and direct instruction (Mode B) development. Uses a **controller pattern** where the main agent orchestrates fresh subagents for each task — implementing, simplifying, and reviewing per-task rather than at the end.

**Style:** Be direct and efficient — skip unnecessary ceremony, execute decisively, implement exactly what's specified without gold-plating.

## Why This Flow

Each phase exists for a specific reason:

1. **Setup & Mode Detection** — Establishes a baseline so you can diff later for review. Determines whether you have a spec (structured) or need to gather context (ad-hoc).
2. **Context Gathering** (Mode B only) — Without a spec, you need to understand the codebase before writing code. Skipped in Mode A because the spec IS the context.
3. **Execute (Per-Task Loop)** — Each task gets a fresh subagent for implementation, keeping the controller's context clean. Per-task review catches issues immediately rather than after all tasks are done — cheaper to fix, less context to unwind. Information asymmetry is preserved: the code quality reviewer sees ONLY the diff.
4. **Final Verification** — Lightweight aggregate check that the full codebase is consistent after all tasks complete. Individual tasks passed their own reviews, but integration issues only surface when you look at the whole.
5. **Completion** — Summary, worktree handling, retrospective capture, and beads sync. Everything needed to close out the work.

## Controller Pattern

The main agent is the **controller**. It:
- Orchestrates the per-task loop (prepare → implement → simplify → review → complete)
- Dispatches subagents for implementation, simplification, and review
- Independently validates all subagent claims (runs tests itself, reads output)
- Never writes implementation code directly
- Answers subagent questions and re-dispatches when needed
- Tracks findings and results for the aggregate report

**Trust rule:** Never trust subagent completion claims. Always run tests independently after each subagent returns.

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

| # | subject | blockedBy |
|---|---------|-----------|
| 1 | Setup and detect mode | — |
| 2 | Gather context and confirm plan | 1 |
| 3 | Execute per-task loop | 1 or 2 |
| 4 | Final verification | 3 |
| 5 | Completion and retrospective | 4 |

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

## Phase 3: Execute (Per-Task Loop)

The controller orchestrates a subagent-driven loop for each task. The controller never writes implementation code directly.

### TDD Methodology
This phase follows test-driven development. The implementer subagent receives `meld:meld-tdd` methodology via the implementer prompt — the core rule is **no production code without a failing test first.**

**Exception:** If the project has no test framework or the task is configuration-only, note the exception in the implementer prompt context.

### Per-Task Loop

For each task in the plan/spec, execute steps 1-6 sequentially:

#### Step 1: PREPARE

```bash
git rev-parse HEAD
```
Store as `task_baseline`. This anchors the diff for this specific task.

If `{beads_active}` and tasks are sub-tickets: `bd update {sub_id} --status in_progress`

Gather task context for the implementer:
- Task description and acceptance criteria
- Files in scope (which files to read/modify)
- Project patterns (from Phase 1/2 context gathering)
- Any relevant context from previous tasks in this loop

#### Step 2: IMPLEMENT (Subagent)

Read `implementer-prompt.md` from this skill directory. Construct the subagent prompt:

```
Agent tool call:
  prompt: "<contents of implementer-prompt.md>

  ## Task Description
  {task_description}

  ## Acceptance Criteria
  {acceptance_criteria}

  ## Files in Scope
  {files_in_scope}

  ## Project Patterns
  {project_patterns}"
  subagent_type: "general-purpose"
  description: "TDD implement task N"
```

The subagent does RED/GREEN/REFACTOR internally and reports results.

**After subagent returns:**
1. **Validate independently** — Run the full test suite yourself. Never trust subagent claims.
2. **If tests pass** → Continue to Step 3.
3. **If tests fail** → Re-dispatch the implementer with the failure output appended to the prompt. Include: "The following tests are failing after your implementation. Fix them."
4. **Max 3 attempts.** After 3 consecutive failures on the same task → halt (see Halt Conditions).

**If subagent asks questions:** Answer them from your controller context, then re-dispatch with the answers included.

#### Step 3: SIMPLIFY (Subagent)

**Skip if** the task diff is fewer than 20 lines (`git diff --stat {task_baseline}` — check total insertions + deletions).

Read `meld:meld-code-simplifier` (the SKILL.md content serves as the subagent prompt). Dispatch:

```
Agent tool call:
  prompt: "<contents of meld-code-simplifier/SKILL.md>

  ## Baseline
  {task_baseline}

  ## Files Changed
  {list of files from task diff}"
  subagent_type: "code-simplifier"
  description: "Simplify task N code"
```

**After subagent returns:**
1. Run the full test suite.
2. **If tests pass** → Accept simplifications. Continue to Step 4.
3. **If tests fail** → Revert the simplifier's changes (`git checkout {task_baseline} -- {affected_files}`), note "Simplifier changes reverted — broke tests", continue to Step 4.

#### Step 4: SPEC REVIEW (Subagent)

Read `spec-reviewer-prompt.md` from this skill directory. Dispatch:

```
Agent tool call:
  prompt: "<contents of spec-reviewer-prompt.md>

  ## Task Description
  {task_description}

  ## Acceptance Criteria
  {acceptance_criteria}

  ## Implementer Report
  {implementer_report_from_step_2}"
  subagent_type: "general-purpose"
  description: "Spec review task N"
```

The spec reviewer reads actual code files and verifies compliance.

**After subagent returns:**
- **If PASS** → Continue to Step 5.
- **If FAIL** → Re-dispatch the implementer (Step 2) with the spec review findings: "The spec reviewer found these issues. Fix them: {findings}". Then re-run spec review.
- **Max 2 spec-review-fix cycles.** After 2 cycles with remaining issues → log unresolved findings and continue.

#### Step 5: CODE QUALITY REVIEW (Subagent)

**Skip if** the task diff is fewer than 10 lines.

Capture the task diff:
```bash
git diff {task_baseline}
```
Include new files.

Read `adversarial-reviewer-prompt.md` from this skill directory. Dispatch with information asymmetry preserved — the reviewer sees ONLY the diff:

```
Agent tool call:
  prompt: "<contents of adversarial-reviewer-prompt.md>

  <diff>
  {task_diff}
  </diff>"
  subagent_type: "general-purpose"
  description: "Code review task N"
```

**After subagent returns:**

Process findings using the framework from `meld:meld-adversarial-review`:
1. Rate each finding for severity (Critical/High/Medium/Low) and validity (Real/Noise/Undecided).
2. **Auto-fix "Real" Critical and High findings** — Re-dispatch implementer with: "Fix these code review findings: {real_critical_and_high_findings}". Run tests after.
3. **Log Medium and Low findings** — Record for the aggregate report but do not auto-fix.
4. **Max 2 review-fix cycles.** After 2 cycles → log remaining findings and continue.

#### Step 6: COMPLETE TASK

1. Run the full test suite one final time.
2. If `{beads_active}` and tasks are sub-tickets: `bd close {sub_id}`
3. Record task results for the aggregate report:
   - Files changed
   - Tests added
   - Spec review result (PASS/FAIL + findings)
   - Code quality findings (count by severity/validity)
   - Simplifications applied (count)
4. Continue immediately to the next task.

### Parallel Execution (Optional)
When the task list contains 2+ **truly independent** tasks (different files, no shared state), the controller MAY dispatch multiple implementer subagents in parallel via `meld:meld-parallel-agents`. Each parallel task still goes through the full Step 1-6 loop — parallelism only applies to Step 2 (IMPLEMENT). Steps 3-6 run sequentially per task after parallel implementation completes.

### Halt Conditions
ONLY halt for:
- 3 consecutive implementer failures on the same task
- Tests failing after max re-dispatch attempts
- Blocking dependency (missing package, API, etc.)
- Ambiguity requiring user decision

When halted, follow `meld:meld-debugging` before attempting more fixes. Do NOT guess.

Do NOT halt for minor warnings, optional improvements, or deprecation notices.

### Phase 3 Beads Sync
If `{beads_active}` (after ALL tasks complete, not per-task):
1. Merge metadata: `{"meld_step": "execute"}`
2. `bd comment {ticket_id} "MELD quick-dev Phase 3 complete — {completed_count}/{total_count} tasks implemented, {total_findings} review findings ({fixed_count} auto-fixed)"`

---

## Phase 4: Final Verification

After all tasks complete, run a lightweight aggregate check. Individual tasks passed their own reviews, so this is about integration consistency.

### Run Verification Gate
**REQUIRED:** Invoke `meld:meld-verification` gate function using `baseline_commit` (from Phase 1, not per-task baselines). Run the full test suite, linter, and build fresh. Read complete output.

### Abbreviated Self-Audit
Verify:
1. All tasks marked complete
2. No tasks skipped without documented reason
3. Full test suite passes (fresh run, not cached)
4. Build succeeds
5. Linter clean (or only pre-existing warnings)

### Handle Failures
If any check fails: attempt to fix, re-run verification. If not fixable after one attempt, document and flag for user.

### Phase 4 Beads Sync
If `{beads_active}`:
1. Merge metadata: `{"meld_step": "verify"}`
2. `bd comment {ticket_id} "MELD quick-dev Phase 4 complete — verification gate passed, all tests green"`

---

## Phase 5: Completion

### Aggregate Summary

Present the full implementation summary:

**Tasks:**
- Completed: N/N
- Per-task results (1-line each): files changed, tests added, review findings

**Findings (aggregate):**

| Severity | Total | Real | Fixed | Logged |
|----------|-------|------|-------|--------|
| Critical | N | N | N | N |
| High | N | N | N | N |
| Medium | N | N | N | N |
| Low | N | N | N | N |

**Tests:** X new, Y total passing
**Files:** N modified, M created

### Worktree Completion
If in a worktree, present completion options via `meld:meld-worktrees`:
- **[P] Create Pull Request** — push branch and create PR
- **[M] Merge to main** — merge branch and clean up worktree
- **[K] Keep for later** — leave worktree in place
- **[D] Discard** — remove worktree and branch

**Wait for user decision.**

### Retrospective Capture

**Skip if** diff is trivial (<20 lines, <3 files) AND no review findings were generated. Note: "Retrospective skipped — trivial change."

Otherwise, **REQUIRED SUB-SKILL:** Use `meld:meld-retrospective`.
Pass: baseline_commit, findings list (aggregate from all tasks), execution mode, spec tasks (Mode A), executed tasks, feature slug/ticket ID.

### Phase 5 Beads Sync
If `{beads_active}`:
1. Merge metadata: `{"meld_phase": "done", "meld_step": "complete", "finding_count": {total}, "findings_fixed": {fixed_count}, "findings_logged": {logged_count}, "spec_status": "completed"}`
2. `bd comment {ticket_id} "MELD quick-dev Phase 5 complete — {finding_count} total findings ({fixed_count} fixed, {logged_count} logged). All tests passing."`
3. Ask user: "Implementation is complete. Close this ticket? (`bd close {ticket_id} --reason 'Implementation complete — {summary}'`)"

---

Suggest: commit, run additional tests, or start a new workflow.
