---
name: meeting-prep
description: Prepare for upcoming meetings by gathering related context from Teams chats, emails, ADO work items, wiki pages, and personal notes. Use this skill when asked to prep for a meeting.
---

# Meeting Prep Skill

## Purpose

Given a meeting title or calendar event, gather all relevant context so you're prepared. Pulls from Teams, email, ADO, wiki, and personal notes to build a concise prep brief.

## Required Configuration

This skill requires the following from the user's global copilot instructions (`~/.copilot/copilot-instructions.md`):
- `## Azure DevOps Defaults` — org, project, area path for work item queries
- `## Wiki Configuration` — wiki name for documentation lookup
- `## Notes Repo` — path to personal notes vault
- `## VIP Contacts` — for identifying important attendees
- `## Meeting Keywords` (optional) — recurring meeting keyword strategies for better context searching

If any required configuration is missing, ask the user for the values and offer to add them to their `~/.copilot/copilot-instructions.md`.

## How to Execute

### Step 1: Identify the Meeting

If the user specifies a meeting name, use it directly. Otherwise, query today's calendar:

```
workiq ask -q "What meetings do I have today? List the time, title, and attendees for each."
```

### Step 2: Gather Context (run in parallel)

For the target meeting, run all of these in parallel:

**Query A — Recent Teams threads about this topic**
```
workiq ask -q "Find Teams messages from the last 7 days that are related to '{meeting title}' or discuss topics likely to come up in a meeting called '{meeting title}'. Summarize the key discussion points and any unresolved questions."
```

**Query B — Related emails**
```
workiq ask -q "Find emails from the last 7 days related to '{meeting title}' or involving the meeting attendees: {attendee names}. Summarize key points and any open questions or requests."
```

**Query C — Prior meeting notes/recaps**
```
workiq ask -q "Find notes, transcripts, or recaps from previous meetings with a similar title to '{meeting title}' or involving the same attendees. Summarize key decisions, action items, and open issues from the most recent instance."
```

**Query D — Related ADO work items**
```powershell
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State], [System.AssignedTo] FROM WorkItems WHERE [System.AreaPath] UNDER '{ADO_AREA_PATH}' AND [System.State] NOT IN ('Closed', 'Removed', 'Done') AND [System.Title] CONTAINS '{relevant keyword}' ORDER BY [Microsoft.VSTS.Common.Priority] ASC" --org "{ADO_ORG}" -p "{ADO_PROJECT}" -o table
```

> Substitute `{ADO_ORG}`, `{ADO_PROJECT}`, and `{ADO_AREA_PATH}` from the user's `## Azure DevOps Defaults` in their global copilot instructions.

Extract relevant keywords from the meeting title to search work items.

**Query E — Wiki pages**
```powershell
az devops wiki page show --wiki "{WIKI_NAME}" --path "/" --org "{ADO_ORG}" -p "{ADO_PROJECT}" --recursion-level 5 --query "page.subPages[].path" -o tsv | Select-String -Pattern "{relevant keyword}"
```

> Substitute `{WIKI_NAME}`, `{ADO_ORG}`, and `{ADO_PROJECT}` from the user's global copilot instructions (`## Wiki Configuration` and `## Azure DevOps Defaults`).

If matching pages are found, fetch their content for reference.

**Query F — Personal notes**
```powershell
Get-ChildItem -Path "{NOTES_PATH}" -Recurse -Include "*.md" -Exclude "README.md","*.instructions.md" | Select-String -Pattern "{meeting topic or attendee name}" -Context 2,2
```

> Substitute `{NOTES_PATH}` from the user's `## Notes Repo` in their global copilot instructions.

Also check for a matching domain folder (e.g., `domains/triitus/`, `domains/hub-hour/`) and read recent notes from it.

### Step 3: Assemble the Prep Brief

```
═══════════════════════════════════════════
  📋 MEETING PREP — {meeting title}
  {time} | {attendees}
═══════════════════════════════════════════

📌 LAST MEETING RECAP
  • Key decisions from the prior meeting
  • Outstanding action items and their status

💬 RECENT DISCUSSION (Teams/Email)
  • Relevant threads and key points from the last 7 days
  • Unresolved questions that may come up

📋 RELATED WORK ITEMS
  #ID | Title | State | Assigned To
  (items relevant to this meeting's topics)

📖 WIKI REFERENCES
  • Relevant wiki pages with links

📝 YOUR NOTES
  • Recent personal notes on this topic
  • Decisions and next steps you've captured

❓ SUGGESTED TALKING POINTS
  • Open questions from recent threads
  • Action items that need status updates
  • Decisions that need to be made

═══════════════════════════════════════════
```

## Meeting-Specific Strategies

### Recurring Meetings

Check `## Meeting Keywords` in the user's global copilot instructions for a table of recurring meetings and their associated search keywords. If found, use those keywords for targeted searches. If not found, extract keywords from the meeting title and attendee names.

### One-off Meetings

For unfamiliar meetings:
1. Search by meeting title keywords
2. Search by attendee names
3. Check if any attendees are on the VIP list
4. Look for related calendar events in the past 30 days

## Constraints

- This is read-only — do not send emails, modify work items, or change calendar events
- Keep the prep brief terse and scannable — bullet points only
- If a data source returns nothing relevant, skip that section with "— No relevant context found"
- Include links where available (Teams messages, wiki pages, ADO work items)
- Prioritize recent context (last 7 days) over older material
