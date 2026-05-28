---
name: address-pr-comments
description: Inspect reviewer and author comments on the current GitHub pull request, ask when the requested change is unclear, make the requested code changes, do not commit or push, then summarize changes and ask whether to commit and push.
argument-hint: "Optional PR number or extra instructions."
allowed-tools: ["bin/gh pr view*", "bin/gh api*", "bin/gh repo view*", "git status*", "git diff*", "git rev-parse*", "git branch*", "git add*", "git commit*", "git push*", "./grind format*", "./grind lint*", "bazel test*"]
---

# Address PR Comments

Use this skill when the user wants reviewer feedback on a GitHub PR addressed, especially phrasing like "look at PR comments", "address review comments", "fix reviewer feedback", or `/skill:address-pr-comments`.

## Core rules

Do **not** commit or push during the initial pass. Make edits only. At the end, provide a concise breakdown of what changed and explicitly ask whether to commit and push.

Only commit and push after the user gives explicit confirmation.

When you are not sure what the reviewer wants, whether a suggestion is correct, or how to choose between plausible fixes, stop and ask the user before editing that part. Do not guess on ambiguous review feedback.

## Workflow

1. **Identify the PR**
   - If the user provided a PR number, use it.
   - Otherwise, run `./bin/gh pr view --json number,url,title,headRefName,baseRefName` for the current branch.
   - If no PR exists, stop and ask the user for the PR number or branch.

2. **Check the working tree before editing**
   - Run `git status --porcelain=v2 -b -uno --ignore-submodules=all`.
   - If there are existing uncommitted changes, note them and avoid overwriting unrelated work. Use `git diff` as needed to distinguish pre-existing changes from your edits.

3. **Collect review feedback**
   - Start with:
     ```bash
     ./bin/gh pr view <PR> --json number,title,url,body,comments,reviews,files,headRefName,baseRefName
     ```
   - Identify the current GitHub user, so comments authored by the user can be treated as instructions or clarifications:
     ```bash
     ./bin/gh api user --jq .login
     ```
   - Also fetch inline review threads with GraphQL when needed:
     ```bash
     REPO_JSON=$(./bin/gh repo view --json owner,name)
     OWNER=$(printf '%s' "$REPO_JSON" | jq -r '.owner.login')
     REPO=$(printf '%s' "$REPO_JSON" | jq -r '.name')
     ./bin/gh api graphql \
       -F owner="$OWNER" \
       -F repo="$REPO" \
       -F number=<PR_NUMBER> \
       -f query='query($owner:String!, $repo:String!, $number:Int!) {
         repository(owner:$owner, name:$repo) {
           pullRequest(number:$number) {
             reviewThreads(first:100) {
               nodes {
                 isResolved
                 path
                 line
                 originalLine
                 comments(first:50) {
                   nodes {
                     author { login }
                     body
                     url
                     createdAt
                     diffHunk
                     path
                     line
                     originalLine
                   }
                 }
               }
             }
           }
         }
       }'
     ```
   - If `jq` is unavailable, use `./bin/gh pr view` output directly or ask for permission to proceed with a simpler view.
   - Read the full conversation in each review thread, not just the first reviewer comment. Pay special attention to comments by the current GitHub user; treat them as guidance about the desired resolution.
   - Also inspect top-level PR comments (`comments`) and review bodies (`reviews[].body`) for follow-up guidance from the user or reviewers. GitHub PRs are issues too; treat these issue-style comments as part of the review context.
   - If the PR body or comments reference related issues that appear relevant to the requested feedback, inspect those issue comments too, especially comments by the current GitHub user:
     ```bash
     ./bin/gh issue view <ISSUE_NUMBER> --comments
     ```

4. **Triage comments**
   - Identify actionable reviewer requests.
   - For each thread, consider later comments by the current GitHub user before deciding what to change; the user may have accepted, rejected, narrowed, or clarified a reviewer suggestion.
   - Ignore or call out resolved/non-actionable comments, praise, duplicates, and comments already addressed by current code.
   - If a comment is ambiguous, conflicts with another requirement, conflicts with the user's comments, or has multiple reasonable fixes, ask before editing that item.

5. **Make changes**
   - Edit code to satisfy actionable comments.
   - Prefer small, focused changes.
   - Preserve unrelated user changes.
   - Follow repository style and instructions.

6. **Validate**
   - Run the narrowest useful checks for the files changed.
   - If appropriate for the repo, run formatting and linting (for this repo: `./grind format` and targeted `./grind lint`/tests when reasonable).
   - Do not hide failures. If checks cannot be run, explain why.

7. **Final response before commit**
   - Do **not** commit or push yet.
   - Include:
     - PR inspected (number/title/url)
     - Reviewer comments addressed, grouped by reviewer or file
     - Files changed and why
     - Validation run and results
     - Any comments not addressed and why
     - Any questions still requiring user judgment
   - End with: `Should I commit and push these changes?`

## If the user says yes

1. Re-check `git status` and `git diff`.
2. Run formatting/tests if not already done or if edits changed after validation.
3. Stage only relevant files.
4. Create a clear commit message summarizing the reviewer-feedback fixes.
5. Push the current branch.
6. Report the commit SHA and push result.
