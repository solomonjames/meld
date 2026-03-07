---
name: meld-verification
description: Verification before completion — 5-step gate function that requires fresh evidence for every completion claim. Referenced by quick-dev Phases 4 and 5.
---

# Verification Before Completion

## The Iron Law

```
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
```

Every claim that something "works" or is "done" must be backed by evidence you gathered THIS session, AFTER the latest change. Not from memory. Not from a previous run. Not from what you "expect" to happen.

- Passing tests from a previous run? Stale. Run again.
- "I already checked that"? Check again.
- Output you didn't read line-by-line? Not verified.

## The Gate Function

Every completion claim — task, phase, or full implementation — passes through these 5 steps:

### Step 1: Identify What to Verify

List every claim you're about to make:
- "All tests pass"
- "The feature works as specified"
- "No regressions introduced"
- "Linter/build clean"

Each claim needs its own evidence.

### Step 2: Run the Verification

Execute the actual commands:

```bash
# Tests
npm test          # or pytest, cargo test, go test, etc.

# Build
npm run build     # or equivalent

# Lint
npm run lint      # or equivalent
```

Do NOT skip any step. Do NOT assume a step will pass because the last one did.

### Step 3: Read the Complete Output

**MANDATORY. Never skip.**

Read the ENTIRE output of each command. Not just the summary line. Not just "X passed."

Look for:
- Failing tests (even if summary says "passed" — check for skipped/pending)
- Warnings that indicate real problems
- Error messages buried in verbose output
- Deprecation warnings that affect functionality

### Step 4: Match Evidence to Claims

For each claim from Step 1, confirm you have evidence from Steps 2-3:

| Claim | Evidence | Status |
|-------|----------|--------|
| All tests pass | `42 passed, 0 failed` from test run at {time} | Verified |
| Build clean | `Build succeeded` from build at {time} | Verified |
| Linter clean | `0 errors, 0 warnings` from lint at {time} | Verified |
| Feature works | Tests cover all ACs, all pass | Verified |

Any claim without matching evidence? Go back to Step 2.

### Step 5: Make the Claim

Only now can you say "done" or "complete." Include the evidence summary.

## Common Failures

| Failure | What Happens | Prevention |
|---------|-------------|------------|
| Stale evidence | Cite results from before latest change | Always re-run after every change |
| Partial read | Miss a failing test in verbose output | Read complete output, not just summary |
| Assumed pass | "Tests were passing earlier" | Run. Read. Verify. Every time. |
| Wrong scope | Verify unit tests but skip integration | Verify everything the claim covers |
| Skipped build | Tests pass but build fails | Always include build in verification |

## Red Flags — STOP and Re-verify

- Claiming "done" without running tests since last code change
- "Tests were passing" (past tense = stale)
- Skipping build or lint because "only tests matter"
- Reading only the last line of test output
- Trusting a cached or partial test run

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "I just ran it" | After your last edit? Run again. |
| "Only changed a comment" | Comments can break things (JSDoc types, lint rules). Run tests. |
| "Tests take too long" | Better slow than wrong. Run them. |
| "The CI will catch it" | CI is a safety net, not a substitute. Verify locally. |
| "I read the output already" | After the last change? Read it again. |

## Integration with Quick-Dev

This skill is invoked at one point in `meld:meld-quick-dev`:

- **Phase 4 (Final Verification)** — After all per-task loops complete, run the full gate function against the entire codebase. Individual tasks already passed per-task reviews, but this catches integration issues and ensures all claims have fresh evidence.

## Verification for Agent Delegation

When using subagents (via `meld:meld-parallel-agents` or Task tool):

- **Never trust subagent completion claims.** Subagents can hallucinate success.
- After a subagent reports "done," run the gate function yourself on their work.
- Verify their code compiles, tests pass, and output matches expectations.
- If a subagent says "all tests pass" — run the tests yourself and read the output.
