---
name: publish-skill
description: Publish a skill directory to the public copilot skills repo using the orphan branch worktree workflow. Use when asked to "publish skill", "push skill to public", or "share skill publicly".
---

# Publish Skill

## Purpose

Automates the workflow of cherry-picking a skill from the private skills repo into the public worktree, committing it, and pushing it to the public GitHub repo — all in one step.

## Required Configuration

This skill requires the following from the user's global copilot instructions (`~/.copilot/copilot-instructions.md`):

- `## Public Skills Publishing` — paths and remote configuration:
  - `private_skills_path`: Path to the private skills repo (e.g., `C:\Users\shsolomo\.copilot\skills`)
  - `public_worktree_path`: Path to the public worktree (e.g., `C:\Users\shsolomo\.copilot\public-skills`)
  - `public_remote_name`: Name of the git remote for the public repo (e.g., `public`)
  - `public_branch`: Name of the local orphan branch (e.g., `public-branch`)
  - `remote_branch`: Name of the branch on the remote (e.g., `main`)

If any required configuration is missing, ask the user for the values and offer to add them to their `~/.copilot/copilot-instructions.md`.

## Instructions

When the user asks to publish or push a skill to their public repo:

1. **Parse the skill name** from the user's request. The skill name is a directory under the private skills path.

2. **Validate the skill exists** — confirm the directory exists at `{private_skills_path}/{skill-name}/`.

3. **Check for uncommitted changes** in the private repo. If the skill has unstaged changes on `main`, warn the user: "The skill has uncommitted changes on main. Commit them first so the public version is up to date."

4. **Navigate to the public worktree**:
   ```powershell
   cd {public_worktree_path}
   ```

5. **Cherry-pick the skill from main**:
   ```powershell
   git checkout main -- {skill-name}/
   ```

6. **Review what changed** — run `git diff --staged --stat` and show the user a summary of files being published.

7. **Confirm with the user** — use `ask_user` to present the file list and ask for confirmation before pushing. Example: "These files will be pushed to the **public** repo: \n\n- skill-name/SKILL.md\n\nPush to public?" with choices `["Yes, push it", "No, cancel"]`. If the user cancels, unstage the changes (`git reset HEAD`) and stop.

8. **Commit the change**:
   ```powershell
   git add -A
   git commit -m "Publish {skill-name}"
   ```

9. **Push to the public remote**:
   ```powershell
   git push {public_remote_name} {public_branch}:{remote_branch}
   ```
   The pre-push hook runs automatically as a silent safety net — it blocks files matching sensitive patterns (secret, private, draft, journal, internal) but does not prompt interactively.

9. **Return to the private repo**:
   ```powershell
   cd {private_skills_path}
   ```

10. **Confirm success**: "Published `{skill-name}` to the public skills repo."

## Constraints

- **Never push `main` to the public remote.** Only push `{public_branch}`.
- **Never publish multiple skills without explicit user approval** for each one.
- **Never modify the skill content** during publishing — copy it exactly as-is from `main`.
- **Never skip the confirmation step** — always use `ask_user` to confirm before committing and pushing.
- **Never skip the pre-push hook** — let it run as a silent safety net for pattern checks.
- If the skill directory doesn't exist, list available skills and ask the user to pick one.

## Examples

**User says:** "push notes to my public skills repo"

**Agent runs:**
```powershell
cd C:\Users\shsolomo\.copilot\public-skills
git checkout main -- notes/
git add -A
git commit -m "Publish notes"
git push public public-branch:main
```

**Agent responds:** "Published `notes` to the public skills repo. The pre-push hook confirmed the push."

---

**User says:** "publish ado and teams skills"

**Agent responds:** "I'll publish them one at a time. Starting with `ado`..."
Then runs the workflow for `ado`, confirms, then repeats for `teams`.
