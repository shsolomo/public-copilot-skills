# Copilot CLI Skills

A starter kit for building and sharing [GitHub Copilot CLI](https://docs.github.com/en/copilot/github-copilot-in-the-cli) skills. Use this repo as a foundation to create your own private skills collection, share skills with your team, and publish skills publicly.

## Quick Start

### 1. Set Up Your Private Skills Repo

Your private skills repo is where all your personal skills live. Copilot automatically scans `~/.copilot/skills/` for skills on every message.

```powershell
# Create your private skills directory and initialize it as a git repo
mkdir ~/.copilot/skills
cd ~/.copilot/skills
git init
```

### 2. Add Your First Skill

Copy any skill from this repo into your private skills directory to try it out:

```powershell
# Clone this repo somewhere temporary
git clone https://github.com/shsolomo/public-copilot-skills.git ~/temp-skills

# Copy a skill you want
Copy-Item -Recurse ~/temp-skills/daily-report ~/.copilot/skills/daily-report

# Clean up
Remove-Item -Recurse ~/temp-skills
```

Now just ask Copilot naturally — "run my daily report" — and the skill activates automatically.

### 3. Create Your Own Skills

Each skill is a folder with a `SKILL.md` file inside:

```
~/.copilot/skills/
├── daily-report/
│   └── SKILL.md
├── my-custom-skill/
│   └── SKILL.md
└── ...
```

The `SKILL.md` has two parts — **frontmatter** that tells Copilot when to use the skill, and a **body** with instructions:

```markdown
---
name: my-custom-skill
description: A short description with keywords. Copilot matches your messages against this text to decide when to load the skill.
---

# My Custom Skill

## Instructions

1. Do this first
2. Then do this
3. Finally do this

## Constraints

- Don't do this
- Never do that
```

**Key things to know:**

- `name` must match the folder name exactly
- `description` is how Copilot discovers the skill — pack it with keywords your team would naturally say
- Write instructions as commands ("Create the file", "Run the query"), not explanations
- One skill should do one thing well — split broad skills into focused ones
- Keep `description` under 200 characters

## How Skills Work

When you send a message to Copilot CLI, it reads the `description` from every `SKILL.md` in your skills directory. If your message matches a skill's description, Copilot loads that skill and follows its instructions.

This means:
- **You don't need special commands.** Just talk naturally — "prep me for my 2pm meeting" — and the matching skill activates.
- **The description matters.** If a skill isn't activating, check that your phrasing overlaps with the keywords in the description.
- **Skills are additive.** They don't replace Copilot's built-in abilities — they teach it new workflows on top.

## Where Skills Live

Copilot looks for skills in two places:

| Location | Scope | Best For |
|----------|-------|----------|
| `~/.copilot/skills/` | **User-level** — available everywhere on your machine | Personal skills, productivity tools |
| `.github/skills/` (in a repo) | **Repo-level** — available to everyone in that repo | Team skills, project-specific workflows |

## Team Collaboration with Symlinks

The most powerful setup combines both locations. Your team puts shared skills in your project repo's `.github/skills/` directory, and individual team members can symlink the ones they want into their global skills directory.

### Why This Works

- **Shared skills stay in the repo** — everyone gets them on `git pull`, changes go through PR review
- **Personal skills stay private** — your `~/.copilot/skills/` repo is yours alone
- **Symlinks bridge the gap** — make any repo skill available globally without copying

### Example: Make a Repo Skill Available Everywhere

Your team has an `ado` skill in the project repo. You want it available in every project, not just when you're working in that repo:

```powershell
# Symlink the repo skill into your global skills directory
New-Item -ItemType SymbolicLink `
  -Path "$HOME\.copilot\skills\ado" `
  -Target "C:\repos\my-project\.github\skills\ado"
```

Now the `ado` skill works everywhere, and updates from `git pull` are reflected automatically.

### Example: Share a Personal Skill into a Repo

You built a skill that would help the team. Share it into the repo:

```powershell
# Symlink your global skill into the repo
New-Item -ItemType SymbolicLink `
  -Path "C:\repos\my-project\.github\skills\my-skill" `
  -Target "$HOME\.copilot\skills\my-skill"
```

> ⚠️ **Note:** Symlinks into a repo point to your local path — other team members won't have the same path. For permanent team sharing, copy the skill folder into the repo instead of symlinking.

### The `skill-link` Skill

This repo includes a **[skill-link](skill-link/)** skill that automates all of this. Once installed, just ask Copilot:

```
> link the ado skill to global
> make daily-report available in this repo
> list linked skills
> unlink wiki-search from global
```

### Recommended Team Setup

```
Your project repo (.github/skills/):     Your machine (~/.copilot/skills/):
├── ado/              ← team skill        ├── ado/          ← symlink to repo
├── wiki-search/      ← team skill        ├── wiki-search/  ← symlink to repo
└── deploy-helper/    ← team skill        ├── daily-report/ ← personal skill
                                          ├── my-notes/     ← personal skill
                                          └── skill-link/   ← from this repo
```

Team members `git pull` to get shared skills. Each person symlinks the ones they want globally and keeps their own private skills separate.

## Organizing Skills with Prefixes

Copilot's skills directory is **flat** — no nested folders. Use prefixes to group related skills:

```
notes-capture/       ← notes group
notes-discovery/     ← notes group
notes-lifecycle/     ← notes group
ado/                 ← standalone
daily-report/        ← standalone
```

The prefix is the domain, the suffix is the capability.

## Publishing Your Own Skills

Once you've built skills you want to share publicly, you can set up a publishing workflow similar to this repo:

1. **[public-repo-setup](public-repo-setup/)** — Creates a public GitHub repo linked to your private skills repo using orphan branches and worktrees. Includes a pre-push safety hook that checks for sensitive file names.
2. **[publish-skill](publish-skill/)** — Publishes individual skills from your private repo to the public one.

This lets you selectively share skills without exposing your entire private skills collection.

## Included Skills

| Skill | Description |
|-------|-------------|
| **[daily-report](daily-report/)** | Morning briefing from Teams, calendar, email, and ADO work items |
| **[meeting-prep](meeting-prep/)** | Gather context for upcoming meetings from Teams, email, ADO, wiki, and notes |
| **[skill-link](skill-link/)** | Symlink skills between global and repo directories for team sharing |
| **[public-repo-setup](public-repo-setup/)** | Set up selective public publishing for a private repo |
| **[publish-skill](publish-skill/)** | Publish skills from private to public repo |

## Contributing

Built a skill that would help the team? Copy it into the repo and open a PR. If you find a bug or have an improvement, reach out or submit a fix.
