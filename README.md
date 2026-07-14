# meeting-manager

A Claude skill for meeting prep and follow-up. Two phases:

- **Pre-meeting** — produces an in-chat brief and a `.docx` prep document from Outlook calendar, email, Slack, Zoom, Glean, and Asana. Depth scales automatically: full briefing for QBRs/EBRs, focused prep for routine check-ins.
- **Post-meeting** — processes a transcript (from Zoom MCP or pasted content) into notes, action items, and Asana tasks. Customer meetings route through the CS toolkit's `meeting-followup` agent; internal meetings get in-chat minutes, a Slack summary, and a `.docx` file.

Per-account Google Drive folders and Asana project GIDs are stored in a Google Sheet routing table (`references/routing-table.md` handles locating, reading, and updating it). Confirmed values are cached in Claude memory so the same account is never re-prompted.

## Prerequisites

### MCP Connectors

| Connector | Used for |
|-----------|----------|
| Microsoft 365 | Calendar, email, and Teams chat search |
| Slack | Channel and DM search, posting summaries |
| Zoom | Transcript and AI summary retrieval |
| Asana | Task creation and duplicate checking |
| Google Drive | Routing table and output document storage |
| Glean | Internal docs and account context |

### Google Drive

- A Google Sheet for the routing table — created automatically on first run if `ROUTING_TABLE_ID` is `UNSET`, or copy an existing file's ID into `SKILL.md`
- Per-account output folders and an internal-meetings output folder — resolved and cached the first time each is needed

### Local-only configuration

This skill deliberately keeps two categories of data out of the repo:

- **Routing IDs** (`ROUTING_TABLE_ID`, `DAILY_BRIEF_OUTPUT_FOLDER_ID`) — set these in the `## Admin Config` block of your local `SKILL.md`
- **Known customer contacts** — real names tied to real accounts; add a `## Known Contacts` block to your local `SKILL.md` per the template in Admin Config

## Integration with daily-brief

If the [`daily-brief`](https://github.com/aaron-hubbart/daily-brief) skill is also installed, `post-meeting.md` Step 6 will patch a matching "not found — needs input" item in today's most recent brief file after processing completes, rather than waiting for the next scheduled brief run. This step no-ops silently if daily-brief isn't installed or its output folder isn't configured.

## Structure

- `SKILL.md` — entry point, routing table load, mode detection
- `agents/pre-meeting.md` — pre-meeting research and document generation
- `agents/post-meeting.md` — transcript processing, task extraction, brief patching
- `references/routing-table.md` — routing table location, parsing, and update procedures
