# Copilot CLI Skills

A collection of reusable skills for [GitHub Copilot CLI](https://docs.github.com/en/copilot/github-copilot-in-the-cli). These extend what Copilot can do for you by teaching it new capabilities — like generating a daily report from your email/Teams/calendar, or prepping you for a meeting.

## What Are Skills?

Skills are instructions (written in Markdown) that tell Copilot how to perform a specific task. When you ask Copilot something that matches a skill's description, it automatically loads the skill and follows the instructions inside.

Think of them as **recipes** — each one teaches Copilot a new workflow it wouldn't know on its own.

Each skill lives in its own folder and contains a `SKILL.md` file:

```
daily-report/
└── SKILL.md      ← The instructions Copilot follows
```

The `SKILL.md` has two parts:

1. **Frontmatter** (YAML at the top) — tells Copilot the skill's name and when to use it
2. **Body** (Markdown) — step-by-step instructions Copilot follows when the skill is activated

## Available Skills

| Skill | What It Does |
|-------|-------------|
| **[daily-report](daily-report/)** | Generates a morning briefing from your Teams chats, calendar, email, and ADO work items. Ask for your "daily report" and get a scannable summary of everything you need to know. |
| **[meeting-prep](meeting-prep/)** | Gathers context for an upcoming meeting — pulls related Teams messages, emails, ADO items, wiki pages, and notes so you walk in prepared. |
| **[public-repo-setup](public-repo-setup/)** | Sets up selective public publishing for a private repo using git orphan branches and worktrees. Includes a safety hook to prevent accidental leaks. |
| **[publish-skill](publish-skill/)** | Publishes a skill from your private skills folder to this public repo using the orphan branch workflow. |

## How to Install

### Option 1: Clone the Whole Repo (Recommended)

Clone this repo into your Copilot skills directory:

```powershell
git clone https://github.com/shsolomo/public-copilot-skills.git ~/.copilot/skills
```

> **Note:** If you already have a `~/.copilot/skills` folder with your own skills, clone this somewhere else and copy the skill folders you want into your existing directory.

### Option 2: Copy Individual Skills

If you only want specific skills, copy the folder into your skills directory:

```powershell
# Example: just grab daily-report
Copy-Item -Recurse ./public-copilot-skills/daily-report ~/.copilot/skills/daily-report
```

### Where Skills Live

Copilot looks for skills in these locations:

| Location | Scope |
|----------|-------|
| `~/.copilot/skills/` | **User-level** — available in every project on your machine |
| `.github/skills/` (in a repo) | **Repo-level** — available to everyone working in that repo |

For personal productivity skills (like daily-report), user-level is usually the right choice.

## How to Use

Once installed, just ask Copilot naturally. You don't need to remember special commands — the skill activates when your request matches its description.

**Examples:**

```
> run my daily report

> prep me for my 2pm meeting with the infrastructure team

> set up public publishing for this repo
```

Copilot will load the matching skill and follow its instructions automatically.

## How Skills Work (Under the Hood)

When you send a message to Copilot CLI:

1. Copilot reads the `description` field in every `SKILL.md` across your skills directory
2. If your message matches a skill's description, that skill gets loaded into context
3. Copilot follows the instructions in the skill's body to complete your request

This means the **description is important** — it's how Copilot decides which skill to use. If a skill isn't activating, check that your phrasing overlaps with keywords in the description.

## Creating Your Own Skills

Want to make a new skill? Create a folder with a `SKILL.md`:

```
my-new-skill/
└── SKILL.md
```

Here's a minimal template:

```markdown
---
name: my-new-skill
description: One or two sentences describing what this skill does and when to use it. Include trigger keywords.
---

# My New Skill

## Instructions

1. First step Copilot should take
2. Second step
3. Third step

## Constraints

- Things Copilot should NOT do
```

**Tips:**

- The `name` must match the folder name exactly
- Keep the `description` under 200 characters and keyword-rich
- Write instructions as direct commands ("Run this", "Create that") not explanations
- Include concrete examples when possible
- One skill = one task. Split broad skills into focused ones

## Organizing Skills with Prefixes

Since Copilot's skills directory is **flat** (no nested folders), use prefixes to group related skills:

```
notes-capture/       ← notes group
notes-discovery/     ← notes group
notes-lifecycle/     ← notes group
ado/                 ← standalone
daily-report/        ← standalone
```

The prefix is the domain, the suffix is the capability.

## Contributing

Found a bug or have an improvement? Open a PR or reach out. If you build a skill that would be useful for the team, we can add it here.
