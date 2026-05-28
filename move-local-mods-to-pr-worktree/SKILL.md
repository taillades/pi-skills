---
name: move-local-mods-to-pr-worktree
description: Move selected local modifications into a fresh git worktree branched from main, commit them, and create a new pull request. Always ask clarifying questions first so the user defines the exact scope and PR context.
argument-hint: "Describe which local changes should move and add PR context."
allowed-tools: ["git *", "bin/gh pr create*", "bin/gh pr view*", "bin/gh repo view*", "./grind format*", "./grind lint*", "bazel test*", "mkdir *", "rsync *", "cp *", "find *"]
---

# Move Local Mods To PR Worktree

Use this skill when the user wants to split local uncommitted or branch-local modifications into a new pull request against `main`, using a separate git worktree.

Examples:
- "move these local changes to a new PR"
- "split this off into a worktree PR"
- "put my current edits in a fresh branch off main"
- `/skill:move-local-mods-to-pr-worktree`

## Core rules

- Start by asking clarifying questions. The user must define the exact scope of what should move.
- Ask the user to add context to the agent prompt: why this PR exists, intended title, risky/irrelevant changes, tests to run, and any reviewer notes.
- Do not move all local changes by default unless the user explicitly says all changes are in scope.
- Do not discard, reset, or overwrite user work.
- Do not commit or push until the user approves the staged diff in the new worktree.
- Create the new branch from latest `origin/main`/`main`, not from the current feature branch, unless the user explicitly requests a different base.
- If any patch does not apply cleanly to `main`, stop and ask the user how to resolve it.

## Initial questions

Before editing or creating the worktree, ask concise questions unless the user's prompt already answers them:

1. Which changes should move?
   - all local modifications, or only specific files/hunks?
   - include untracked files?
   - include staged changes?
2. What should stay behind in the current worktree?
3. What branch name should the new PR use?
4. What PR title/body context should be used?
5. Should the agent commit and push after showing the new worktree diff, or stop for review first?
6. What validation should run?

If the user gives incomplete scope, inspect status/diff and ask targeted follow-ups with the file list.

## Workflow

1. **Inspect current state**
   ```bash
   git status --porcelain=v2 -b --ignore-submodules=all
   git diff --no-color --minimal -w --no-prefix --word-diff -U1
   git diff --cached --no-color --minimal -w --no-prefix --word-diff -U1
   ```
   - Include untracked files in the summary.
   - Group changes by likely concern if possible.
   - Ask the user to confirm exact files/hunks to move.

2. **Determine base and new worktree path**
   - Default base: `origin/main` if available, otherwise `main`.
   - Suggested branch name format: `<user-or-topic>/<short-purpose>`.
   - Suggested worktree path: sibling of repo root, e.g. `../atmo-<branch-slug>`.
   - Confirm branch/path with the user if ambiguous.

3. **Prepare patch from selected changes**
   - For all selected tracked changes, create a patch:
     ```bash
     git diff --binary -- <paths> > /tmp/<slug>.patch
     git diff --cached --binary -- <paths> >> /tmp/<slug>.patch
     ```
   - For selected untracked files, copy them separately or use `git diff --no-index /dev/null <file>` when appropriate.
   - If only selected hunks are in scope, use `git diff` inspection and ask before manual patch surgery.

4. **Create fresh worktree from main**
   ```bash
   git fetch origin main
   git worktree add -b <new-branch> <worktree-path> origin/main
   ```
   - If branch exists, ask whether to reuse it, choose a new name, or stop.
   - If worktree path exists, ask before using it.

5. **Apply selected changes in the new worktree**
   ```bash
   git -C <worktree-path> apply --index /tmp/<slug>.patch
   ```
   - Copy selected untracked files into the matching paths in the new worktree, then `git -C <worktree-path> add <files>`.
   - If patch application fails, stop. Show reject/error summary and ask how to proceed.

6. **Review new worktree diff with the user**
   ```bash
   git -C <worktree-path> status --porcelain=v2 -b --ignore-submodules=all
   git -C <worktree-path> diff --cached --no-color --minimal -w --no-prefix --word-diff -U1
   git -C <worktree-path> diff --no-color --minimal -w --no-prefix --word-diff -U1
   ```
   - Summarize exactly what moved.
   - Confirm nothing out-of-scope moved.
   - Ask before committing/pushing unless the user already explicitly authorized this step.

7. **Validate**
   - Run formatting/tests requested by the user or appropriate narrow checks.
   - For this repo, prefer:
     ```bash
     ./grind format
     bazel test --config=agent <targets>
     ```
   - Run commands from inside the new worktree.
   - Report failures honestly and ask before broadening scope.

8. **Commit and push after approval**
   ```bash
   git -C <worktree-path> add <selected-files>
   git -C <worktree-path> commit -m "<message>"
   git -C <worktree-path> push -u origin <new-branch>
   ```

9. **Create PR against main**
   - Generate PR body from user-provided context plus actual diff.
   - Use:
     ```bash
     ./bin/gh pr create --repo <owner/repo> --base main --head <new-branch> --title "<title>" --body "<body>" --assignee "@me"
     ```
   - If running from the new worktree, prefer its local `./bin/gh` if present; otherwise use the original repo's `./bin/gh` with `--repo`.

10. **Optionally remove moved changes from original worktree**
    - Only do this if the user explicitly asks.
    - Before removing, show exactly what would be reverted/deleted in the original worktree.
    - Never run broad `git reset --hard` or `git clean` without explicit confirmation.

## Final response

Include:
- original repo path and new worktree path
- base branch/commit used
- new branch name
- files/hunks moved
- validation run and results
- commit SHA if committed
- PR URL if created
- anything left behind in the original worktree
- any unresolved questions
