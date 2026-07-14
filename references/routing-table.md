# Routing Table

How to locate, read, and update the Excel file that stores account→Asana GID and
account→GDrive folder mappings. Follow these steps on every skill invocation before
doing anything else.

The routing table is an .xlsx file stored in Google Drive. It is generated and updated
programmatically using openpyxl — no user paste-in required.

---

## Step 1 — Resolve the File ID

Check in this order:

**1a. Claude memory** — Look through memory for a control containing
`meeting_manager_config_id`. If found, extract the ID and jump to Step 2.

**1b. SKILL.md config** — Read the `ROUTING_TABLE_ID` value from SKILL.md.
If it is not `UNSET` and looks like a Drive file ID (~28–33 alphanumeric characters,
no spaces or brackets), use it and jump to Step 2.

**1c. Neither found** — Run First-Run Setup.

---

## First-Run Setup

Tell the user: "I need to create your Meeting Manager Config file. Creating it now."

Then run the following bash script to generate the xlsx:

```python
# /tmp/create_routing_table.py
import openpyxl
from openpyxl.styles import Font, PatternFill, Alignment

wb = openpyxl.Workbook()

# --- Accounts sheet ---
ws = wb.active
ws.title = "Accounts"

headers = ["Account Name", "Aliases", "Asana Project GID",
           "GDrive Output Folder Name", "GDrive Output Folder ID",
           "GDrive Output Folder URL"]
ws.append(headers)

# Style header row
for cell in ws[1]:
    cell.font = Font(bold=True, color="FFFFFF")
    cell.fill = PatternFill("solid", start_color="1A2B4A")
    cell.alignment = Alignment(horizontal="center")

# Seed data
rows = [
    ["Bank of America", "BofA, BoA", "", "", "", ""],
    ["JPMorgan Chase",  "JPMC",      "",                 "", "", ""],
    ["Wells Fargo",     "WF",        "",                 "", "", ""],
    ["Goldman Sachs",   "GS",        "",                 "", "", ""],
    ["Optum",           "UnitedHealth Group", "",         "", "", ""],
    ["Blink Health",    "",          "",                 "", "", ""],
]
for row in rows:
    ws.append(row)

# Column widths
col_widths = [20, 20, 22, 28, 36, 60]
for i, w in enumerate(col_widths, 1):
    ws.column_dimensions[ws.cell(row=1, column=i).column_letter].width = w

# --- Config sheet ---
cfg = wb.create_sheet("Config")
cfg.append(["Key", "Value"])
for cell in cfg[1]:
    cell.font = Font(bold=True, color="FFFFFF")
    cell.fill = PatternFill("solid", start_color="1A2B4A")
cfg.column_dimensions["A"].width = 30
cfg.column_dimensions["B"].width = 60

cfg_rows = [
    ["Internal Output Folder Name", ""],
    ["Internal Output Folder ID",   ""],
    ["Internal Output Folder URL",  ""],
    ["Created",                     ""],
]
for row in cfg_rows:
    cfg.append(row)

wb.save("/tmp/meeting-manager-config.xlsx")
print("done")
```

Run: `python3 /tmp/create_routing_table.py`

Then upload to Google Drive:
1. Read the file as base64:
   ```bash
   python3 -c "import base64; print(base64.b64encode(open('/tmp/meeting-manager-config.xlsx','rb').read()).decode())"
   ```
2. Ask where to store it using the **Folder Resolution Procedure** below.
   If the user has no preference, omit `parentId` to place it in Drive root.
3. Call `Google Drive:create_file` with:
   - `title`: "Meeting Manager Config.xlsx"
   - `base64Content`: the base64 string from step 1
   - `contentMimeType`: `"application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"`
   - `disableConversionToGoogleType`: `true` (keeps it as xlsx, not auto-converted to GSheet)
   - `parentId`: resolved folder ID, or omit for Drive root
4. From the response, extract the file ID.
5. Store in memory: `memory_user_edits(command="add", control="meeting_manager_config_id: [ID]")`
6. Tell the user the file is ready and share the Drive URL. Instruct them to set
   `ROUTING_TABLE_ID: [ID]` in SKILL.md for permanent persistence.

---

## Folder Resolution Procedure

Use this any time a folder ID is needed but unknown. Never ask the user for a folder ID
directly — they should only ever need to provide a name, a path, or a Drive URL.

**Step A — Ask for name and path**

Ask the user: "What's the folder name and where does it live in your Drive?
For example: 'BofA Meeting Docs' inside 'Customer Success > Bank of America'."

**Step B — Try to find it automatically**

Call `Google Drive:list_recent_files` (up to 50 results). Scan the results for a
folder whose name matches what the user described. Drive file metadata includes
`mimeType`; folders have `mimeType: "application/vnd.google-apps.folder"`.

- One clear match: confirm with the user ("Found 'BofA Meeting Docs' — is that the
  right folder?") and use its ID.
- Multiple possible matches: list them with their names and parent paths, ask the
  user to pick.
- No match: proceed to Step C.

**Step C — Fall back to Drive URL**

Tell the user: "I couldn't locate that folder automatically. Open it in Google Drive,
copy the URL from your browser's address bar, and paste it here."

Extract the folder ID from the URL:
```python
import re
match = re.search(r'/folders/([a-zA-Z0-9_-]+)', url)
folder_id = match.group(1) if match else None
```

Supported URL formats:
- `https://drive.google.com/drive/folders/FOLDER_ID`
- `https://drive.google.com/drive/u/0/folders/FOLDER_ID`
- `https://drive.google.com/drive/folders/FOLDER_ID?...`

Confirm the extracted ID back to the user before storing it.

---

## Step 2 — Read the Routing Table

Download and parse the xlsx locally:

```bash
# 1. Download from Drive
# Call: Google Drive:download_file_content(fileId=ID)
# Response contains base64-encoded file content.

# 2. Decode and save
python3 -c "
import base64, sys
data = sys.argv[1]  # the base64 string from the API response
with open('/tmp/routing-table.xlsx', 'wb') as f:
    f.write(base64.b64decode(data))
"

# 3. Parse with pandas
python3 -c "
import pandas as pd, json
accounts = pd.read_excel('/tmp/routing-table.xlsx', sheet_name='Accounts', dtype=str).fillna('')
config   = pd.read_excel('/tmp/routing-table.xlsx', sheet_name='Config',   dtype=str).fillna('')
print('ACCOUNTS:', accounts.to_json(orient='records'))
print('CONFIG:',   config.to_json(orient='records'))
"
```

Parse the printed JSON to get the lookup data. Index accounts by Account Name and
all alias values (split on comma, strip whitespace, case-insensitive).

If `download_file_content` fails (permissions error, file not found): tell the user,
ask them to verify the file ID and Drive permissions. Do not proceed.

---

## Step 3 — Resolve Account Data

Given an account name from the meeting context, match against Account Name or any
alias (case-insensitive, strip whitespace).

### Asana Project GID

If populated in the xlsx: use it directly.

If blank:
1. Check memory for a control containing `asana_gid [AccountName]`
2. If found: use it
3. If not: call `Asana:search_objects(resource_type="project", query=[AccountName])`
4. Present matches, ask Aaron to confirm
5. After confirmation: go to Step 4

### GDrive Output Folder (per account)

If populated in the xlsx: use it directly.

If blank:
1. Check memory for `gdrive_folder [AccountName]`
2. If found: use it
3. If not: use the **Folder Resolution Procedure** to locate the correct folder
4. After confirmation: go to Step 4

### Internal Meeting Folder

Read `Internal Output Folder ID` from the Config sheet.

If blank:
1. Check memory for `gdrive_folder_internal`
2. If found: use it
3. If not: use the **Folder Resolution Procedure** to locate the correct folder
   (prompt: "Where in Google Drive should I save internal meeting files?")
4. After confirmation: go to Step 4

---

## Step 4 — Persist Confirmed Values

After any value is confirmed, do the following:

### 4a — Cache in Claude memory immediately

`memory_user_edits(command="add")` with:
- Asana GID: `asana_gid [AccountName]: [GID]`
- Account folder: `gdrive_folder [AccountName]: name=[Name] id=[ID] url=[URL]`
- Internal folder: `gdrive_folder_internal: name=[Name] id=[ID] url=[URL]`

### 4b — Update the xlsx in Drive

This fully automates the update — no manual paste needed.

```python
# /tmp/update_routing_table.py
import openpyxl, sys

path  = "/tmp/routing-table.xlsx"  # already downloaded in Step 2
sheet = sys.argv[1]   # "Accounts" or "Config"
col   = int(sys.argv[2])   # column index (1-based)
match = sys.argv[3]   # value to find in column A
value = sys.argv[4]   # new value to write

wb = openpyxl.load_workbook(path)
ws = wb[sheet]
for row in ws.iter_rows(min_row=2):
    if str(row[0].value or "").strip().lower() == match.lower():
        row[col - 1].value = value
        break
wb.save(path)
print("saved")
```

Run the update script for each changed field.

Then re-upload:
1. Re-encode as base64:
   ```bash
   python3 -c "import base64; print(base64.b64encode(open('/tmp/routing-table.xlsx','rb').read()).decode())"
   ```
2. Call `Google Drive:create_file`:
   - `title`: "Meeting Manager Config.xlsx"
   - `base64Content`: new base64
   - `contentMimeType`: `"application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"`
   - `disableConversionToGoogleType`: `true`
   - `parentId`: same folder as the previous version (get this from
     `Google Drive:get_file_metadata` on the old ID before uploading)
3. From the response, extract the new file ID.
4. Update memory: find the `meeting_manager_config_id` control with
   `memory_user_edits(command="view")`, then `memory_user_edits(command="replace",
   line_number=N, replacement="meeting_manager_config_id: [new ID]")`
5. Tell the user the routing table is updated and share the new Drive URL.
6. Note: the old file in Drive is now an orphan. The user can delete it; the skill
   will use the new ID from memory going forward.

---

## Available Google Drive Tools Reference

| Tool | Parameters of note |
|---|---|
| `Google Drive:create_file` | `base64Content`, `contentMimeType`, `disableConversionToGoogleType`, `parentId`, `title` |
| `Google Drive:download_file_content` | `fileId` only (no `exportMimeType` for xlsx/docx — returns raw base64) |
| `Google Drive:get_file_metadata` | `fileId` — use to get folder/parent before re-uploading |
| `Google Drive:read_file_content` | `fileId` — human-readable; use for GDocs/GSheets, not xlsx |
| `Google Drive:list_recent_files` | Lists recently accessed files; use to verify upload succeeded |
| `Google Drive:copy_file` | `fileId`, optional `parentId`, `title` |

There is **no** `search_files` tool and **no** `update_file` tool.
Updates are done by: download → modify locally → upload new file → update memory ID.
