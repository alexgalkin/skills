---
name: commit-n-push
description: Commit the current working-tree changes (signed) and push them to the checked-out feature branch and its open PR.
argument-hint: "[commit message hint] (optional)"
allowed-tools: Bash(git:*), Bash(gh:*), Read, Grep, Glob
---

# Commit and push to the current branch / PR

Message hint: `$ARGUMENTS` — optional; otherwise write the message from the diff.

Invoking this command is explicit authorization, **for this one run only**, to commit the current changes and push
the checked-out branch. Nothing else: no branch switch or creation, no merge, no rebase, no force-push, no
`--no-verify`, no bare `git stash`.

## 1. Pre-flight — stop rather than improvise

```bash
git rev-parse --abbrev-ref HEAD
git status --short
gh repo view --json defaultBranchRef --jq .defaultBranchRef.name
gh pr view --json number,url,state,headRefName,baseRefName
```

Abort and report if:

- HEAD is detached.
- The branch is the repo's default branch, or a long-lived shared branch (e.g. `dev`, `develop`, `staging`,
  `production`, `release/*`, or any branch the repo's docs mark as protected).
- There is nothing to commit.
- A merge/rebase/cherry-pick is in progress.
- The branch's PR is `MERGED` or `CLOSED` — ask which branch to use; never create one.
- `origin/<branch>` has commits the local branch lacks (`git fetch` then `git status -sb` shows "behind") —
  report and stop; do not pull/rebase on your own.

No PR for the branch is fine: push the branch anyway and say there is no PR.

## 2. Review what goes in

- `git diff` and `git status --short`. Stage only files that belong to the change you made in this session.
  Leave unrelated or unknown files (scratch files, local env, other sessions' edits) unstaged and list them in the report.
- If the changes clearly fall into separate concerns, ask whether to split into several commits — unless the user
  already said how many.

## 3. Commit

- Always signed: `git commit -S`. Never `--no-gpg-sign` / `-c commit.gpgsign=false`. If signing fails, stop and report.
- Follow the repo's commit style (check `git log --oneline -20` and any contributing docs). If the branch name
  carries a ticket key, include it the way past commits do, e.g. `fix(ABC-123): <what changed>`.
  Add a short body saying why, when it isn't obvious.
- No `Co-Authored-By: Claude` trailer or any Claude/Anthropic attribution.
- If a pre-commit hook fails, fix the cause and commit again (a new commit attempt, never `--amend` of an
  already-pushed commit); never bypass it.

## 4. Push

```bash
git push origin <branch>
```

Pre-push hooks may run long test suites: use the max Bash timeout (600000 ms).
If a hook blocks, read the failure, fix it in a follow-up commit, and push again. Never push with `--no-verify`.
If hooks appear to be silently skipped because dependencies aren't installed, report it rather than pushing unchecked.

## 5. Report

- Commit SHA(s) and subject(s); `origin/<branch>` before → after.
- Hook results (pass/fail, test counts), or that no hooks ran.
- PR number/URL that now includes the commit.
- Any files left uncommitted, and why.
