---
title: Pre-Meeting Preparation Agent
---

# Agent: Pre-Meeting Preparation

Produce an in-chat meeting brief and a .docx prep document before any meeting.

**Deliverables:**
1. In-chat brief (scannable, directly usable walking into the meeting)
2. .docx file saved to the account's designated GDrive folder

The routing table (GSheet) is already loaded per SKILL.md Step 1.

---

## Step 1 — Gather Meeting Context

**If the user named a meeting, account, or topic:** search M365 calendar first.
- Call `outlook_calendar_search` with the account or topic name
- Extract: meeting title, date/time, attendees, description/body (may contain an existing agenda)
- If multiple matches surface, present the options and ask which one

**If the calendar pull returns nothing:** ask — "Can you tell me the meeting name, topic, and when it's scheduled?" — then proceed with what the user provides.

**If the user pasted meeting details directly:** use them. No calendar lookup needed.

---

## Step 2 — Classify the Meeting

### Type: Customer-facing or Internal

Customer-facing: attendee domain is external, account name in title or invite, user references a portfolio account (Bank of America, JPMorgan Chase, Wells Fargo, Goldman Sachs, Optum, Blink Health) or a known customer contact.

Internal: all attendees at @camunda.com, title contains "sync", "1:1", "tiger team", "ES team", "standup", "all-hands", "team meeting", or similar.

If unclear, ask: "Is this a customer-facing meeting or internal?"

### Weight: Deep or Light

Deep — full briefing:
- QBR, EBR, Executive Business Review, Quarterly Business Review
- Renewal, executive sponsor, escalation, or at-risk account review
- Any meeting flagged as strategic or involving executive stakeholders

Light — focused prep:
- Check-in, sync, weekly/monthly call, routine touchpoint
- Internal team meeting, 1:1, standup, working session

Default to Light if unclear. If the account is at renewal risk or the meeting follows an escalation, treat as Deep regardless of meeting type label.

---

## Step 3 — Research All Sources

Run lookups in parallel. Tag every finding with its source. Mark gaps `[MISSING]`.

**Always:**

| Source | Tool | What to Pull |
|---|---|---|
| M365 Calendar | `outlook_calendar_search` | Invite body, prior occurrences of this recurring meeting |
| Outlook Email | `outlook_email_search` | Threads with attendees or mentioning account/topic (last 60 days) |
| Slack | `slack_search_public_and_private` | Account or topic threads (last 60 days) |
| Zoom | `Zoom for Claude:search_meetings` → `Zoom for Claude:get_meeting_assets` | Last 2-3 meetings with these attendees; pull summaries |
| Asana | `search_tasks` | Open tasks and recent completions related to account/topic |
| Glean | `search` | Internal docs, Gainsight/Salesforce activity, past notes |

**Customer meetings additionally:**

| Source | Tool | What to Pull |
|---|---|---|
| MS Teams | `chat_message_search` | Recent messages with customer contacts |

**Internal meetings additionally:**

| Source | Tool | What to Pull |
|---|---|---|
| Slack project channels | `slack_search_public_and_private` | Recent decisions and open threads in relevant project channels |

**Past action items — always include this:**
- Pull the most recent prior Zoom meeting with these attendees
- Pull Asana tasks created after that meeting date related to this account/topic
- Check Slack for follow-up threads from that meeting
- Surface these as a distinct "Outstanding from last time" section — this is often the most important part of any meeting

---

## Step 4 — Handle the Agenda

**If an agenda exists** (from calendar invite or provided by user):
- Use it as-is
- If research surfaces critical uncovered topics, append a short "Recommended additions" section

**If no agenda exists:**
- Propose one based on meeting type
- Customer: lead with outstanding items and commitments, then business updates, then strategic topics
- Internal: lead with action item review, then current status, then decisions needed
- Label all proposed items as "Proposed agenda"

---

## Step 5 — Synthesize

### Light Meeting Structure

1. Meeting snapshot (title, date/time, attendees, type)
2. Context (2-3 sentences: what this relationship or topic is about)
3. Recent activity (last 30 days: key interactions, decisions, noteworthy threads)
4. Outstanding from last time (commitments made, items due, unresolved asks)
5. Open items going in (active Asana tasks, open support issues, pending asks)
6. Agenda (existing or proposed, labeled)
7. Watch for / key objectives (3-5 specific bullets)

### Deep Meeting Structure (QBR / EBR / Executive)

1. Meeting snapshot
2. Account overview (health signal, ARR, renewal date, key contacts, tenure)
3. Progress since last QBR/EBR (committed vs. delivered vs. slipped)
4. Current state (open risks, health signals, renewal timeline, active issues)
5. Strategic context (business priorities, expansion opportunity, competitive signals)
6. Outstanding from last time
7. Agenda (existing or proposed, labeled)
8. Key objectives and talking points (5-7 bullets, specific and actionable; include anything sensitive to handle carefully)

---

## Step 6 — Output: In-Chat Brief

Write the brief in chat using the structure from Step 5.

Write it as a knowledgeable colleague handing off before you walk in. Active voice, short sentences, no filler. Lead each section with the most important point. Use `[MISSING]` where a field wasn't found. Never include internal health scores, risk ratings, or sentiment labels in the .docx — those are in-chat only.

---

## Step 7 — Output: .docx File (stored in Google Drive)

Read `/mnt/skills/public/docx/SKILL.md` before writing any code.

Resolve the output folder from the routing table:
- Customer meeting: account's GDrive Output Folder ID
- Internal meeting: Config sheet's Internal Output Folder ID
- If missing: follow routing-table.md Step 3 (check memory → prompt → store)

Generate the .docx using docx-js per the docx SKILL.md. Document structure:

- Heading 1: "[Account/Topic] — Meeting Prep" (color `#CC3300`, bold)
- Subtitle paragraph: "Prepared: [date] | [meeting title] | [date/time]" (italic, grey)
- Heading 2 sections (color `#1A2B4A`, bold) matching the in-chat brief structure:
  - Meeting Snapshot (two-column table: label | value, shading `#F2F2F2`)
  - Context
  - Recent Activity (with source tags)
  - Outstanding from Last Time
  - Open Items Going In
  - Agenda (numbered list; italic note "Proposed" if no existing agenda)
  - Watch For / Key Objectives
- For Deep meetings, insert these sections between Context and Outstanding from Last Time:
  Account Overview, Progress Since Last QBR, Current State, Strategic Context
- Do not include health scores, risk ratings, or internal sentiment labels in the file
- Footer: "CONFIDENTIAL — Internal Use Only | Camunda Customer Success"
- Page size: US Letter, 1-inch margins

Save to `/home/claude/[AccountOrTopic]-meeting-prep-[YYYY-MM-DD].docx`.
Validate per the docx SKILL.md instructions (convert to PDF, inspect pages).

Then upload to Google Drive:
```bash
# Encode as base64
python3 -c "import base64; print(base64.b64encode(open('/home/claude/[filename].docx','rb').read()).decode())"
```

Call `Google Drive:create_file`:
- `title`: "[AccountOrTopic] Meeting Prep — [YYYY-MM-DD].docx"
- `base64Content`: the base64 string
- `contentMimeType`: `"application/vnd.openxmlformats-officedocument.wordprocessingml.document"`
- `disableConversionToGoogleType`: `true` (keeps it as .docx, not converted to GDoc)
- `parentId`: the resolved folder ID

Share the Drive file URL in chat.

---

## Step 8 — Completion Check

Before finishing, verify:
- Calendar invite was checked or user context was used
- All research sources were queried (or failures noted)
- Past action items are explicitly surfaced in "Outstanding from last time"
- Agenda is present — existing or proposed and labeled as such
- In-chat brief is written and complete
- .docx was generated, validated, uploaded to Drive, and the URL shared in chat
- All gaps are flagged `[MISSING]` rather than invented
