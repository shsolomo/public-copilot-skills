---
name: meeting-recap
description: Analyze a meeting transcript to produce a detailed technical recap with key decisions, action items, discussion topics, and open questions. Use this skill when asked to recap, summarize, or debrief a meeting.
---

# Meeting Recap Skill

## Purpose

After a meeting, quickly extract and organize everything important from the transcript — decisions made, action items assigned, technical details discussed, open questions, and follow-ups. Designed for highly technical engineering meetings where precision matters and no detail should be lost.

## Required Configuration

This skill requires the following from the user's global copilot instructions (`~/.copilot/copilot-instructions.md`):
- `## VIP Contacts` — for flagging action items and contributions from key people
- `## Meeting Keywords` (optional) — recurring meeting keyword strategies for better topic extraction
- `## Notes Repo` (optional) — path to personal notes vault, for saving the recap

If any required configuration is missing, ask the user for the values and offer to add them to their `~/.copilot/copilot-instructions.md`.

## How to Execute

### Step 1: Identify the Meeting

Ask the user for the meeting name if not provided. Then fetch meeting details:

```
workiq ask -q "Find the most recent meeting titled '{meeting title}'. Return the date, time, duration, attendees, and full transcript."
```

If multiple matches are found, present them and ask the user to confirm which one.

If the transcript is truncated or incomplete, run a follow-up query:

```
workiq ask -q "Continue returning the transcript for the meeting '{meeting title}' on {date}. Pick up where you left off."
```

Repeat until the full transcript is retrieved.

### Step 2: Analyze the Transcript

With the full transcript in hand, perform a thorough analysis covering every category below. Do NOT summarize loosely — extract specific details, exact quotes where relevant, and attribute statements to speakers.

**Analysis categories:**

1. **Meeting overview** — Date, duration, attendees, and the general purpose/agenda of the meeting.

2. **Key topics discussed** — Every distinct topic or agenda item covered. For each topic:
   - What was discussed
   - Who drove the discussion
   - What conclusions were reached (if any)

3. **Technical details** — Architecture decisions, design patterns, code changes, system behaviors, performance metrics, infrastructure changes, API contracts, data models, or any other technical specifics mentioned. Be precise — include version numbers, service names, config values, error codes, metric thresholds, and any concrete technical artifacts referenced.

4. **Decisions made** — Explicit decisions the group committed to. Include:
   - The decision itself
   - Who made or approved it
   - Any conditions or caveats

5. **Action items** — Tasks assigned or volunteered for. For each:
   - What needs to be done
   - Who owns it
   - Any stated deadline or urgency
   - Flag items owned by VIP contacts (from `## VIP Contacts` in global instructions)

6. **Open questions & unresolved items** — Topics raised but not resolved, questions left unanswered, or items explicitly deferred.

7. **Risks & concerns raised** — Any risks, blockers, or concerns mentioned by attendees.

8. **Follow-ups & next steps** — Agreed-upon next steps, follow-up meetings, or future checkpoints.

### Step 3: Present the Recap

Format the output as follows:

```
═══════════════════════════════════════════════════
  📝 MEETING RECAP — {meeting title}
  📅 {date} | ⏱️ {duration} | 👥 {attendee count} attendees
═══════════════════════════════════════════════════

👥 ATTENDEES
  • {name1}, {name2}, {name3}, ...

🎯 MEETING OVERVIEW
  {1-3 sentence summary of the meeting's purpose and outcome}

💡 KEY TOPICS DISCUSSED
  1. {Topic Title}
     • {details, who drove it, conclusions}
  2. {Topic Title}
     • {details}
  ...

🔧 TECHNICAL DETAILS
  • {Specific technical detail with exact values}
  • {Architecture or design decision with rationale}
  • {Code change, API contract, or system behavior discussed}
  ...

✅ DECISIONS MADE
  • {Decision} — decided by {who}, {any caveats}
  ...

📋 ACTION ITEMS
  • [ ] {Task description} — Owner: {name} {⭐ if VIP} {deadline if stated}
  • [ ] {Task description} — Owner: {name}
  ...

❓ OPEN QUESTIONS
  • {Question or unresolved item} — raised by {who}
  ...

⚠️ RISKS & CONCERNS
  • {Risk or blocker} — raised by {who}
  ...

➡️ NEXT STEPS
  • {Follow-up action or checkpoint}
  ...

═══════════════════════════════════════════════════
```

### Step 4: Offer Follow-up Actions

After presenting the recap, offer:

1. **Save to notes** — "Would you like me to save this recap to your notes vault?"
   - If yes, save to `{NOTES_PATH}/domains/{meeting-domain}/` or `{NOTES_PATH}/inbox/` using the notes skill
2. **Create ADO work items** — "Would you like me to create work items for any of the action items?"
3. **Share via clipboard** — "Would you like me to copy this recap to your clipboard for sharing?"

## Constraints

- This is read-only by default — do not create work items, send messages, or modify anything unless the user explicitly opts in via the follow-up actions
- Do not fabricate or infer information not present in the transcript — if something is unclear, flag it as an open question
- Attribute statements and action items to specific people whenever the transcript makes this clear
- Preserve technical precision — do not simplify or generalize technical details (keep exact names, numbers, versions, and config values)
- If the transcript is unavailable or empty, inform the user and suggest they check whether the meeting was recorded
- Keep the recap comprehensive but scannable — use bullet points, not paragraphs
- Omit any section that has no content (e.g., if no risks were raised, skip the Risks section)

## Examples

### Basic usage
```
User: "Recap the AET Hub Hour meeting"
→ Skill fetches transcript via workiq, analyzes it, produces the structured recap
```

### With date disambiguation
```
User: "Recap the Triitus sync"
→ workiq returns multiple matches
→ Skill presents: "I found 3 instances of Triitus Weekly Sync. Which one?"
   1. Feb 25, 2026 (most recent)
   2. Feb 18, 2026
   3. Feb 11, 2026
→ User picks one, skill proceeds with that transcript
```
