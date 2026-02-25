---
name: skill-link
description: Symlink skills between global (~/.copilot/skills/) and repo (.github/skills/) directories. Use when asked to "link skill", "symlink skill", "make skill global", or "add skill to repo".
---

# Skill Link

## Purpose

Create symlinks between repo-level skills (`.github/skills/`) and user-level skills (`~/.copilot/skills/`) so a skill defined in one location is available in the other — without duplicating files.

## Instructions

### Determine Direction

When the user asks to link a skill, figure out the direction:

- **"make it global"**, **"use everywhere"**, **"link to global"** → repo skill → global (symlink in `~/.copilot/skills/` pointing to repo)
- **"add to repo"**, **"share with team"**, **"link to repo"** → global skill → repo (symlink in `.github/skills/` pointing to global)

If the direction is ambiguous, ask.

### Link a Repo Skill → Global

Make a repo-level skill available everywhere on this machine.

1. Identify the repo's `.github/skills/` directory (use the current git repo root).
2. Confirm the skill folder exists in `.github/skills/{skill-name}/`.
3. Check that `~/.copilot/skills/{skill-name}` does not already exist.
4. Create the symlink:

```powershell
New-Item -ItemType SymbolicLink -Path "$HOME\.copilot\skills\{skill-name}" -Target "{repo-root}\.github\skills\{skill-name}"
```

5. Verify the link:

```powershell
Get-Item "$HOME\.copilot\skills\{skill-name}" | Select-Object Name, LinkTarget
```

6. Confirm: "Linked `{skill-name}` → now available globally. Updates in the repo will be reflected automatically."

### Link a Global Skill → Repo

Share a user-level skill into a repo's `.github/skills/` directory.

1. Confirm the skill folder exists in `~/.copilot/skills/{skill-name}/`.
2. Identify the repo root (use `git rev-parse --show-toplevel` or the current working directory).
3. Create `.github/skills/` in the repo if it doesn't exist.
4. Check that `.github/skills/{skill-name}` does not already exist.
5. Create the symlink:

```powershell
New-Item -ItemType SymbolicLink -Path "{repo-root}\.github\skills\{skill-name}" -Target "$HOME\.copilot\skills\{skill-name}"
```

6. Verify the link:

```powershell
Get-Item "{repo-root}\.github\skills\{skill-name}" | Select-Object Name, LinkTarget
```

7. Warn: "Symlinks may not resolve for other team members since the target is your local path. If you want the skill available for the whole team, consider copying the skill folder instead of symlinking."

### List Linked Skills

When the user asks to see existing links ("list linked skills", "show symlinks"):

```powershell
# Global skills that are symlinks
Get-ChildItem -Path "$HOME\.copilot\skills" -Directory | Where-Object { $_.Attributes -match 'ReparsePoint' } | ForEach-Object {
    [PSCustomObject]@{ Name = $_.Name; Direction = "repo → global"; Target = $_.LinkTarget }
} | Format-Table -AutoSize

# Repo skills that are symlinks (if in a git repo)
$repoRoot = git rev-parse --show-toplevel 2>$null
if ($repoRoot -and (Test-Path "$repoRoot\.github\skills")) {
    Get-ChildItem -Path "$repoRoot\.github\skills" -Directory | Where-Object { $_.Attributes -match 'ReparsePoint' } | ForEach-Object {
        [PSCustomObject]@{ Name = $_.Name; Direction = "global → repo"; Target = $_.LinkTarget }
    } | Format-Table -AutoSize
}
```

### Unlink a Skill

When the user asks to remove a link ("unlink skill", "remove symlink"):

1. Identify which link to remove (ask if ambiguous).
2. Verify it's actually a symlink (not a real directory):

```powershell
$item = Get-Item "{path-to-skill}"
if ($item.Attributes -match 'ReparsePoint') {
    Remove-Item "{path-to-skill}" -Force
    Write-Output "Unlinked {skill-name}"
} else {
    Write-Output "⚠️ {skill-name} is not a symlink — it's a real directory. Not removing."
}
```

3. Confirm removal. The original skill folder is untouched.

## Constraints

- Never delete or modify the original skill directory — only create/remove symlinks
- Always verify a path is a symlink before removing it
- If a skill with the same name already exists at the target, do NOT overwrite — inform the user
- Warn about portability when symlinking global → repo (other team members won't have the same local path)
- On Windows, creating symlinks may require Developer Mode or elevated permissions — if `New-Item -ItemType SymbolicLink` fails, suggest enabling Developer Mode in Windows Settings

## Examples

```
> link the ado skill to global
# Creates symlink: ~/.copilot/skills/ado → {repo}/.github/skills/ado

> make daily-report available in this repo
# Creates symlink: {repo}/.github/skills/daily-report → ~/.copilot/skills/daily-report

> list linked skills
# Shows all symlinked skills and their targets

> unlink wiki-search from global
# Removes symlink at ~/.copilot/skills/wiki-search (original untouched)
```
