# Pi Skills

## Install pi

Install the Pi coding agent CLI with npm:

```bash
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
```

Or use the installer:

```bash
curl -fsSL https://pi.dev/install.sh | sh
```

Start pi with an API key:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
pi
```

Or start pi and sign in with an existing subscription:

```bash
pi
/login  # Then select provider
```

Install these skills locally:

```bash
git clone git@github.com:taillades/pi-skills.git ~/.pi/skills-repo
rsync -avc --exclude '.git/' ~/.pi/skills-repo/ ~/.pi/agent/skills/
```

Personal skill definitions for the [Pi coding agent](https://pi.dev). Each directory in this repository is a Pi skill and contains a `SKILL.md` file with the trigger description, workflow, and tool constraints for that task.

## Skills

| Skill | Purpose |
| --- | --- |
| `address-pr-comments` | Inspect comments on the current GitHub pull request, apply clear requested changes, and summarize before commit/push. |
| `dataset-analysis` | Safely inspect arbitrary datasets and produce a concise PDF report with inventory, sanity checks, and plots. |
| `fix-tests` | Read CI logs, identify failing tests, make straightforward fixes, validate, commit, and push when safe. |
| `move-local-mods-to-pr-worktree` | Move selected local modifications into a fresh worktree, commit them, and create a PR. |
| `pr` | Create or update a pull request. |
| `rebase-main-into-branch` | Rebase the current branch onto latest `main` safely, ask on edge cases/conflicts, validate, and force-push with lease when clean. |
| `sync-pi-skills` | Bidirectionally sync local Pi skills with this GitHub-backed repository. |

## Layout

```text
<skill-name>/
  SKILL.md
```

This repo mirrors the local Pi skills directory:

```text
~/.pi/agent/skills
```

## Syncing

Use the `sync-pi-skills` skill from Pi, or manually copy changes between this repository and the local skills directory.

Typical manual flow:

```bash
# Pull remote skill updates
git pull --no-rebase

# Copy repo skills into Pi's local skills directory
rsync -avc --exclude '.git/' ./ ~/.pi/agent/skills/

# Copy local edits back into the repo
rsync -avc --exclude '.git/' ~/.pi/agent/skills/ ./

# Commit and push
git status
git add .
git commit -m "Sync Pi skills"
git push
```

Use a dry run before applying deletes or broad syncs:

```bash
rsync -avnc --delete --exclude '.git/' ./ ~/.pi/agent/skills/
```

## Adding a skill

1. Create a directory named after the skill.
2. Add `SKILL.md` with frontmatter and task-specific instructions.
3. Sync it to `~/.pi/agent/skills`.
4. Commit and push the new directory.

Example:

```text
my-new-skill/
  SKILL.md
```

## Safety notes

- Do not auto-resolve semantic conflicts between local and repo versions.
- Ask before deleting skills, overwriting conflicting changes, resetting history, or force-pushing.
- Keep skills focused: one task, clear triggers, explicit workflow, and concrete stopping conditions.
