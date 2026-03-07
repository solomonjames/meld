---
name: meld-worktrees
description: Git worktree creation with ticket-based branch naming — isolate work in a separate directory with automatic setup and lifecycle management. Referenced by quick-dev Phase 1 and 6.
---

# Git Worktrees

## When to Use

- Starting work on a ticket that should be isolated from main
- Multi-agent workflows where each agent needs its own working copy
- Any work that benefits from a clean checkout without stashing

## Directory Selection

Priority order for worktree location:

1. **Existing `.worktrees/` directory** — if present, use it
2. **CLAUDE.md preference** — if the project specifies a worktree directory, use it
3. **Ask the user** — prompt for location if neither exists

## Branch Naming

### With Beads Active

When `{beads_active}` is true and a ticket ID is available:

```
feature/{ticket_id}
```

If the ticket metadata contains `spec_slug`:

```
feature/{ticket_id}-{spec_slug}
```

Examples:
```
feature/meld-abc123
feature/meld-abc123-avatar-upload
```

### Without Beads

Use a kebab-case description of the work:

```
feature/{kebab-case-description}
```

Examples:
```
feature/add-avatar-upload
feature/fix-notification-timing
```

## Creation

### 1. Safety Check

Verify `.worktrees/` is in `.gitignore`:

```bash
grep -q '.worktrees' .gitignore 2>/dev/null
```

If not present, add it:

```bash
echo '.worktrees/' >> .gitignore
```

### 2. Create the Worktree

```bash
# Determine branch name (see Branch Naming above)
BRANCH="feature/{ticket_id}"

# Create worktree
git worktree add .worktrees/{branch_name} -b {branch_name}
```

If the branch already exists:

```bash
git worktree add .worktrees/{branch_name} {branch_name}
```

### 3. Project Setup

Auto-detect and run project setup based on files present:

| File | Setup Command |
|------|--------------|
| `package.json` + `package-lock.json` | `npm install` |
| `package.json` + `bun.lockb` | `bun install` |
| `package.json` + `yarn.lock` | `yarn install` |
| `package.json` + `pnpm-lock.yaml` | `pnpm install` |
| `Gemfile` | `bundle install` |
| `requirements.txt` | `pip install -r requirements.txt` |
| `go.mod` | `go mod download` |
| `Cargo.toml` | `cargo build` |

### 4. Baseline Test Verification

Run the project's test suite to establish a baseline:

```bash
# In the worktree directory
npm test  # or equivalent
```

If tests fail on a clean checkout, note the failures — they're pre-existing, not caused by your work.

### 5. Record Worktree State

Store worktree info for later phases:

```
worktree_path: .worktrees/{branch_name}
worktree_branch: {branch_name}
baseline_commit: {HEAD of the new worktree}
```

If `{beads_active}`:
- Read-merge-write metadata: merge `{"worktree_path": "{path}", "worktree_branch": "{branch_name}"}`

## Development

All work happens in the worktree directory. The main checkout remains clean.

Key reminders:
- Run all commands from the worktree path
- Git operations (commit, diff, log) work normally within the worktree
- The worktree shares the same `.git` storage — commits are visible from both directories

## Completion

When work is done (quick-dev Phase 5), present these options:

### Option 1: Create Pull Request

```bash
cd {worktree_path}
git push -u origin {branch_name}
gh pr create --title "{PR title}" --body "{PR body}"
```

### Option 2: Merge to Main

```bash
cd {main_checkout}
git merge {branch_name}
git worktree remove {worktree_path}
git branch -d {branch_name}
```

### Option 3: Keep for Later

Leave the worktree in place. User can return to it later.

### Option 4: Discard

```bash
git worktree remove {worktree_path}
git branch -D {branch_name}
```

**Always ask the user which option they prefer.** Never auto-discard.

## Integration with Quick-Dev

This skill is invoked at two points in `meld:meld-quick-dev`:

1. **Phase 1 (Setup & Mode Detection)** — After "Capture Baseline Commit":
   - If `{beads_active}`: auto-create worktree with ticket-based branch name
   - If not: offer worktree as an option
   - Re-capture baseline commit from worktree HEAD

2. **Phase 5 (Completion)** — After final verification:
   - If in a worktree, present completion options (PR/Merge/Keep/Discard)
   - User decides what to do with the worktree
