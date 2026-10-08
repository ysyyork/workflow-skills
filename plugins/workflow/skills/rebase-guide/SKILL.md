---
name: rebase-guide
description: Rebase the current branch cleanly onto a base branch (default main). Squashes only when the branch and base overlap in files, makes a backup ref first, resolves conflicts while keeping both sides' features, verifies files that git merged silently, and requires a conflict report afterward. For a chain of stacked branches, rebase each one in order instead.
argument-hint: "[base-branch] (defaults to main)"
allowed-tools: Bash(git *), Read, Edit, AskUserQuestion
user-invocable: true
---

# Rebase Guide

Keep history clean, keep features from both branches, and make conflict resolution traceable and reviewable.

**Usage:** `/rebase-guide [base-branch]`. `base-branch` defaults to `main`.

This skill rebases a **single branch** onto its base. If each branch in a stack is based on the previous one, rebase them in order, bottom first, each onto the branch below it.

## Before you start

1. **Clean worktree:** `git status` should show nothing you don't intend to carry.
2. **Fetch and measure:**
   ```bash
   git fetch origin <base-branch>
   git rev-list --count origin/<base-branch>..HEAD   # ahead
   git rev-list --count HEAD..origin/<base-branch>   # behind
   ```
3. **Backup ref** before touching anything:
   ```bash
   git branch backup/<branch-name>-pre-rebase
   ```
4. **Squash only when conflicts are possible, not based on commit count.** Squashing exists to avoid re-resolving the same conflict once per commit. First check whether there's anything to re-resolve:
   ```bash
   MB=$(git merge-base HEAD origin/<base-branch>)
   comm -12 <(git diff --name-only $MB..HEAD | sort) \
            <(git diff --name-only $MB..origin/<base-branch> | sort)
   ```
   - **No overlapping files:** don't squash, whatever the commit count. Squashing destroys messages that explain each change.
   - **Overlapping files and more than about three commits:** squash, then rebase once.

   Squash non-interactively:
   ```bash
   git log --oneline origin/<base-branch>..HEAD      # draft the aggregated message from this
   git reset --soft $(git merge-base HEAD origin/<base-branch>)
   git commit -m "<aggregated message>"
   ```
   > Never `git reset --soft origin/<base-branch>` directly. If the base has moved since you branched, that pulls unrelated changes into the diff. Always reset to the merge-base.

## Rebase

1. `git rebase origin/<base-branch>`
2. **On each conflict:**
   - Find out why it exists: `git log --oneline -- <file>`, and `git show <commit>` for context.
   - Keep the functionality from both branches. Don't drop a feature to make a conflict go away.
   - If you can't see how to keep both, stop and ask the user.
   - `git add <resolved-files>` then `git rebase --continue`.
3. **Verify files git merged quietly.** The conflict list only covers files git couldn't merge. Non-overlapping hunks can merge silently even when both sides changed adjacent logic, and a bad interleaving can drop a guard clause or let two handlers overwrite each other. For each file both sides changed (the `comm` command above):
   - grep the merged file for a marker from each side and check both survive;
   - re-run the tests that cover both;
   - check that co-existing constructs don't cancel each other out.
4. **Check the branch against the new base**, even with zero overlap. A large base jump can change an API you depend on without touching your files. Confirm that the symbols you call still exist upstream, then run the tests and lint.

## If the rebase aborts midway

A failed checkout can leave many modified and untracked files while `HEAD` is unchanged. That looks like data loss, but it usually isn't.

1. Confirm the damage is worktree-only: `git rev-parse HEAD`, and check that `.git/rebase-merge` and `.git/rebase-apply` are absent. The backup ref and `origin/<branch>` should still point at the original tip.
2. Fix the cause (commonly files owned by another user or left by a container run), then `git reset --hard HEAD`.
3. Before `git clean -fd`, check each untracked path is checkout residue, not your work: `git cat-file -e origin/<base-branch>:<path>`.
4. Re-run the rebase.

## When confidence is low

If there are many conflicts, or you can't follow the logic well enough to be sure, pause and raise it with the user before continuing.

## Conflict report (required after the rebase)

One entry per resolved conflict:

```
Rebase conflict report

1) <file path>
   - Conflict cause: <short explanation>
   - Resolution: <what you kept or changed>
   - Why: <rationale and behavior impact>
```

If the rebase was clean, say so, and list what you verified (steps 3 and 4). "No conflicts" must not be mistaken for "not checked."

## Rollback

The backup ref is the escape hatch. Coordinate with the user before resetting to it.
