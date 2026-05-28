---
name: rebase-main-into-branch
description: Safely update the current feature branch by rebasing it onto latest main. Fetch main, check for dirty worktree, run the rebase, handle conflicts by asking when unsure, validate, and ask before force-pushing.
argument-hint: "Optional base branch, default main."
allowed-tools: ["git *", "./grind format*", "./grind lint*", "bazel test*"]
---

# Rebase Main Into Branch

Use this skill when the user wants to update the current branch with latest `main`, especially phrasing like "rebase main into this branch", "rebase onto main", "update from main", or `/skill:rebase-main-into-branch`.

## Core rules

- Default base is `main` / `origin/main` unless the user specifies another base.
- Do not start a rebase with uncommitted changes unless the user explicitly approves stashing or committing them.
- Prefer `git rebase origin/main` over merging main into the branch.
- Ask before resolving non-trivial conflicts. Do not guess semantic conflict resolutions.
- Do not force-push unless the user explicitly confirms after the rebase.
- Never use `git reset --hard`, `git rebase --abort`, or destructive cleanup without telling the user what will happen. Abort is okay when the user asks or conflict resolution is not possible.

## Workflow

1. **Inspect current branch and worktree**
   ```bash
   git status --porcelain=v2 -b --ignore-submodules=all
   git branch --show-current
   git remote -v
   ```
   - If on `main` or detached HEAD, stop and ask what branch should be rebased.
   - If dirty, summarize changes and ask whether to stash, commit, or stop.

2. **Fetch latest base**
   ```bash
   git fetch origin main --prune
   ```
   - If the user specified a different base, fetch that base instead.

3. **Show what will be replayed**
   ```bash
   git log --oneline --decorate origin/main..HEAD
   git diff --stat origin/main...HEAD
   ```
   - Confirm if the branch appears unexpectedly large or unrelated.

4. **Run the rebase**
   ```bash
   git rebase origin/main
   ```
   - If no conflicts, continue to validation.
   - If conflicts occur:
     - Show conflicted files with `git status`.
     - Inspect conflict hunks.
     - Resolve obvious mechanical conflicts only.
     - Ask the user before semantic choices or when unsure.
     - After resolving, `git add <files>` and `git rebase --continue`.

5. **Validate**
   - Run appropriate checks for changed files if reasonable.
   - For this repo, use targeted Bazel tests with `--config=agent`; run `./grind format` if conflict resolutions touched formatted files.
   - If validation fails, report failures and ask how to proceed.

6. **Ask before push**
   - Show final status:
     ```bash
     git status --porcelain=v2 -b --ignore-submodules=all
     git log --oneline --decorate -5
     ```
   - Ask: `Should I force-push with lease?`

## If the user says yes to push

1. Push safely:
   ```bash
   git push --force-with-lease
   ```
2. Report the resulting HEAD SHA and push result.

## Final response before push

Include:
- branch rebased
- base commit used
- conflicts encountered and how resolved
- validation run and results
- current HEAD SHA
- whether push is still pending
