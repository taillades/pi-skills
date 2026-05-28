---
name: fix-tests
description: Start by reading GitHub CI logs to locate failing tests, make simple obvious fixes, validate them, commit and push when the fix is straightforward. If failures are ambiguous or require design choices, ask questions and propose a plan before changing code.
argument-hint: "Optional test command, target, or failure log."
allowed-tools: ["git *", "bazel test*", "bazel build*", "./grind format*", "./grind lint*", "python *", "pytest *", "bin/gh pr view*", "bin/gh run view*", "bin/gh run list*", "bin/gh api*", "find *"]
---

# Fix Tests

Use this skill when the user wants tests fixed, especially phrasing like "fix tests", "CI is failing", "make tests pass", "fix this failure", or `/skill:fix-tests`.

## Core rules

- Start by reading GitHub CI logs when a PR/branch has CI failures, so you know exactly where tests failed before running local commands.
- Prefer the narrowest failing test target/command first.
- Make and push fixes only when they are simple, local, and clearly implied by the failure.
- If the failure is ambiguous, broad, flaky, infrastructure-related, or requires product/design judgment, stop and ask questions with a concrete proposed plan.
- Do not paper over failures by deleting tests, weakening assertions, skipping tests, increasing timeouts, or swallowing exceptions unless the user explicitly approves and the reason is documented.
- Preserve unrelated user changes.
- Before committing/pushing, validate the exact failing test and a reasonable adjacent check.

## What counts as simple enough to fix and push

Usually simple:
- missing Bazel deps/imports
- stale snapshots/golden files when the intended output is obvious
- renamed symbols/files after a refactor
- lint/format failures
- deterministic typo or type error
- test fixture setup mismatch with obvious expected API
- updating test expectations for already-intended behavior

Ask first:
- semantic behavior changes
- model/science/metric correctness questions
- flaky/concurrency/timing failures without clear root cause
- large refactors
- deleting or skipping tests
- changing public APIs
- failures involving credentials, external systems, or expensive jobs
- fixes touching many unrelated modules

## Workflow

1. **Identify current branch and dirty state**
   ```bash
   git status --porcelain=v2 -b --ignore-submodules=all
   git branch --show-current
   ```
   - If there are existing uncommitted changes, summarize them and avoid overwriting unrelated work.

2. **Start with GitHub logs**
   - Unless the user provided a complete local failure log and explicitly says not to inspect CI, look at the GitHub PR/check logs first.
   - Find the current PR and recent failing runs:
     ```bash
     ./bin/gh pr view --json number,url,headRefName,statusCheckRollup
     ./bin/gh run list --branch $(git branch --show-current) --limit 10
     ```
   - Read failed job logs before guessing:
     ```bash
     ./bin/gh run view <run-id> --log-failed
     ```
   - If multiple runs failed, inspect the newest relevant failure for the current head commit.
   - Capture the failing workflow/job, command/target, and first real error, not just follow-on errors.
   - If GitHub logs are unavailable or inconclusive, say so and fall back to the provided command/log or ask for the failing command.

3. **Reproduce narrowly**
   - Run the provided command or the narrowest target identified from GitHub logs, for this repo using `--config=agent`:
     ```bash
     bazel test --config=agent <target>
     ```

4. **Diagnose**
   - Map the failure to the smallest responsible file/change.
   - Check whether the failure is already fixed locally.
   - Decide whether it is simple enough to fix without asking.

5. **If not simple, ask before editing**
   - Summarize:
     - failing command/target
     - root-cause hypothesis
     - why the fix is not obvious
     - 1-3 proposed fix options
     - recommended validation
   - Ask the user which direction to take.

6. **If simple, fix**
   - Make the smallest code/test change that addresses the root cause.
   - Do not introduce private/underscored class names.
   - Follow repo style and existing patterns.
   - Run formatting if needed:
     ```bash
     ./grind format
     ```

7. **Validate**
   - Re-run the originally failing test/command.
   - Run adjacent tests or build targets when cheap and relevant.
   - If validation reveals a second unrelated failure, stop and ask unless it is also simple and local.

8. **Commit and push simple fixes**
   - If the fix is simple and validation passes:
     ```bash
     git status --porcelain=v2 -b --ignore-submodules=all
     git diff --stat
     git add <changed-files>
     git commit -m "Fix failing tests"
     git push
     ```
   - If push is rejected, fetch and inspect. Ask before rebasing/force-pushing unless the user has already authorized that workflow.

9. **Final response**
   - Include:
     - failing test/command
     - root cause
     - files changed
     - validation run/results
     - commit SHA and push result if pushed
     - any remaining failures/questions

## If the user only wants a plan

Do not edit. Run/inspect enough to diagnose, then provide the plan and ask for approval.
