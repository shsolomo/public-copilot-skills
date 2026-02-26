---
name: work-item-health
description: Analyzes ADO work items for sprint-aware aging, type-specific health, velocity tracking, and refactoring suggestions. Use this skill when asked about work item health, velocity, aging, or when other skills need work item analysis context.
---

# Work Item Health Skill

## Purpose

Analyze Azure DevOps work items against sprint cadence expectations to surface:
- Items that are aging beyond type-appropriate thresholds
- Velocity metrics (items closed per sprint by type)
- Refactoring suggestions (e.g., promote overdue task to user story)
- Lingering item root-cause hints from cross-referencing ADO activity, notes, and session history

This skill is designed to be **invoked by other skills** (e.g., daily-report) or directly by the user.

## Required Configuration

Reads from the user's global copilot instructions (`~/.copilot/copilot-instructions.md`):
- `## Azure DevOps Defaults`  Organization, Project, Area Path

## Sprint Cadence Model

The team uses **2-week (14-day) sprints**. Aging is measured in sprint increments.

### Type-Specific Expectations

| Work Item Type | Expected Cycle Time | Throughput Target | Aging Thresholds |
|---|---|---|---|
| **Task** | < 5 days | Multiple per sprint |  >5d,  >1 sprint (14d),  >2 sprints (28d) |
| **User Story** | 1 sprint (14 days) | 1-2 per sprint |  >1 sprint,  >2 sprints (28d),  >3 sprints (42d) |
| **Bug** | < 5 days (P1-P2), < 1 sprint (P3+) | Varies | Same as Task for P1-P2, same as Story for P3+ |
| **Feature** | Multi-sprint OK | ~1 per quarter |  >3 sprints (42d),  >6 sprints (84d),  >9 sprints (126d) |

### Refactoring Suggestions

When the skill detects items exceeding their type expectations, it should suggest structural changes:

- **Task > 5 days active**: "Consider promoting to User Story with child Tasks to break down the work."
- **User Story > 2 sprints**: "Consider promoting to Feature, or decompose into smaller Stories."
- **Feature > 6 sprints with no child items closed**: "Feature may be too large or stalled  consider breaking into smaller Features or re-scoping."
- **Any item > 3 sprints with no ADO activity (comments, state changes)**: "Item appears abandoned  close, re-scope, or re-assign."

## How to Execute

### Step 1: Fetch Active Work Items

**IMPORTANT:** Always use `-o table` for `az boards query`. Do NOT use `-o json`.

```powershell
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State], [System.WorkItemType], [System.CreatedDate], [Microsoft.VSTS.Common.Priority], [Microsoft.VSTS.Scheduling.StoryPoints], [Microsoft.VSTS.Common.ActivatedDate], [Microsoft.VSTS.Common.StateChangeDate] FROM WorkItems WHERE [System.AssignedTo] = @Me AND [System.State] NOT IN ('Closed', 'Removed', 'Done') AND [System.AreaPath] UNDER '{ADO_AREA_PATH}' ORDER BY [System.WorkItemType] ASC, [Microsoft.VSTS.Common.Priority] ASC" --org "{ADO_ORG}" -p "{ADO_PROJECT}" -o table
```

### Step 2: Fetch Recently Closed Items (velocity window: last 28 days / 2 sprints)

```powershell
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State], [System.WorkItemType], [Microsoft.VSTS.Common.ClosedDate], [Microsoft.VSTS.Common.ActivatedDate] FROM WorkItems WHERE [System.AssignedTo] = @Me AND [System.State] IN ('Closed', 'Done') AND [Microsoft.VSTS.Common.ClosedDate] >= @Today - 28 AND [System.AreaPath] UNDER '{ADO_AREA_PATH}' ORDER BY [Microsoft.VSTS.Common.ClosedDate] DESC" --org "{ADO_ORG}" -p "{ADO_PROJECT}" -o table
```

### Step 3: Calculate Metrics

For each active work item, calculate:

1. **Age in days**: `today - ActivatedDate` (or `CreatedDate` if never activated)
2. **Age in sprints**: `ceil(age_days / 14)`
3. **Health status**: Compare age against type-specific thresholds above
4. **Last activity**: Check `StateChangeDate`  if >14 days since any state change, flag as "stale"

For velocity:
1. **Current sprint velocity**: Count items closed in last 14 days, grouped by type
2. **Previous sprint velocity**: Count items closed 15-28 days ago, grouped by type
3. **Trend**: Compare current vs previous sprint ( improving,  steady, ↓ declining)

### Step 4: Cross-Reference for Root Causes (optional, when invoked standalone)

For items flagged  or , check:

1. **ADO comments**: Use `az boards work-item show --id {ID} --fields System.History` to check for blockers mentioned in comments
2. **Notes repo**: Search `C:\repos\notes` for mentions of the work item ID or title keywords
3. **Session store**: Query `session_store` search_index for work item IDs to see if recent Copilot sessions touched related code

```sql
SELECT content, session_id, source_type 
FROM search_index 
WHERE search_index MATCH '{work_item_id} OR {title_keywords}'
ORDER BY rank LIMIT 5;
```

### Step 5: Generate Report

#### When invoked standalone, output the full report:

```
 WORK ITEM HEALTH REPORT  {date}
Sprint cadence: 2-week | Velocity window: last 2 sprints

 VELOCITY
  Current sprint:  {n} Tasks, {n} Stories, {n} Bugs closed
  Previous sprint: {n} Tasks, {n} Stories, {n} Bugs closed
  Trend: {//}

 CRITICAL (3+ sprints overdue for type)
  #{ID} | {Type} | {Title} | {age}d ({n} sprints) | Last activity: {date}
     Suggestion: {refactoring suggestion}
     Context: {root cause hints from ADO/notes/sessions}

 WARNING (2 sprints overdue for type)
  #{ID} | {Type} | {Title} | {age}d ({n} sprints) | Last activity: {date}
     Suggestion: {refactoring suggestion}

 WATCH (1 sprint overdue for type)
  #{ID} | {Type} | {Title} | {age}d ({n} sprints)

 HEALTHY
  {count} items within expected cycle time for their type

 REFACTORING SUGGESTIONS
  #{ID} "{Title}"  Task active {n} days  Promote to User Story with child Tasks
  #{ID} "{Title}"  Story active {n} sprints  Decompose or promote to Feature
```

#### When invoked by another skill (e.g., daily-report), return a condensed summary:

```
� HEALTH SNAPSHOT: {n}  critical, {n}  warning, {n}  watch, {n}  healthy
 VELOCITY: {n} items closed this sprint ({trend} vs last sprint)
 REFACTORING: {one-line suggestions, max 3}
 TOP CONCERN: #{ID} "{Title}"  {type} at {n} sprints, {suggestion}
```

## Formatting Rules

- Always show age in both days and sprint increments: `32d (3 sprints)`
- Sort by severity (      ), then by age descending
- For velocity, always show by-type breakdown and trend arrow
- Refactoring suggestions should be actionable: include the specific structural change to make
- When cross-referencing, cite sources: "Discussed in Hub Hour on 2/25", "Session: Query Azure SQL From VM"
- Empty sections: show " None" instead of omitting (confirms analysis was done)

## Constraints

- Read-only: do not modify work items, create child items, or change ADO state
- Do not fabricate activity data  only report what comes from ADO, notes, and session_store queries
- If a query fails, note the failure and continue with available data
- Sprint boundaries are approximated from item age (no ADO iteration path query required)
