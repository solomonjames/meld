---
name: meld-retrospective
description: Lightweight post-implementation retrospective — categorizes findings, measures spec accuracy, captures estimation signals, and persists learnings to project memory
---

# Retrospective Capture

Post-implementation retrospective that extracts learning signals and persists them to project memory. Transforms implementation experience into reusable knowledge.

**Style:** Be concise and data-driven — extract signals, skip narrative, persist facts.

## Inputs

- `baseline_commit` — Git commit before implementation started
- `findings` — List of adversarial review findings (F1, F2...) with severity, type, and resolution
- `execution_mode` — "A" (spec-driven) or "B" (direct instructions)
- `spec_tasks` — (Mode A only) Tasks from the tech spec
- `executed_tasks` — Tasks actually completed
- `feature_slug` — Kebab-case feature name for file naming
- `ticket_id` — (Optional) Beads ticket ID

---

## Section 1: Findings Pattern Analysis

### Categorize Findings

Classify each adversarial review finding into exactly one category:

| Category | Description |
|----------|-------------|
| `security` | Auth, injection, data exposure, secrets |
| `correctness` | Logic errors, wrong behavior, data integrity |
| `error-handling` | Missing catches, silent failures, bad error messages |
| `testing` | Missing tests, weak assertions, untested paths |
| `performance` | N+1 queries, unnecessary allocations, blocking calls |
| `style` | Naming, formatting, convention violations |
| `other` | Anything that doesn't fit above |

### Produce Summary Table

```
| Category | Count | Severity Breakdown | Fixed | Skipped |
```

### Check for Repeated Patterns

Read `_meld-output/learnings.md` if it exists. Cross-reference current findings against "Common Finding Patterns" section. Flag any pattern that has occurred before — these are systemic issues worth calling out.

---

## Section 2: Spec Accuracy (Mode A Only)

If `execution_mode` is "B", output "N/A — Mode B execution" and skip this section.

### Compare Planned vs Actual

For each spec task, categorize:
- **Completed as planned** — Implemented as specified
- **Modified** — Implemented but approach changed
- **Added** — Task not in spec but needed during implementation
- **Removed** — Spec task that turned out unnecessary

### Calculate Accuracy

```
accuracy_ratio = completed_as_planned / total_spec_tasks
```

### Note Drift Causes

For each modified/added/removed task, note WHY the deviation occurred (1 line each). These causes feed future spec quality improvements.

---

## Section 3: Estimation Signal

### Task Variance

```
planned_tasks = count(spec_tasks or initial_plan_tasks)
actual_tasks = count(executed_tasks)
variance = (actual_tasks - planned_tasks) / planned_tasks
```

### Diff Stats

```bash
git diff --stat {baseline_commit}
```

Capture: files changed, insertions, deletions.

### Flag Variance

If task variance exceeds 30%, note it as a significant estimation miss. Record whether the miss was over-estimation (planned more than needed) or under-estimation (discovered work during implementation).

---

## Section 4: Persist to Project Memory

### 4a: Per-Feature Retrospective (Archival)

Write to `_meld-output/retrospectives/{YYYY-MM-DD}-{feature_slug}.md`:

```markdown
# Retrospective: {feature_slug}
Date: {YYYY-MM-DD}
Mode: {A|B}
Ticket: {ticket_id or "N/A"}

## Findings Summary
{summary table from Section 1}

## Spec Accuracy
{accuracy table from Section 2, or "N/A — Mode B"}

## Estimation
- Planned tasks: {N}
- Actual tasks: {N}
- Variance: {%}
- Diff: {files changed}, +{insertions}, -{deletions}

## Repeated Patterns
{any patterns flagged from learnings.md cross-reference}
```

### 4b: Accumulated Learnings (Read-Merge-Write)

Read `_meld-output/learnings.md` if it exists. Merge new data, then write back.

Structure:

```markdown
# MELD Learnings

Accumulated patterns from past implementations. Auto-updated by meld-retrospective.

## Common Finding Patterns

| Pattern | Category | Occurrences | Last Seen |
|---------|----------|-------------|-----------|

## Spec Accuracy Trends (Mode A Only)

| Date | Feature | Accuracy | Planned | Actual | Primary Drift Cause |
|------|---------|----------|---------|--------|---------------------|

## Estimation Signals

| Date | Feature | Planned Tasks | Actual Tasks | Variance | Files Changed |
|------|---------|---------------|--------------|----------|---------------|
```

Update rules:
- **Common Finding Patterns:** Increment occurrence count for existing patterns. Add new rows for new patterns. Update "Last Seen" date.
- **Spec Accuracy Trends:** Append row. Only for Mode A executions.
- **Estimation Signals:** Append row.

### 4c: CLAUDE.md Auto-Reference (One-Time)

On first retrospective run:
1. Check if `CLAUDE.md` exists in the project root
2. Check if it already contains a reference to `_meld-output/learnings.md`
3. If CLAUDE.md exists and has no reference, append:

```markdown

## MELD Learnings
See `_meld-output/learnings.md` for accumulated patterns from past implementations (common review findings, spec accuracy trends, estimation signals).
```

Subsequent runs detect the existing reference and skip this step.

---

## Beads Integration

If `{beads_active}` and `{ticket_id}` is set:

1. Append retrospective summary to ticket notes:
   ```
   bd update {ticket_id} --notes "... existing notes ...\n\n**Retrospective:**\n{findings_count} findings ({fixed} fixed, {skipped} skipped)\nSpec accuracy: {ratio or N/A}\nTask variance: {variance}%"
   ```

2. Add summary comment:
   ```
   bd comment {ticket_id} "Retrospective captured — {findings_count} findings analyzed, {repeated_count} repeated patterns flagged"
   ```

3. Merge metadata (read-merge-write):
   ```json
   {
     "retro_finding_types": {"security": 0, "correctness": 1, "error-handling": 0, "testing": 2, "performance": 0, "style": 1, "other": 0},
     "spec_accuracy": 0.85
   }
   ```
