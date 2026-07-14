---
title: Post-Meeting Processing Agent
---

# Agent: Post-Meeting Processing

Process a completed meeting into notes, action items, and Asana tasks.
The routing table is already loaded per SKILL.md Step 1.

## Tool Reference (use exact names — no others exist for these functions)

| Function | Tool |
|---|---|
| Create Asana tasks | `Asana:create_tasks` |
| Search Asana projects | `Asana:search_objects` with `resource_type: "project"` |
| Search Asana tasks (duplicate check) | `Asana:search_tasks` with `text` param |
| My Tasks (Aaron's personal list) | `Asana:create_tasks` with `default_assignee: "me"`, no `default_project` |
| Upload .docx to Drive | `Google Drive:create_file` with `base64Content`, `contentMimeType`, `disableConversionToGoogleType: true` |
| Download xlsx/docx from Drive | `Google Drive:download_file_content` — returns raw base64 for non-Google files |

---

## Step 1 — Get the Transcript

**If the user referenced a Zoom meeting by name or description:**
- Call `Zoom for Claude:search_meetings` with the account name, topic, or attendee name as the `q` parameter
- If multiple meetings match, present them with dates and titles and ask to confirm
- Once confirmed, call `Zoom for Claude:get_meeting_assets` with the `meeting_uuid` to retrieve the transcript and summary
- If Zoom returns no transcript (recording disabled or still processing): tell the user and ask them to paste

**If the user pasted a transcript or uploaded a file:**
- Use the provided content directly

**If neither:** ask — "Can you paste the transcript, share the Zoom meeting ID, or tell me the meeting name?"

---

## Step 2 — Classify the Meeting

Customer-facing: external attendee domains, account name in title, transcript references a portfolio account or known contact.

Internal: all attendees @camunda.com, topic is a team sync, 1:1, tiger team, or working session.

Known customer contacts: maintained in your local copy only (not committed here — see
`## Known Contacts` in the local Admin Config block). This keeps named individuals at
customer accounts out of the public repo the same way Drive folder IDs and Asana GIDs
are kept local-only elsewhere in this skill set.

If unclear after reading the transcript, ask one question.

---

## Step 3a — Customer Meeting Processing

No .docx file is created for customer meetings. The CS toolkit's recap email and `.eml` are the written deliverables.

First read: `/mnt/skills/organization/customer-success-toolkit/references/camunda-context.md`

Then read and follow all instructions in: `/mnt/skills/organization/customer-success-toolkit/agents/meeting-followup.md`

That agent produces:
1. Customer recap email (HTML)
2. Internal notes (sentiment, health signals, Salesforce activity log fields, risks)
3. `.eml` file (saved and presented for download)
4. Slack summary (posted inline as a code block)

Do not duplicate those outputs. When the CS toolkit agent completes, continue to Step 4.

---

## Step 3b — Internal Meeting Processing

Generate the following directly. No customer email. No `.eml`.

### In-Chat Meeting Minutes

```
Meeting: [Title]
Date: [Date] | Duration: [if known]
Attendees: [Names and roles]

Summary
[2-3 sentences: what was discussed and the net outcome]

Key Decisions
- [Decision with owner if named]

Action Items
| Action | Owner | Due Date |
|---|---|---|
| [action] | [name] | [date or TBD] |

Blockers / Open Questions
[Include only if present; omit section if none]

Next Meeting
[Date and type if scheduled, or "Not yet scheduled"]
```

### Slack Summary (inline as a code block)

```
📋 *[Meeting Name]* | [Date]
*Attendees:* [first names or roles]
*Summary:* [one sentence]

*Decisions:*
- [decision]

*Actions:*
- [Owner]: [action] — due [date or TBD]

*Next:* [date/type or "Not scheduled ⚠️"]
```

Rules: short sentences, no filler. `[VERIFY]` for anything unconfirmed. Never invent.

### .docx File — Internal Meeting Minutes

Read `/mnt/skills/public/docx/SKILL.md` before writing any code.

Resolve the output folder from the routing table Config sheet (`Internal Output Folder ID`).
If missing, follow routing-table.md Step 3 (check memory → prompt → store).

Generate the .docx using docx-js per the docx SKILL.md:

- Heading 1: "[Meeting Name] — Minutes" (bold, dark navy `#1A2B4A`)
- Subtitle: "[Date] | [Duration if known]" (italic, grey)
- Heading 2 sections (bold, `#1A2B4A`): Attendees, Summary, Key Decisions, Action Items, Blockers / Open Questions (omit if none), Next Meeting
- Action Items as a table: Action | Owner | Due Date, header row shading `#F2F2F2`, alternating rows `#FAFAFA`
- `[VERIFY]` items in italic, amber `#E6A800`
- Footer: "INTERNAL — Camunda Customer Success"
- Page size: US Letter, 1-inch margins

Save to `/home/claude/[MeetingName]-minutes-[YYYY-MM-DD].docx`.
Validate per the docx SKILL.md instructions (convert to PDF, inspect pages).

Upload to Google Drive:
```bash
python3 -c "import base64; print(base64.b64encode(open('/home/claude/[filename].docx','rb').read()).decode())"
```

Call `Google Drive:create_file`:
- `title`: "[Meeting Name] Minutes — [YYYY-MM-DD].docx"
- `base64Content`: the base64 string
- `contentMimeType`: `"application/vnd.openxmlformats-officedocument.wordprocessingml.document"`
- `disableConversionToGoogleType`: `true`
- `parentId`: the resolved internal folder ID

Share the Drive file URL in chat.

---

## Step 4 — Extract and Route Asana Tasks

Parse all action items from the output of Step 3 (CS toolkit output for customer; minutes you generated for internal).

### 4a — Check for Duplicates

For each action item, call `Asana:search_tasks` with `text` set to a key phrase from the item.
If a matching open task already exists, note it in the summary and skip creation.

### 4b — Classify Ownership

- **Aaron's task:** Aaron is explicitly named, or the action is clearly TAM/CS-side work
- **Account task:** action relates to a specific customer account deliverable
- **Other owner:** action belongs to a named teammate
- **Unclear:** flag and ask before creating

### 4c — Determine Project

**Account tasks:**

1. Look up the account in the loaded routing table
2. If the Asana GID is populated: use it directly — no confirmation needed
3. If GID is blank:
   - Check memory for `asana_gid [AccountName]`
   - If found: use it
   - If not: call `Asana:search_objects` with `resource_type: "project"` and `query` set to the account name
   - Present the matches to Aaron; ask him to confirm before creating
   - After confirmation: call `memory_user_edits` to store the GID + output a routing table update block

**Aaron's personal tasks:**
- Use `Asana:create_tasks` with `default_assignee: "me"` and no `default_project`
- This routes directly to Aaron's My Tasks list without needing a project GID

**Other-owner tasks:**
- Call `Asana:search_objects` with `resource_type: "user"` and the person's name to find their user GID
- Set `assignee` to their GID on the task; set the same `project_id` as Aaron's task if it's the same project

### 4d — Create Tasks

Call `Asana:create_tasks` with an array of task objects:

```
name: clear, actionable verb phrase
  (e.g., "Send BofA the Zeebe sizing guide", "Schedule JPMC architecture review")
assignee: "me" for Aaron, user GID for others
due_on: YYYY-MM-DD if stated; omit if not
notes: "[Meeting Name] | [Date]\n[verbatim action item text from transcript]"
project_id: the resolved account GID (omit for My Tasks)
```

### 4e — Task Summary

After all tasks are handled, output:

```
Asana Tasks:
✓ [Task name] → [Project or My Tasks] (due [date or none])
↩ [Task name] → already exists, skipped
? [Task name] → waiting for project confirmation
```

---

## Step 5 — Completion Check

Verify before finishing:
- Transcript was retrieved (not skipped)
- Customer: CS toolkit produced all four deliverables (email, .eml, internal notes, Slack summary)
- Internal: in-chat minutes, Slack summary, and .docx file are all present
- All action items were created, flagged as duplicates, or held for confirmation
- Nothing was invented — every output traces to the transcript
- `[VERIFY]` is on anything unconfirmed

---

## Step 6 — Patch Today's Brief (if applicable)

If this meeting appears in today's most recent Daily Brief as a Section 1 Part A item
carrying the "not found — needs input" badge, patch it per the `daily-brief` skill's
Post-Meeting Patch Runs protocol (see that skill's `SKILL.md`, under HTML Output):

1. List files in the daily-brief `BRIEF_OUTPUT_FOLDER_ID` matching
   `Daily Brief_<today>_*.html`, take the latest by filename timestamp.
2. Download it, find the `.item[data-id]` block matching this meeting by title text
   (not `data-id` — that's not stable across files).
3. If found: remove the `bbad` badge, replace the subtitle with a one-line outcome,
   swap the button row (drop the `claude://` deep link, keep/update the Asana link,
   add any new real link). Leave everything else untouched.
4. If not found: stop silently, note it in the completion summary.
5. Save as a new `Daily Brief_YYYY-MM-DD_hh-mm.html` (current time), same folder —
   there is no Drive delete/update tool available, so this does not attempt to
   remove or overwrite the original file.
6. Don't reproduce brief content in chat — one line plus the Drive link, same as
   any other brief-touching operation.

If the daily-brief skill isn't installed or its Drive folder ID isn't configured,
skip this step silently — it's a nice-to-have sync, not a requirement for post-meeting
processing to be considered complete.
