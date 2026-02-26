---
name: daily-report
description: Generates a comprehensive daily report covering Teams chats, calendar, email, and ADO work items. Run this each morning to get a scannable summary of everything you need to know to start your day.
---

# Daily Report Skill

## Purpose

Generate a terse, scannable daily report every morning. The report is **delta-focused by default**  it highlights what changed since yesterday, not a full inventory of everything. The goal is a report you can read in under 2 minutes.

## Required Configuration

This skill reads user-specific values from the user's global copilot instructions (`~/.copilot/copilot-instructions.md`). The following sections must be present:

### Azure DevOps Defaults
- **Organization** - ADO org URL (e.g., `https://dev.azure.com/myorg`)
- **Project** - ADO project name
- **Area Path** - Area path for work item queries (with `UNDER` semantics)

### Teams Channels
A list of Teams group chats to monitor, each with:
- **Chat Name** - display name of the chat
- **Thread ID** - the `19:...@thread.v2` identifier for reliable querying

### VIP Contacts
A list of people (names, aliases, distribution lists) whose communications should be surfaced with higher priority.

> **Fallback:** If any required configuration section is missing from the user's global copilot instructions (`~/.copilot/copilot-instructions.md`), ask the user for the values and offer to add them.

## How to Execute

Execute in 5 steps. ADO is fetched FIRST for correlation context. Steps 2 and 3 run in parallel. Step 4 assembles the report. Step 5 persists it.

---

### Step 1: Fetch ADO Context (run all in parallel, BEFORE other queries)

These queries establish what changed and what the user is actively working on.

**IMPORTANT:** Always use `-o table` for `az boards query` output. Do NOT use `-o json`  the CLI can produce malformed JSON when work item titles contain newlines or special characters.

> **Note:** Substitute `{ADO_ORG}`, `{ADO_PROJECT}`, and `{ADO_AREA_PATH}` with the values from `## Azure DevOps Defaults` in global copilot instructions.

**Query 1a - My active work items**
```powershell
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State], [System.WorkItemType], [System.CreatedDate], [Microsoft.VSTS.Common.Priority], [Microsoft.VSTS.Common.StateChangeDate] FROM WorkItems WHERE [System.AssignedTo] = @Me AND [System.State] NOT IN ('Closed', 'Removed', 'Done') AND [System.AreaPath] UNDER '{ADO_AREA_PATH}' ORDER BY [Microsoft.VSTS.Common.Priority] ASC" --org "{ADO_ORG}" -p "{ADO_PROJECT}" -o table
```

**Query 1b - State changes in last 24h (MY items only)**
```powershell
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State], [System.ChangedDate], [System.ChangedBy] FROM WorkItems WHERE [System.AssignedTo] = @Me AND [System.AreaPath] UNDER '{ADO_AREA_PATH}' AND [System.ChangedDate] >= @Today - 1 ORDER BY [System.ChangedDate] DESC" --org "{ADO_ORG}" -p "{ADO_PROJECT}" -o table
```

**Query 1c - Team state changes in last 24h (for awareness, NOT assigned to me)**
```powershell
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State], [System.ChangedDate], [System.ChangedBy] FROM WorkItems WHERE [System.AssignedTo] <> @Me AND [System.AreaPath] UNDER '{ADO_AREA_PATH}' AND [System.ChangedDate] >= @Today - 1 ORDER BY [System.ChangedDate] DESC" --org "{ADO_ORG}" -p "{ADO_PROJECT}" -o table
```

After running these, extract the list of active work item titles and IDs. Build a comma-separated summary string:
`"#ID1 Title1, #ID2 Title2, #ID3 Title3, ..."`
This becomes `{WORK_ITEMS_CONTEXT}` used in Step 2.

---

### Step 2: Run Work Item Health Analysis

Invoke the **work-item-health** skill. Pass it the ADO data from Step 1. Request the **condensed summary** format (not the full standalone report). The work-item-health skill handles:
- Sprint-aware aging analysis (2-week sprints)
- Type-specific health thresholds (Tasks <5d, Stories <1 sprint, Features multi-sprint)
- Velocity calculation (items closed per sprint by type)
- Refactoring suggestions (e.g., promote overdue Task  User Story)

Save the condensed output as `{HEALTH_SNAPSHOT}` for inclusion in Step 4.

> **If the work-item-health skill is not available**, fall back to basic aging analysis: flag items with `age > 14 days` as , `> 28 days` as , `> 42 days` as . Skip velocity and refactoring suggestions.

---

### Step 3: Gather M365 Data (run ALL in parallel, using ADO context)

**Query 3a - Teams: Persistent chat rollups**

For each chat listed under `### Persistent Chats` in `## Teams Channels` in global copilot instructions, run a query using the thread ID:
```
workiq ask -q "Summarize the messages from the last 24 hours in the Teams chat thread {THREAD_ID} (this is the '{CHAT_NAME}' group chat). For context, my current work items are: {WORK_ITEMS_CONTEXT}. Organize by: 1) Mentions of me or my work items, 2) Open questions needing input, 3) Decisions made, 4) Action items assigned. Correlate topics to my work items where relevant. Include links to messages."
```

> If no `## Teams Channels` section exists in global copilot instructions, skip this query and note in the report: " No Teams channels configured  add a `## Teams Channels` section to `~/.copilot/copilot-instructions.md`"

**Query 3a2 - Teams: Recurring meeting chat rollups**

For each meeting listed under `### Recurring Meetings` in `## Teams Channels`, search by name pattern instead of thread ID (thread IDs change when new meeting series are created):
```
workiq ask -q "Find the most recent Teams meeting chat matching the name pattern '{SEARCH_KEYWORDS}' from the last 24 hours. Summarize: 1) Key discussion points, 2) Decisions made, 3) Action items assigned (especially to me or by me), 4) Open questions. For context, my current work items are: {WORK_ITEMS_CONTEXT}. Correlate topics to my work items where relevant. Include links."
```

Run one query per recurring meeting. These can all run in parallel with the persistent chat queries. If no messages are found for a meeting pattern (i.e., it didn't occur yesterday), skip it silently.

**Query 3b - Teams: Direct mentions across all chats**
```
workiq ask -q "Find any Teams messages from the last 24 hours where I was directly mentioned or tagged, across all chats and channels. List each with the chat name, who mentioned me, and what they said. Note if any relate to these work items: {WORK_ITEMS_CONTEXT}."
```

**Query 3c - Calendar: Today's meetings**
```
workiq ask -q "What meetings do I have today? For each meeting, list the time, title, attendees, and any agenda or related context. Flag any scheduling conflicts. Note if any meetings relate to these work items: {WORK_ITEMS_CONTEXT}."
```

**Query 3d - Calendar: Yesterday's outcomes**
```
workiq ask -q "Summarize decisions and action items from meetings I attended yesterday. Use transcripts if available. Note if any relate to these work items: {WORK_ITEMS_CONTEXT}."
```

**Query 3e - Email: Urgent/VIP + awaiting reply (COMBINED)**
```
workiq ask -q "Search my email from the last 48 hours and find: 1) Emails marked high importance OR from {VIP_NAMES_LIST}. 2) Email threads where someone asked me a direct question or is waiting for my response. For each match, list: sender, subject, what action is needed, and any deadline. Note if any relate to these work items: {WORK_ITEMS_CONTEXT}."
```

> `{VIP_NAMES_LIST}` is the comma-separated list from `## VIP Contacts` in global copilot instructions.

---

### Step 3.5: Query Session Store (runs in parallel with Step 3)

Query the Copilot CLI session store to find what the user actually worked on yesterday:

```sql
SELECT s.id, s.branch, s.summary, s.created_at,
       (SELECT COUNT(*) FROM session_files sf WHERE sf.session_id = s.id AND sf.tool_name = 'edit') as files_edited
FROM sessions s
WHERE s.created_at >= date('now', '-1 day')
ORDER BY s.created_at DESC;
```

Also check for any commits or PRs created:
```sql
SELECT sr.ref_type, sr.ref_value, s.summary
FROM session_refs sr
JOIN sessions s ON sr.session_id = s.id
WHERE sr.created_at >= date('now', '-1 day');
```

Save results as `{YESTERDAY_SESSIONS}`.

---

### Step 4: Assemble the Report

Combine all results into the format below. **Delta-focused**: only show what changed, what needs attention, and what's coming up. Skip sections with nothing to report using " Nothing to report".

The report has two tiers:
1. ** RIGHT NOW** (top)  things requiring action today. Max ~10 lines.
2. ** FULL CONTEXT** (below)  supporting details for reference.

```

   DAILY REPORT  {today's date}


━  RIGHT NOW 
  {Top 3-5 items needing action TODAY, selected from:
   - Meetings in the next 2 hours
   - Emails with deadlines
   - Direct mentions awaiting response
   - Blocked or  critical work items
   - Action items assigned to you from yesterday's meetings}

  FULL CONTEXT 

 TODAY'S MEETINGS
  {time | title | key attendees |  conflicts |  related work items}

 YESTERDAY'S OUTCOMES
  {Merged view: decisions + action items from meetings and chats.
   Group by meeting/source. Show YOUR action items in bold.
   Show OTHER PEOPLE's action items you're waiting on.}

 TEAMS ACTIVITY
  {Direct mentions first, then chat summaries.
   Only show chats with relevant activity. Skip quiet chats.}

 EMAIL
  {Combined: VIP/urgent emails + threads awaiting your reply.
   Format: From | Subject | Action needed | Deadline}

 ADO: WHAT CHANGED (last 24h)
  MY ITEMS:
    {Only items whose state changed in last 24h}
    #{ID} | {Type} | {Title} | {old state}  {new state}
  TEAM ITEMS (awareness):
    {Max 10 most relevant team state changes. Summarize the rest as a count.}

 WORK ITEM HEALTH
  {Insert condensed output from work-item-health skill}
  {Shows: health snapshot, velocity, top refactoring suggestions}

 WHAT I WORKED ON YESTERDAY
  {From session_store: Copilot sessions, branches, files edited, PRs/commits}
  {Format: branch | summary | files edited}

 INVENTORY (collapsed)
  Active: {n} items ({n} Tasks, {n} Stories, {n} Bugs, {n} Features)
  By state: {n} New, {n} Active, {n} Ready to Code
  {Do NOT list individual items unless the user asks for --full}

═
```

---

### Step 5: Persist to Notes

After generating the report, save it to the notes repo for historical tracking.

**Directory**: `C:\repos\notes\domains\daily-reports\`
**Filename**: `{YYYY-MM-DD}.md` (e.g., `2026-02-26.md`)

Create the directory if it doesn't exist. If a file already exists for today, append a horizontal rule and the new report below the existing content (allows re-running the report).

The persisted version should include a YAML frontmatter block:
```markdown
---
date: {YYYY-MM-DD}
active_items: {count}
items_closed_this_sprint: {count}
health_critical: {count}
health_warning: {count}
velocity_trend: {//}
---
```

This frontmatter enables future skills to query velocity trends across report files.

After saving, note in the output: ` Saved to C:\repos\notes\domains\daily-reports\{YYYY-MM-DD}.md`

---

## Variant: Full Report (--full)

If the user says "run daily-report --full" or "full daily report", expand the collapsed sections:
- Show all individual active work items with age and priority
- Show all team state changes (not just top 10)
- Show all recently completed items (last 30 days)
- Include the full work-item-health standalone report instead of condensed

---

## Formatting Rules

- **Terse and scannable**  bullet points, no prose paragraphs
- The  RIGHT NOW section should be readable in 10 seconds
- Include Teams message links where available
- For ADO items: `#{ID} | {Type} | {Title} | {State}`
- For meetings: `HH:MM - HH:MM | Title | Key attendees`
- For emails: `From: {sender} | Subject: {subject} | {action}`
- Empty sections: show " Nothing to report"
- Maximum report length target: ~60 lines (delta view), ~180 lines (full view)

## Constraints

- Do not fabricate data  only report what comes from WorkIQ, ADO, session_store, and notes queries
- If a query fails, note the failure in the report and continue with other sections
- Do not send emails or modify work items  this is read-only
- Always include the date in the report header
- If the work-item-health skill is unavailable, fall back to basic aging analysis


