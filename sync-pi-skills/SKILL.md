---
name: sync-pi-skills
description: Sync local Pi skills with a GitHub-backed skills repository. Pull remote changes, merge with ~/.pi/agent/skills, commit local skill changes when needed, and push back to GitHub. Ask before resolving conflicts or destructive changes.
argument-hint: "Optional repo path or GitHub repo URL/name."
allowed-tools: ["git *", "gh repo clone*", "gh repo view*", "rsync *", "find *", "mkdir *", "cp *", "rm *"]
---

# Sync Pi Skills

Use this skill when the user wants to sync local Pi skills with a GitHub repository, especially phrasing like "sync pi skills", "push my skills", "pull skills", or `/skill:sync-pi-skills`.

## Defaults

- Local Pi skills directory: `~/.pi/agent/skills`
- Repo working tree, in priority order:
  1. Path or repo argument supplied by the user.
  2. `$PI_SKILLS_REPO_DIR` if set.
  3. `~/.pi/skills-repo`.
- If the repo path does not exist and the user provided a GitHub repo URL/name, clone it there (or to the provided path if the argument is a path + URL).
- If no repo can be identified, ask the user for the GitHub repo URL/name or local repo path.

## Core rules

- Sync is bidirectional: pull/merge remote updates into the repo, copy them into local Pi skills, copy local Pi skill changes back into the repo, then push.
- Ask before any destructive action: deleting files, overwriting conflicting changes, resetting, force-pushing, or resolving non-trivial merge conflicts.
- If you are not sure which version of a skill should win, ask the user.
- Never force push unless explicitly instructed.
- Preserve unrelated files in the repo unless the repo is clearly dedicated to Pi skills.

## Workflow

1. **Identify paths and repo**
   - Resolve the local skills directory:
     ```bash
     printf '%s\n' "${PI_SKILLS_DIR:-$HOME/.pi/agent/skills}"
     ```
   - Resolve the repo working tree from the user argument, `$PI_SKILLS_REPO_DIR`, or `~/.pi/skills-repo`.
   - If the repo is missing and a GitHub repo URL/name was provided, clone it:
     ```bash
     gh repo clone <repo> <repo-dir>
     ```
   - Verify the repo:
     ```bash
     git -C <repo-dir> status --porcelain=v2 -b --ignore-submodules=all
     git -C <repo-dir> remote -v
     ```

2. **Inspect local and repo state**
   - Check local skills:
     ```bash
     find <local-skills-dir> -maxdepth 2 -name SKILL.md -print
     ```
   - Check repo skills layout. Common accepted layouts:
     - repo root contains skill directories with `SKILL.md`
     - repo contains a `skills/` directory with skill directories
   - If the layout is unclear, ask the user which repo subdirectory should mirror local skills.

3. **Pull remote changes first**
   - Fetch and pull with merge:
     ```bash
     git -C <repo-dir> fetch --all --prune
     git -C <repo-dir> pull --no-rebase
     ```
   - If pull creates conflicts, stop and ask the user how to resolve them. Do not auto-resolve semantic conflicts.

4. **Merge repo → local skills**
   - Copy new/updated skills from the repo skills directory into local Pi skills.
   - Use a dry-run first when possible:
     ```bash
     rsync -avnc --delete --exclude '.git/' <repo-skills-dir>/ <local-skills-dir>/
     ```
   - If `--delete` would remove local skills, ask before applying deletes.
   - Apply non-conflicting additions/updates:
     ```bash
     rsync -avc --exclude '.git/' <repo-skills-dir>/ <local-skills-dir>/
     ```

5. **Merge local skills → repo**
   - Copy local skill changes back to the repo skills directory.
   - Use dry-run first:
     ```bash
     rsync -avnc --delete --exclude '.git/' <local-skills-dir>/ <repo-skills-dir>/
     ```
   - If `--delete` would remove repo files, ask before applying deletes.
   - Apply non-conflicting additions/updates:
     ```bash
     rsync -avc --exclude '.git/' <local-skills-dir>/ <repo-skills-dir>/
     ```

6. **Commit and push repo changes**
   - Inspect final diff:
     ```bash
     git -C <repo-dir> status --porcelain=v2 -b --ignore-submodules=all
     git -C <repo-dir> diff --stat
     git -C <repo-dir> diff -- <repo-skills-dir>
     ```
   - If there are repo changes, commit them with a clear message:
     ```bash
     git -C <repo-dir> add <repo-skills-dir>
     git -C <repo-dir> commit -m "Sync Pi skills"
     ```
   - Push:
     ```bash
     git -C <repo-dir> push
     ```

7. **Final response**
   - Summarize:
     - repo used
     - local skills directory used
     - skills pulled from GitHub
     - skills pushed to GitHub
     - commit SHA, if a commit was created
     - any skipped conflicts or questions

## Conflict handling

Stop and ask the user when:
- both local and repo versions changed the same skill differently
- a skill exists locally but was deleted remotely
- a skill exists remotely but was deleted locally
- repo layout is ambiguous
- Git reports merge conflicts
- push is rejected after a normal pull/merge attempt

When asking, show the relevant file paths and a short diff/summary of the conflict.
