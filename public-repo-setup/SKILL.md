---
name: public-repo-setup
description: Set up and manage selective public publishing for any private git repo using orphan branches and worktrees. Use when asked to "set up public publishing", "create public repo", "add safety hook", or manage private/public repo pairs.
---

# Public Repo Setup

## Purpose

Automates the full setup and ongoing management of selective public publishing for private git repos. Uses the orphan branch + worktree pattern so published content shares zero history with the private repo.

## Trigger Phrases

- "set up public publishing for {repo}"
- "create a public repo for {repo}"
- "add a safety hook"
- "show my public repos"
- "check public repo status"
- "remove public publishing"

## Concepts

- **Private repo**: The user's existing private git repo with all their content.
- **Public repo**: A separate public GitHub repo that receives cherry-picked files.
- **Orphan branch**: A branch with no shared history — the bridge between private and public.
- **Worktree**: A second checkout directory linked to the same `.git` database, checked out on the orphan branch.
- **Pre-push hook**: A safety script that prevents accidentally pushing private branches to the public remote.

## Commands

### setup {repo-path}

Full end-to-end setup of public publishing for a private repo.

**Steps:**

1. **Validate the repo** — confirm `{repo-path}` is a git repo with at least one commit. If not, stop and explain.

2. **Gather configuration** — ask the user for:
   - **Public repo name** on GitHub (e.g., `public-notes`). Default: `public-{private-repo-name}`.
   - **Public repo description** (optional).
   - **Worktree location** — where to place the public worktree. Default: `{repo-parent-dir}/{public-repo-name}`.
   - **Remote name** for the public repo. Default: `public`.
   - **Orphan branch name**. Default: `public-branch`.

3. **Create the public GitHub repo:**
   ```powershell
   gh repo create {username}/{public-repo-name} --public --description "{description}"
   ```
   If `gh` is not available or not authenticated, provide the manual GitHub steps instead.

4. **Create the orphan branch** in the private repo:
   ```powershell
   cd {repo-path}
   git switch --orphan {orphan-branch}
   git commit --allow-empty -m "Init public branch"
   git switch {original-branch}
   ```

5. **Add the public remote:**
   ```powershell
   git remote add {remote-name} https://github.com/{username}/{public-repo-name}.git
   ```

6. **Create the worktree:**
   ```powershell
   git worktree add {worktree-path} {orphan-branch}
   ```

7. **Install the pre-push safety hook** — see the `add-hook` command below. Always install the hook during setup.

8. **Push the initial empty commit:**
   ```powershell
   cd {worktree-path}
   git push {remote-name} {orphan-branch}:main
   cd {repo-path}
   ```

9. **Confirm success** — summarize what was created:
   - Public repo URL
   - Worktree path
   - Remote name
   - How to publish files: `cd {worktree-path} && git checkout {main-branch} -- path/to/file`

10. **Offer to save config** — ask if the user wants the configuration saved to their global copilot instructions (`~/.copilot/copilot-instructions.md`).

### add-hook {repo-path}

Install or update the pre-push safety hook on a repo.

**Steps:**

1. **Locate the git dir** — resolve `{repo-path}/.git` (or the linked gitdir if it's a worktree).

2. **Check for existing hook** — if `.git/hooks/pre-push` exists, show it and ask whether to replace or append.

3. **Write the hook:**
   ```bash
   #!/bin/bash
   remote="$1"

   if [[ "$remote" != "{remote-name}" ]]; then
     exit 0
   fi

   echo "⚠️  Pushing to PUBLIC remote"
   echo "Files in this push:"
   echo "---"
   git diff --name-only HEAD~1 HEAD
   echo "---"
   read -p "Are you sure? (y/n) " -n 1 -r
   echo
   if [[ ! $REPLY =~ ^[Yy]$ ]]; then
     echo "Push aborted."
     exit 1
   fi

   PRIVATE_PATTERNS=("secret" "private" "draft" "journal" "internal")
   FILES=$(git diff --name-only HEAD~1 HEAD)

   for pattern in "${PRIVATE_PATTERNS[@]}"; do
     if echo "$FILES" | grep -qi "$pattern"; then
       echo "🚫 Blocked: found file matching '$pattern'"
       exit 1
     fi
   done
   ```

4. **Ask if the user wants to customize** the blocked patterns list.

5. **Confirm** the hook is installed.

### status {repo-path?}

Show the current state of public publishing for a repo (or all configured repos).

**Steps:**

1. If `{repo-path}` is provided, inspect that repo. Otherwise, check all repos listed in the user's global copilot instructions under any "Public Publishing" config sections.

2. For each repo, report:
   - Private repo path and branch
   - Public remote URL
   - Worktree path and whether it exists
   - Orphan branch name
   - Whether the pre-push hook is installed
   - Number of files on the public branch vs. private branch
   - Last push date to the public remote (from `git log {orphan-branch} -1 --format=%ci`)

3. Flag any issues:
   - Worktree missing or corrupt
   - Public remote unreachable
   - Pre-push hook missing
   - Unpushed commits on the orphan branch

### teardown {repo-path}

Remove public publishing setup from a repo.

**Steps:**

1. **Confirm with the user** — this is destructive. List what will be removed.

2. **Remove the worktree:**
   ```powershell
   git worktree remove {worktree-path}
   ```

3. **Delete the orphan branch:**
   ```powershell
   git branch -D {orphan-branch}
   ```

4. **Remove the public remote:**
   ```powershell
   git remote remove {remote-name}
   ```

5. **Optionally delete the hook** — ask if the user wants to remove the pre-push hook (they may have other hooks in the same file).

6. **Optionally delete the public GitHub repo** — ask, and if yes:
   ```powershell
   gh repo delete {username}/{public-repo-name} --yes
   ```

7. **Remove config** from global copilot instructions if it was saved there.

8. **Confirm** what was cleaned up.

## Constraints

- **Never push a non-orphan branch to the public remote.** This is the cardinal rule.
- **Always install the pre-push hook during setup.** It's not optional.
- **Never delete or modify the private repo's main branch** — only create/manage the orphan branch.
- **Always ask for confirmation** before destructive operations (teardown, deleting repos).
- **Use `git switch --orphan`** not `git checkout --orphan` — it starts with a clean working tree automatically.
