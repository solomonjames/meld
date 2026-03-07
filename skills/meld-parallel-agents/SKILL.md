---
name: meld-parallel-agents
description: Parallel agent dispatch — when to split work across subagents, how to craft prompts, and how to integrate results. Optional in quick-dev Phase 3.
---

# Parallel Agent Dispatch

## When to Use

Dispatch parallel agents when you have **2 or more independent tasks** that:
- Touch different files (no shared state)
- Have no ordering dependency
- Can be verified independently

## When NOT to Use

- Tasks share files or state
- Task B depends on Task A's output
- Changes must be ordered (migrations, schema changes)
- Single task that's just large (break it down first, then evaluate)

## The Pattern

### 1. Identify Independent Domains

Look at your task list. Group by file/module boundaries:

```
Task 1: Add user avatar upload (models/user.ts, routes/avatar.ts, tests/)
Task 2: Add email notification preferences (models/preferences.ts, routes/notifications.ts, tests/)
Task 3: Update dashboard stats query (services/stats.ts, tests/)
```

Tasks 1, 2, and 3 touch different files → safe to parallelize.

### 2. Craft Focused Prompts

Each agent needs a self-contained prompt. Include:

```
## Scope
{What to implement — be specific}

## Files to Modify
{Exact file paths}

## Existing Patterns
{Code conventions, import style, error handling approach}

## Acceptance Criteria
{What "done" looks like for this specific task}

## Constraints
- Do NOT modify files outside your scope
- Follow existing patterns exactly
- Write tests for all new behavior
- Run tests before reporting completion
```

**Critical:** Each prompt must be complete. Agents don't share context with each other or with your conversation.

### 3. Dispatch via Task Tool

```
Use the Task tool with subagent_type appropriate to the work:
- "Bash" for execution-heavy tasks
- "general-purpose" for tasks needing multiple tool types
```

Dispatch all independent agents simultaneously in a single message with multiple Task tool calls.

### 4. Review and Integrate

After all agents complete:

1. **Read each agent's output** — Don't trust "done" claims
2. **Check for conflicts** — Did agents modify overlapping files despite instructions?
3. **Run full test suite** — Agents may have tested in isolation; integration matters
4. **Verify with `meld:meld-verification`** — Fresh evidence for all claims

## Agent Prompt Template

```markdown
You are implementing a specific task as part of a larger feature.

## Task
{task_description}

## Files in Scope
- {file_1} — {what to change}
- {file_2} — {what to change}

## Project Patterns
- Test framework: {jest/pytest/etc.}
- Import style: {ES modules/CommonJS/etc.}
- Error handling: {pattern description}
- Naming: {camelCase/snake_case/etc.}

## Acceptance Criteria
- {AC_1}
- {AC_2}

## Constraints
- Only modify files listed above
- Match existing code style exactly
- Write tests following existing test patterns
- Run all tests and report results
```

## Common Mistakes

| Mistake | Why It Fails | Prevention |
|---------|-------------|------------|
| Overlapping file scope | Merge conflicts, lost work | Map file ownership before dispatch |
| Vague prompts | Agent guesses wrong, wastes time | Include exact files, patterns, ACs |
| Trusting agent claims | Agents hallucinate success | Verify everything yourself after |
| Too many agents | Context overhead, coordination cost | 2-4 agents max for most tasks |
| Shared test files | Agents write conflicting tests | Assign test file ownership too |

## Integration with Quick-Dev

This skill is optionally invoked in `meld:meld-quick-dev` Phase 3 (Execute):

- When the task list contains 2+ independent tasks touching different files
- Dispatch agents for independent tasks, execute dependent tasks sequentially
- After all agents complete, each task continues through per-task review steps (simplify, spec review, code review)
- Continue to Phase 4 (Final Verification) as normal
