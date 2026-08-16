---
name: checkout-branch
description: Orient on the current git branch when the user says "check out branch". Read-only: never switches or checks out a different branch, only reports on the one you're already on. Runs git branch, git log, git status, git diff HEAD, and gh pr view to review recent commit history, uncommitted changes, and any open PR before starting work.
---

## When to Use

- The user says "check out branch," with or without a branch name after it, if they just say "checkout branch" without naming one, assume it's the branch you're currently on
- Orienting on the branch already checked out, before starting work

# Check out branch (orientation)

When the user says **"check out branch"** (with or without a branch name after it: this skill always orients on whatever branch is currently checked out, it never runs `git checkout` to switch), run these in the current working directory, in order:

## Gather

- `git branch`, identify the current branch
- `git log --oneline -10`, review recent commit history and understand what's been done and why
- `git status`, see the current working tree state
- `git diff HEAD`, review all uncommitted changes
- `gh pr view --json title,state,url,body,reviewDecision,statusCheckRollup`, check whether a PR already exists for this branch. If the command errors (no PR open), note that and move on, don't treat it as a failure.

All five are read-only.

## Report

Summarize what you find (current branch, recent commits, uncommitted changes, and PR title/state/review/CI status if one exists) rather than dumping raw output.
