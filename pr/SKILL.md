---
description: Create or update a pull request.
argument-hint: "Title for the pull request. Optionally include reviewer with ~username."
allowed-tools: ["bin/gh pr create*", "bin/gh pr edit*", "bin/gh pr view*", "git commit*"]
---
You are an expert software developer assisting with the final step of a feature.
The user wants to create or update a pull request.

**Task:**
1. **Parse the arguments:**
   * Extract the PR title from the user's argument.
   * Check if the argument contains a reviewer mention (e.g., "~username" or "review by ~username").
   * If a reviewer is specified, extract the username (without the ~ symbol).
2. **Ensure changes are committed:**
   * Run `git status` to check if there are uncommitted changes.
   * If there are unstaged or staged changes that haven't been committed, run `./grind format` and commit them first before creating/updating the PR.
   * Do NOT stash changes - always commit them.
3. **Check if PR already exists:**
   * Run `./bin/gh pr view --json number,title,body` to check if a PR exists for the current branch.
   * If a PR exists, note the PR number for updating it later.
4. **Analyze the changes:** Run `git diff` against the base branch to understand the specific changes that will be included in the pull request.
5. **Generate the PR body** using this format:

   ```
   ## Summary

   A short, focused summary of what the change does.

   ## Description

   Detailed explanation of the change.

   ## Risks

   (Only include this section if there are meaningful risks)
   ```

   **Writing guidelines:**
   * **Overview**: Write in imperative mood (e.g., "Add feature X" not "Adding feature X"). Keep it informative enough for someone skimming version control history without reading the full description or having code context.
   * **Description**: Explain the "why" - the problem being solved and why this approach was chosen. Include decisions not reflected in the code, potential shortcomings, or alternative approaches considered. Add references to bug numbers, benchmarks, or design docs if applicable.
   * **Risks**: Only include when there are meaningful risks such as breaking changes, migrations, performance implications, security considerations, or dependencies on external systems. Omit this section entirely for low-risk changes.

6. **Create or update the PR:**
   * If a PR **does not exist**, use `./bin/gh pr create`:
     - Set the title using `--title "$TITLE"` (excluding any reviewer mentions).
     - Set the body using the generated description with `--body "$PR_BODY"`.
     - Always add `--assignee "@me"` to assign the PR to the current user.
     - If a reviewer was specified, add `--reviewer "username"` to the command.
     - Do not publish the PR as a draft unless explicitly instructed.
   * If a PR **already exists**, use `./bin/gh pr edit`:
     - Set the title using `--title "$TITLE"` (excluding any reviewer mentions).
     - Set the body using the generated description with `--body "$PR_BODY"`.
     - Always add `--add-assignee "@me"` to assign the PR to the current user if not already assigned.
     - If a reviewer was specified, add `--add-reviewer "username"` to the command.
     - Inform the user that the existing PR has been updated with the new title and description.