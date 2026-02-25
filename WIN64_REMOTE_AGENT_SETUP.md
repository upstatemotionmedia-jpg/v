# SAFE Zero-Cost Remote Agent Setup (Windows 64-bit, GUI-first)
## Assumptions (so you can start now)
- You will only work inside `C:\UMM_AI`.
- You will run flows manually from Power Automate Desktop (PAD).
- You will use local AI UI (Open WebUI + Ollama) and upload files manually.
- You want Review steps before destructive actions.

## 1) Folder structure + architecture (build this first)
### Final structure
- `C:\UMM_AI\INBOX`
- `C:\UMM_AI\CLIENTS`
- `C:\UMM_AI\CLIENTS\_Incoming`
- `C:\UMM_AI\PROJECTS`
- `C:\UMM_AI\PROJECTS\_Incoming\Video`
- `C:\UMM_AI\PROJECTS\_Incoming\Photo`
- `C:\UMM_AI\FINANCE`
- `C:\UMM_AI\FINANCE\_Incoming`
- `C:\UMM_AI\REVIEW`
- `C:\UMM_AI\REVIEW\Unknown`
- `C:\UMM_AI\REVIEW\Duplicates`
- `C:\UMM_AI\ARCHIVE`
- `C:\UMM_AI\LOGS`
- `C:\UMM_AI\TEMPLATES`
- `C:\UMM_AI\TEMP`

### Flow architecture (safe-by-default)
- Flow A: Auto-Sort INBOX
  - Reads `INBOX` only.
  - Classifies by extension + filename keywords.
  - Copies to destination first, then moves original.
  - Unknowns go to `REVIEW\Unknown`.
  - Logs every step to `LOGS`.
- Flow B: Rename Standardizer
  - Prompts simple fields (Client/Project/Asset type).
  - Renames with sequence format.
  - Checks collision; increments index.
  - Moves to `PROJECTS\Client\Project\Raw`.
  - Logs every action.
- Flow C: Duplicate / Near-Duplicate Finder
  - Scans `INBOX` + `_Incoming` folders.
  - Compares filename + size (exact duplicate rule).
  - Optional hash step if PAD function available.
  - Sends duplicates to `REVIEW\Duplicates` after copy backup.
  - Writes summary report in `LOGS`.
- Optional Flow D: Daily Reset
  - Archives old incoming files (>30 days).
  - Adds daily log header.
  - Cleans only `C:\UMM_AI\TEMP`.

## 2) Install + first-time setup (exact click path)
### Install Power Automate Desktop (free)
- Open browser.
- Go to Microsoft page: `powerautomate.microsoft.com`.
- Click **Products** -> **Power Automate Desktop**.
- Click **Download**.
- Run installer.
- Accept defaults.
- Sign in with Microsoft account (free works).
- Launch **Power Automate** desktop app.

### Create root folder safely (no system folders touched)
- Open File Explorer.
- In address bar type `C:\` and Enter.
- Right-click blank area -> **New** -> **Folder** -> name `UMM_AI`.
- Open `C:\UMM_AI`.
- Create subfolders:
  - `INBOX`, `CLIENTS`, `PROJECTS`, `FINANCE`, `REVIEW`, `ARCHIVE`, `LOGS`, `TEMPLATES`, `TEMP`.
- Create second-level folders:
  - `CLIENTS\_Incoming`
  - `PROJECTS\_Incoming\Video`
  - `PROJECTS\_Incoming\Photo`
  - `FINANCE\_Incoming`
  - `REVIEW\Unknown`
  - `REVIEW\Duplicates`

## 3) Build Flow A: Auto-Sort INBOX (manual run)
### A. Create flow shell
- Open PAD.
- Click **+ New flow**.
- Name: `Auto-Sort INBOX`.
- Click **Create**.

### B. Variables you add first
- Action: **Set variable**
  - `%Root% = C:\UMM_AI`
- **Set variable** `%Inbox% = %Root%\INBOX`
- **Set variable** `%LogFile% = %Root%\LOGS\autosort_%CurrentDateTime:yyyy-MM-dd%.log`

### C. Start log header
- Action: **Get current date and time** -> `%Now%`
- Action: **Write text to file**
  - File: `%LogFile%`
  - Text: `=== Auto-Sort Start %Now% ===`
  - If exists: **Append**

### D. Loop files in INBOX
- Action: **Get files in folder**
  - Folder: `%Inbox%`
  - Include subfolders: No
  - Store in `%Files%`
- Action: **For each** `%CurrentFile%` in `%Files%`

### E. Inside loop: classify
- Action: **Get file path part**
  - Path: `%CurrentFile%`
  - Part: Extension -> `%Ext%`
- Action: **Convert text to lowercase** `%Ext%` -> `%ExtLower%`
- Action: **Get file path part**
  - Part: Filename -> `%NameOnly%`
- Action: **Convert text to lowercase** `%NameOnly%` -> `%NameLower%`
- Action: **Set variable** `%Destination% = %Root%\REVIEW\Unknown`

### F. Routing logic (If / Else If)
- If `%ExtLower%` is `.mp4` OR `.mov` OR `.mkv` OR `.avi`
  - `%Destination% = %Root%\PROJECTS\_Incoming\Video`
- Else if `%ExtLower%` is `.jpg` OR `.jpeg` OR `.png` OR `.heic` OR `.gif`
  - `%Destination% = %Root%\PROJECTS\_Incoming\Photo`
- Else if `%ExtLower%` is `.pdf` OR `.doc` OR `.docx` OR `.xls` OR `.xlsx` OR `.txt`
  - Nested If `%NameLower%` contains `invoice` OR `receipt` OR `contract`
    - If contains `invoice` OR `receipt` -> `%Destination% = %Root%\FINANCE\_Incoming`
    - Else (contract) -> `%Destination% = %Root%\CLIENTS\_Incoming`
  - Else -> `%Destination% = %Root%\CLIENTS\_Incoming`
- Else
  - keep `%Destination% = %Root%\REVIEW\Unknown`

### G. Safe move pattern (copy first)
- Action: **Copy file(s)**
  - Source: `%CurrentFile%`
  - Destination: `%Destination%`
  - If exists: **Do nothing** (safe)
- Action: **Display message**
  - Title: `Review Move`
  - Message: `Copied %CurrentFile% to %Destination%. Click OK to move original.`
- Action: **Move file(s)**
  - Source: `%CurrentFile%`
  - Destination: `%Destination%`
  - If exists: **Rename**
- Action: **Write text to file** Append:
  - `%Now% | %CurrentFile% -> %Destination%`

### H. End log
- After loop, write:
  - `=== Auto-Sort End ===`

### I. Save and test
- Click **Save**.
- Click **Run**.

## 4) Build Flow B: Rename Standardizer (manual run + simple form)
### A. Create flow
- PAD -> **+ New flow** -> `Rename Standardizer` -> **Create**.

### B. Prompt form fields
- Action: **Display input dialog** (or **Display custom form** if available)
  - Prompt 1: ClientName
  - Prompt 2: ProjectName
  - Prompt 3: AssetType (Video/Photo/Doc)
- Action: **Get current date and time** -> `%Now%`
- Action: **Convert datetime to text** format `yyyy-MM-dd` -> `%DateText%`

### C. Load INBOX files
- Set `%Inbox% = C:\UMM_AI\INBOX`
- Get files in folder `%Inbox%` -> `%Files%`
- Set `%Counter% = 1`

### D. Ensure destination folder exists
- Set `%DestFolder% = C:\UMM_AI\PROJECTS\%ClientName%\%ProjectName%\Raw`
- Action: **If folder exists** `%DestFolder%`
  - If No -> **Create folder** `%DestFolder%`

### E. Loop + rename format
- For each `%CurrentFile%` in `%Files%`
- Get extension `%Ext%`
- Convert `%Counter%` to 3-digit text (`001`,`002`,`003`) -> `%Seq%`
  - Use **Format number** action with `000`
- Set `%BaseName% = %DateText%__%ClientName%__%ProjectName%__%AssetType%__%Seq%`
- Set `%NewName% = %BaseName%%Ext%`

### F. Collision handling
- Set `%TargetPath% = %DestFolder%\%NewName%`
- While file exists `%TargetPath%`
  - Increment `%Counter%`
  - Rebuild `%Seq%`, `%BaseName%`, `%NewName%`, `%TargetPath%`
- Copy `%CurrentFile%` -> `%DestFolder%` (backup-first)
- Display message: `About to rename+move %CurrentFile% to %NewName%. Continue?`
- Move/Rename file to `%TargetPath%`
- Log append to `C:\UMM_AI\LOGS\rename_%DateText%.log`:
  - `%CurrentFile% => %TargetPath%`
- Increment `%Counter%`

### G. Save and run
- Save.
- Run manually when needed.

## 5) Build Flow C: Duplicate / Near-Duplicate Finder
### A. Create flow
- New flow -> `Duplicate Finder`.

### B. Define scan folders
- `%Root% = C:\UMM_AI`
- Build list variable `%ScanFolders%` with:
  - `%Root%\INBOX`
  - `%Root%\PROJECTS\_Incoming\Video`
  - `%Root%\PROJECTS\_Incoming\Photo`
  - `%Root%\FINANCE\_Incoming`
  - `%Root%\CLIENTS\_Incoming`

### C. Prepare outputs
- `%DupFolder% = %Root%\REVIEW\Duplicates`
- `%Summary% = %Root%\LOGS\duplicates_%CurrentDateTime:yyyy-MM-dd_HH-mm%.txt`
- Write summary header line.

### D. Build comparison key: filename + size
- Create dictionary `%Seen%` (key-value).
- For each folder in `%ScanFolders%`:
  - Get files in folder.
  - For each file:
    - Get filename with extension -> `%FileName%`
    - Get file size -> `%FileSize%`
    - Key `%Key% = %FileName%|%FileSize%`
    - If `%Seen%` contains `%Key%`:
      - Copy current file to `%DupFolder%`.
      - Display Review message before move.
      - Move current file to `%DupFolder%` (rename if needed).
      - Append summary line:
        - `DUPLICATE | %CurrentFile% | original=%Seen[%Key%]%`
    - Else:
      - Add `%Key% -> %CurrentFile%` to `%Seen%`

### E. Optional hash match (only if PAD has it in your version)
- If action **Calculate hash** is available:
  - Use SHA256 on each file.
  - Use hash key instead of filename+size for stricter match.
- If not available:
  - Keep filename+size method only.

### F. Final report
- Append totals:
  - scanned count
  - duplicate count
  - destination folder
- Save flow.

## 6) Export packages OR recreate checklist
### Export (preferred)
- PAD home screen.
- Select flow -> click **Export**.
- Include dependencies if prompted.
- Save zip to `C:\UMM_AI\TEMPLATES\FlowExports\`.
- Repeat for A/B/C.

### Recreate checklist (if export unavailable)
- Confirm all root path variables use `C:\UMM_AI`.
- Confirm all flows are manual run.
- Confirm every destructive step has message review popup.
- Confirm copy happens before move/delete.
- Confirm log path writes to `C:\UMM_AI\LOGS`.

## 7) AI Review workflow (local-first, no full-disk access)
### Install local AI stack (GUI-friendly)
- Install Ollama:
  - Open browser -> `ollama.com` -> Download Windows installer -> run.
- Install Open WebUI (Windows easiest path)
  - If Docker Desktop already installed:
    - Install Open WebUI container from official instructions page.
  - If Docker not installed and you want simpler:
    - Use any local UI that connects to Ollama and supports file upload.
- Keep data handling safe:
  - Only upload selected files manually from `C:\UMM_AI`.
  - Never grant broad folder sync permissions.

### AI usage workflow
- Step 1: Collect files in `C:\UMM_AI\REVIEW\AI_DROP` (create this folder).
- Step 2: Open AI UI in browser.
- Step 3: Start new chat.
- Step 4: Attach only required files.
- Step 5: Paste one of prompts below.
- Step 6: Copy AI output into `C:\UMM_AI\LOGS\AI_Notes_yyyy-mm-dd.txt`.

## 8) Copy/paste prompts
### Prompt 1: Invoice scan + unpaid list + next action
```text
You are my bookkeeping assistant.
Task:
1) Read all attached files.
2) Identify invoices, receipts, and payment confirmations.
3) Build a table with columns:
- Client
- Invoice Number
- Invoice Date
- Due Date
- Amount
- Paid? (Yes/No/Unknown)
- Evidence (file name + snippet)
4) Return a section: UNPAID INVOICES ONLY (sorted by due date oldest first).
5) Return a section: NEXT BEST ACTION for each unpaid invoice in 1 sentence.
Rules:
- If data is missing, write "Unknown".
- Do not invent values.
- Keep output concise and actionable.
```

### Prompt 2: Client follow-up generator (short, payment-first)
```text
You are generating payment follow-up messages.
Use the unpaid invoice table I provide.
Create 3 message versions per client:
1) Friendly
2) Firm
3) Final reminder
Constraints:
- 60 to 110 words each
- Clear amount + invoice number + due date
- Include one direct payment call-to-action
- Professional, not aggressive
Output format:
Client Name
- Friendly:
- Firm:
- Final reminder:
```

### Prompt 3: Project status summary from notes
```text
You are my project coordinator.
Read all attached project notes and produce:
1) Project snapshot (5 bullets max)
2) Completed items
3) In-progress items
4) Blockers / risks
5) Next 3 priorities for this week
6) One client update message (short, clear)
Rules:
- Cite the file name for each blocker and priority.
- If uncertain, label as "Needs confirmation".
- Keep language simple and practical.
```

## 9) Optional upgrade: Flow D “Daily Reset” (manual)
### What it does
- Archive finished items older than 30 days from all `_Incoming` folders.
- Create daily log header file.
- Delete only contents of `C:\UMM_AI\TEMP` after review popup.

### Build steps (short)
- New flow `Daily Reset`.
- Set `%ArchiveDate% = Today - 30 days`.
- For each `_Incoming` folder:
  - Get files.
  - If file modified date older than `%ArchiveDate%`:
    - Copy to `C:\UMM_AI\ARCHIVE\yyyy-MM\`
    - Review popup.
    - Move to archive.
    - Log action.
- Write `daily_yyyy-mm-dd.log` header in LOGS.
- Get files in `C:\UMM_AI\TEMP`.
- Popup confirmation.
- Delete files if confirmed.

## 10) Acceptance test plan (run this yourself)
### Test file set (place into `C:\UMM_AI\INBOX`)
- `invoice_Acme_1001.pdf`
- `receipt_lensmart_2025-01.pdf`
- `contract_BetaLLC.docx`
- `wedding_shoot.mov`
- `portrait_001.jpg`
- `notes_clientcall.txt`
- `unknown_payload.xyz`
- Duplicate pair:
  - `dup_test.pdf` (copy twice with same size and name in two incoming folders)

### Expected Flow A routing
- `invoice_*`, `receipt_*` -> `FINANCE\_Incoming`
- `contract_*` -> `CLIENTS\_Incoming`
- `.mov` -> `PROJECTS\_Incoming\Video`
- `.jpg` -> `PROJECTS\_Incoming\Photo`
- `.txt` -> `CLIENTS\_Incoming`
- `.xyz` -> `REVIEW\Unknown`

### Expected Flow B rename examples
- Inputs:
  - ClientName `Acme`
  - ProjectName `Q1Campaign`
  - AssetType `Video`
- Output examples:
  - `2026-02-23__Acme__Q1Campaign__Video__001.mov`
  - `2026-02-23__Acme__Q1Campaign__Video__002.mov`
- Destination:
  - `C:\UMM_AI\PROJECTS\Acme\Q1Campaign\Raw`

### Expected Flow C outputs
- Duplicate moved to `C:\UMM_AI\REVIEW\Duplicates`
- Summary file in `C:\UMM_AI\LOGS\duplicates_*.txt`
- Summary includes scanned count + duplicates count + source/original mapping

### Expected logs
- `autosort_yyyy-mm-dd.log` lines like:
  - `timestamp | source -> destination`
- `rename_yyyy-mm-dd.log` lines like:
  - `oldpath => newpath`
- `duplicates_yyyy-mm-dd_hh-mm.txt` lines like:
  - `DUPLICATE | file | original=...`

### Failure modes + recovery
- Destination file exists:
  - Flow should rename automatically.
  - Recovery: rerun flow; check log for final filename.
- File locked/open:
  - Move may fail.
  - Recovery: close app using file, rerun.
- Wrong classification:
  - Recovery: move from `REVIEW\Unknown` manually, add new keyword rule in Flow A.
- Too many confirmations:
  - Recovery: keep review popup only for move/delete, not copy.
- Duplicate false positive with filename+size:
  - Recovery: compare manually or enable hash action if available.

## 11) One-screen quick start checklist
- [ ] Install PAD
- [ ] Create `C:\UMM_AI` structure
- [ ] Build Flow A and test with 5 files
- [ ] Build Flow B and test naming sequence
- [ ] Build Flow C and test duplicate detection
- [ ] Export flows to `TEMPLATES\FlowExports`
- [ ] Install local AI UI + Ollama
- [ ] Save prompts in `C:\UMM_AI\TEMPLATES\prompts.txt`
- [ ] Run acceptance tests and review logs
