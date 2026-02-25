---
name: daily-report
description: Generates a comprehensive daily report covering Teams chats, calendar, email, and ADO work items. Run this each morning to get a scannable summary of everything you need to know to start your day.
---

# Daily Report Skill

## Purpose

Generate a terse, scannable daily report every morning that covers:
- Teams chat activity (last 24h)
- Today's calendar/meetings
- Urgent and VIP emails
- Azure DevOps work item status

## Required Configuration

This skill reads user-specific values from the user's global copilot instructions (`~/.copilot/copilot-instructions.md`). The following sections must be present:

### Azure DevOps Defaults
- **Organization** — ADO org URL (e.g., `https://dev.azure.com/myorg`)
- **Project** — ADO project name
- **Area Path** — Area path for work item queries (with `UNDER` semantics)

### Teams Channels
A list of Teams group chats to monitor, each with:
- **Chat Name** — display name of the chat
- **Thread ID** — the `19:...@thread.v2` identifier for reliable querying

### VIP Contacts
A list of people (names, aliases, distribution lists) whose communications should be surfaced with higher priority.

> **Fallback:** If any required configuration section is missing from the user's global copilot instructions (`~/.copilot/copilot-instructions.md`), ask the user for the values and offer to add them.

## How to Execute

Execute in 3 steps. ADO is fetched FIRST so that work item titles and IDs can be passed as context into the Teams and email queries for better correlation.

### Step 1: Fetch ADO Context (run both in parallel, BEFORE other queries)

These queries establish what the user is actively working on. Their results (work item titles and IDs) will be injected into the Teams and email queries in Step 2.

**IMPORTANT:** Always use `-o table` for `az boards query` output. Do NOT use `-o json` — the CLI can produce malformed JSON when work item titles contain newlines or special characters.

> **Note:** Substitute `{ADO_ORG}`, `{ADO_PROJECT}`, and `{ADO_AREA_PATH}` with the values from `## Azure DevOps Defaults` in global copilot instructions.

**Query 1a — ADO: Active + recently completed work items**
```powershell
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State], [System.WorkItemType], [System.CreatedDate], [Microsoft.VSTS.Common.Priority], [Microsoft.VSTS.Common.StateChangeDate] FROM WorkItems WHERE [System.AssignedTo] = @Me AND ([System.State] NOT IN ('Closed', 'Removed', 'Done') OR ([System.State] IN ('Closed', 'Done') AND [Microsoft.VSTS.Common.ClosedDate] >= @Today - 30)) AND [System.AreaPath] UNDER '{ADO_AREA_PATH}' ORDER BY [Microsoft.VSTS.Common.Priority] ASC" --org "{ADO_ORG}" -p "{ADO_PROJECT}" -o table
```

**Query 1b — ADO: State changes in last 24h**
```powershell
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State], [System.ChangedDate], [System.ChangedBy] FROM WorkItems WHERE [System.AreaPath] UNDER '{ADO_AREA_PATH}' AND [System.ChangedDate] >= @Today - 1 ORDER BY [System.ChangedDate] DESC" --org "{ADO_ORG}" -p "{ADO_PROJECT}" -o table
```

After running these, extract the list of work item titles and IDs. Build a comma-separated summary string of your active work items, e.g.:
`"#ID1 Title1, #ID2 Title2, #ID3 Title3, ..."`

This becomes `{WORK_ITEMS_CONTEXT}` used in Step 2.

### Step 2: Gather M365 Data (run ALL in parallel, using ADO context)

**Query 2a — Teams: Monitored chat rollups**

For each chat listed in `## Teams Channels` in global copilot instructions, run a query using this template:
```
workiq ask -q "Summarize the messages from the last 24 hours in the Teams chat thread {THREAD_ID} (this is the '{CHAT_NAME}' group chat). For context, my current work items are: {WORK_ITEMS_CONTEXT}. Organize by: 1) Any mentions of me or my work items, 2) Open questions needing input, 3) Decisions made, 4) Action items assigned, 5) Announcements or knowledge sharing. Correlate any discussion topics to my work items where relevant. Include links to messages."
```

Run all chat queries in parallel.

**Query 2c — Teams: Direct mentions across all chats**
```
workiq ask -q "Find any Teams messages from the last 24 hours where I was directly mentioned or tagged, across all chats and channels. List each with the chat name, who mentioned me, and what they said. Note if any relate to these work items: {WORK_ITEMS_CONTEXT}."
```

**Query 2d — Calendar: Today's meetings**
```
workiq ask -q "What meetings do I have today? For each meeting, list the time, title, attendees, and any agenda or related context. Flag any scheduling conflicts. Note if any meetings relate to these work items: {WORK_ITEMS_CONTEXT}."
```

**Query 2e — Calendar: Yesterday's meeting recaps**
```
workiq ask -q "Summarize decisions and action items from meetings I attended yesterday. Use transcripts if available. Note if any relate to these work items: {WORK_ITEMS_CONTEXT}."
```

**Query 2f — Email: Urgent/VIP emails**
```
workiq ask -q "Find emails from the last 24 hours that are either marked high importance OR are from anyone listed in ## VIP Contacts in the user's global copilot instructions. Summarize each briefly and flag any that need my reply. Note if any relate to these work items: {WORK_ITEMS_CONTEXT}."
```

**Query 2g — Email: Awaiting my reply**
```
workiq ask -q "Find email threads from the last 48 hours where someone asked me a direct question or is waiting for my response. List sender, subject, and what they're waiting on."
```

### Step 3: Assemble the Report

Combine all results into this format. Use bullet points. Be terse. Skip empty sections with "— Nothing to report".

```
═══════════════════════════════════════════
  🌅 DAILY REPORT — {today's date}
═══════════════════════════════════════════

🔔 MENTIONS & DIRECT REQUESTS
  {mentions from all chats, tagged items}

❓ OPEN QUESTIONS
  {unresolved threads needing your input}

✅ DECISIONS MADE (last 24h)
  {decisions from chats and meetings}

📌 ACTION ITEMS
  {commitments by/to you from chats and meetings}

📅 TODAY'S MEETINGS
  {time | title | key attendees | prep notes}

📝 YESTERDAY'S MEETING RECAPS
  {decisions and action items from yesterday}

🚨 URGENT / VIP EMAILS
  {high-priority emails and VIP sender emails}

📬 AWAITING YOUR REPLY
  {email threads waiting on you}

📋 ACTIVE WORK ITEMS (ADO)
  {#ID | Type | Title | State | Priority | Age in days}

✅ RECENTLY COMPLETED (last 30 days)
  {#ID | Type | Title | Closed date}

🔥 AGING ITEMS (>14 days)
  {work items activated >14 days ago still open}

🚦 ADO STATE CHANGES (last 24h)
  {items that changed state, who changed them}

💡 KNOWLEDGE DROPS
  {useful tips, links, learnings shared in chats}

🎯 FOCUS RECOMMENDATIONS
  {top 3 suggested priorities based on urgency, age, blockers, and meetings}

═══════════════════════════════════════════
```

## VIP Email List

Uses the VIP contacts list from `## VIP Contacts` in global copilot instructions (`~/.copilot/copilot-instructions.md`).

## ADO Configuration

Uses the ADO defaults from global copilot instructions (`~/.copilot/copilot-instructions.md`). Additional report-specific settings:
- Aging threshold: 14 days (flag as 🔥)
- Stale threshold: 30 days (flag as 🚨)
- Calculate age from `System.CreatedDate` or `Microsoft.VSTS.Common.ActivatedDate`

## Formatting Rules

- **Terse and scannable** — bullet points only, no prose paragraphs
- Include Teams message links where available
- For ADO items, format as: `#ID | Type | Title | State | P{priority} | {age}d`
- For meetings, format as: `HH:MM - HH:MM | Title | Key attendees`
- For emails, format as: `From: {sender} | Subject: {subject} | {action needed}`
- Empty sections: show "— Nothing to report"
- Focus recommendations: pick top 3 items based on urgency, blockers, upcoming meetings, and aging items

## Constraints

- Do not fabricate data — only report what comes from WorkIQ and ADO queries
- If a query fails, note the failure in the report and continue with other sections
- Do not send emails or modify work items — this is read-only
- Always include the date in the report header
