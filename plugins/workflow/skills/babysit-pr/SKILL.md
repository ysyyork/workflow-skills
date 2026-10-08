---
name: babysit-pr
description: Monitor a pull request for new review comments and CI failures, fix what is valid, reply inline, and push. Runs as a periodic cron check and stops when the PR is merged, closed, or fully green. Use when the user asks to watch, babysit, or shepherd a PR through review and CI.
argument-hint: <PR_NUMBER> [--interval <minutes>] [--resolve]
allowed-tools: Read, Grep, Glob, Bash(gh *), Bash(git *), Bash(make *), Bash(pytest *), Bash(ruff *), Bash(npm *), Edit, Write, Agent, CronCreate, CronDelete
---

# Babysit PR

Watch a pull request for review comments and CI failures. Fix valid issues, reply to every comment, push, and stop when the PR is done.

**Usage:** `/babysit-pr <PR_NUMBER> [--interval <minutes>] [--resolve]`

- `--interval` sets the polling interval (default 11 minutes).
- `--resolve` auto-resolves threads after a fix-reply. Without it, threads stay open for a human to check.

Replace `{owner}`, `{repo}`, and `{pr_number}` in the commands below with real values.

## 1. Setup

1. Read `{owner}/{repo}` from `git remote -v`.
2. `gh pr checkout <PR_NUMBER>`.

## 2. Initial pass

Check for unresolved threads and failing CI:

```bash
gh api graphql -f query='{
  repository(owner: "{owner}", name: "{repo}") {
    pullRequest(number: {pr_number}) {
      state
      reviewThreads(first: 100) {
        nodes { id isResolved comments(first: 10) { nodes { id databaseId body path line author { login } } } }
      }
    }
  }
}' --jq '.data.repository.pullRequest | {state, unresolved: ([.reviewThreads.nodes[] | select(.isResolved == false)] | length)}'

# Failing, pending, or cancelled checks
gh pr checks {pr_number} --json bucket,name --jq '.[] | select(.bucket == "fail" or .bucket == "pending" or .bucket == "cancel") | [.bucket, .name] | @tsv'

# Full list, to tell "all green" apart from "CI never ran"
gh pr checks {pr_number} --json bucket,name --jq '.[] | [.bucket, .name] | @tsv'
```

Handle unresolved comments (step 5) and CI failures (step 6). Then evaluate the readiness gate (step 4) on the resulting state.

**A green check list doesn't prove HEAD is green.** Checks belong to the commit that triggered them, not necessarily the current head. Compare the run's commit with HEAD:

```bash
git rev-parse HEAD
gh run list --branch <branch> --limit 5 --json headSha,status,conclusion,name \
  --jq '.[] | [.headSha[0:10], .status, (.conclusion // "-"), .name] | @tsv'
```

If the passing runs' `headSha` isn't HEAD, CI hasn't tested the current commit. Treat it as absent and re-trigger it under step 4 if your repo gates CI.

**An empty failure list isn't "all green."** It means all green only when the expected CI checks actually ran. If the full list has no real CI checks, CI never ran on this commit.

## 3. Cron monitor

Create a recurring job with `CronCreate` using `*/N * * * *`, where N is the interval. Each cycle:

1. `git pull --rebase`. If it conflicts, abort and report. Don't resolve automatically.
2. Check unresolved threads and the latest automated-review verdict (see below).
3. Check CI.
4. Act on the first match, in this order. Comment handling comes before CI, so CI only runs on a fully addressed PR:
   - PR merged or closed: delete the cron and write the final summary (step 9).
   - Unaddressed review comments or reviewer findings (the last comment in the thread isn't a `[Claude Code]` reply): fix or push back, reply inline (step 5), commit and push. Don't trigger CI.
   - Last fix pushed, but the reviewer hasn't re-reviewed the new commit: wait for the next cycle.
   - CI failed on a fully addressed PR: investigate, fix, push, re-trigger CI (step 6).
   - Fully addressed but CI absent for this commit: trigger CI once, then wait.
   - All CI green on HEAD and fully addressed: delete the cron and write the final summary.
   - CI cancelled: report and wait.
   - CI pending: brief status, then wait.

A thread counts as handled when its last comment starts with `[Claude Code]`. If a human replies after that, process the thread again.

## 4. Readiness gate and CI trigger

If your repo runs expensive CI only on demand (a label, a comment, or a manual dispatch), don't trigger it on every push. Trigger it when the PR is **fully addressed**:

- (a) Every review thread's last comment is a `[Claude Code]` reply.
- (b) Every automated reviewer's finding on the latest commit has a `[Claude Code]` reply.

Before triggering, confirm the PR can merge. A conflicted PR may start no CI at all, which looks like success:

```bash
gh pr view {pr_number} --json mergeable,mergeStateStatus --jq '{mergeable, mergeStateStatus}'
```

If it's `CONFLICTING` or `DIRTY`, merge or rebase the base in, push, and then trigger CI.

### Automated reviewers

Bots may review the PR. Query for the reviewers actually active; don't hardcode them.

```bash
gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews --paginate \
  --jq '.[] | select(.user.login | endswith("[bot]")) | {bot: .user.login, state, commit_id, submitted_at}' \
  | jq -s 'sort_by(.submitted_at) | last'
```

Compare the review's `commit_id` against `git rev-parse HEAD`. Both are full SHAs.

Some bots signal "clean" differently. One may post a reaction instead of a review. When the clean signal is older than HEAD and nothing new has appeared within about two cycles, treat the PR as addressed. Don't wait forever for a verdict that won't come. Any finding that appears later is caught by the next cycle.

## 5. Review comments

Evaluate each comment on its merits.

- Fix valid suggestions.
- Push back only with concrete evidence that the comment is wrong. Reply with the evidence and leave the thread open, so a human can re-engage.
- If unsure, fix instead of pushing back.

Low-priority automated findings (below P2 or the equivalent) need a higher bar. Before changing code, confirm all of these:

1. The finding describes a concrete defect or a violated invariant, not a style preference or hypothetical edge case.
2. You can show why the current code produces the bad outcome, from the code, tests, docs, or a minimal reproduction.
3. The proposed change fixes that outcome without changing intended behavior or adding disproportionate complexity.
4. The benefit justifies the diff and its review cost.

If the finding doesn't clear the bar, decline with a short, evidence-based reply. Leave that thread open.

Reply inline:

```bash
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments \
  --method POST \
  -f body="[Claude Code] Fixed in <commit_hash>" \
  -F in_reply_to=<comment_database_id>
```

Don't resolve threads unless `--resolve` was passed. Don't trigger CI after a comment-only fix push. Let the next cycle pick up the reviewer's re-review.

## 6. CI failures

Fetch the failing logs with `gh run view <run_id> --log-failed`. Fix failures caused by the PR. For flaky or unrelated failures, note them in the status and don't change code for them.

After a CI-caused fix push, re-trigger CI if your repo gates it. This is the only point where a fix push should trigger CI, since comments are already clean by then.

## 7. Per-cycle status

One line per cycle:

- `PR #N: no changes, CI pending.`
- `PR #N: fixed 2 comments, pushed abc1234.`
- `PR #N: blocked, CI infrastructure failure, needs a human.`

## 8. Cleanup

The cron expires after three days. Stop it when:

- The PR is fully addressed and CI is green on HEAD, with gated checks that actually ran on it. A green run on an earlier commit doesn't count.
- The user asks you to stop.
- The PR is merged or closed.

## 9. Final summary

Rebuild the summary from `git log` and the `[Claude Code]` replies:

- **Comments addressed:** each comment and whether it was fixed or pushed back, grouped by file.
- **CI failures fixed:** what broke and how it was fixed.
- **Commits pushed:** hashes and descriptions.
- **Current state:** ready for review, or what still needs a human.
