---
name: meeting-manager
description: "Two-phase meeting skill for customer and internal meetings. PRE-MEETING produces an in-chat brief and .docx file from M365 calendar, Outlook, Slack, Zoom, Glean, Asana, and Teams. Depth auto-scales: full briefing for QBRs/EBRs, focused prep for check-ins. POST-MEETING processes transcripts from Zoom MCP or pasted content. Customer meetings route through the CS toolkit meeting-followup agent then log Asana tasks to the account project. Internal meetings generate minutes, Slack summary, and .docx file. Per-account GDrive folders and Asana GIDs stored in a GSheet routing table; confirmed values cached so never prompted again. Trigger on: \"prep for my meeting\", \"meeting prep\", \"pre-meeting for X\", \"prep me for my call\", \"what do I need before my call/QBR/EBR\", \"run pre-meeting\", \"post-meeting\", \"process this transcript\", \"meeting notes for X\", \"follow up from my call\", \"write up this meeting\", \"log action items\", \"run post-meeting\". Always use for meeting prep or post-meeting work."
---

# Meeting Manager

Two-phase skill: pre-meeting preparation and post-meeting processing.

## Admin Config

Configure these in your local copy (not committed here, since they're account-specific):

```
ROUTING_TABLE_ID: UNSET
DAILY_BRIEF_OUTPUT_FOLDER_ID: <the daily-brief skill's BRIEF_OUTPUT_FOLDER_ID, for Step 6 patch runs in post-meeting.md>
```

Replace `UNSET` with the Google Sheets file ID once the routing table GSheet has been
created. Until then, the skill will run first-time setup on every invocation.

### Known Contacts (local only)

Named individuals at customer accounts are kept out of this repo. In your local copy,
add a block like:

```
## Known Contacts
- [Account]: [Contact name], [Contact name]
```

`agents/post-meeting.md` Step 2 references this block when classifying a meeting as
customer-facing vs. internal.

## Step 1 — Load Routing Table (REQUIRED — do not skip)

Read `references/routing-table.md` and follow its instructions to locate and load the
routing table GSheet.

If the routing table cannot be loaded for any reason — ID is UNSET, GSheet not found,
Drive access fails — **stop immediately**. Do not proceed to Step 2. Do not emit a
[MISSING] note and continue. Run the First-Run Setup section of routing-table.md
instead and wait for the user to complete it.

Only proceed to Step 2 once the routing table is successfully loaded.

## Step 2 — Detect Mode

Pre-meeting signals: upcoming or scheduled meeting, "prep", "what do I need to know
before", "get me ready", meeting name without transcript.

Post-meeting signals: transcript provided or referenced, "just had a call", "follow up",
"meeting notes", "write up", "process this", "log action items."

If genuinely ambiguous, ask: "Are you prepping for an upcoming meeting or processing
one that just happened?"

## Step 3 — Route

| Mode | Load |
|---|---|
| Pre-meeting | `agents/pre-meeting.md` |
| Post-meeting | `agents/post-meeting.md` |
