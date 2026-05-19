---
name: kbi-exec-minutes
description: >
  Generate official KBI (Kickboxing Ireland) Executive Committee meeting minutes from an AI meeting summary. Use this skill any time Pierce mentions writing up, drafting, or creating minutes for a KBI executive meeting — even if the exact words differ (e.g. "do the minutes", "write up the exec meeting", "process the meeting notes"). Takes a meeting date, AI-generated summary, and any extra notes. Automatically looks up attendance from the KBI Planning Book, fills the official Word template, saves to the correct year folder in Google Drive, and writes all action points back into the Planning Book. This skill knows all the file paths and handles year changes automatically — no need to specify where things go.
---

# KBI Executive Meeting Minutes Skill

This skill takes an AI-generated meeting summary and turns it into the official KBI Executive Committee minutes document, filed in the right place, with attendance pulled from the Planning Book and action points logged automatically.

## What to collect from the user before starting

You need all three of these — ask if any are missing:

1. **Meeting date** — e.g. "21st March 2026" or "21/03/2026"
2. **AI-generated minutes** — the full markdown output from the meeting AI tool (summary + next steps)
3. **Any notes** — things like "previous minutes were skipped", "meeting ran to 23:30", specific agenda items that weren't captured, corrections

## Hardcoded paths (Google Drive — host filesystem)

These are fixed and never need to change unless the folder structure in Google Drive changes.

```
BASE = /Users/piercemcgeough/Library/CloudStorage/GoogleDrive-secretary@kickboxingireland.ie/Shared drives/Kickboxing Ireland/00 - Executive

TEMPLATE   = {BASE}/Executive Meetings/Kickboxing Ireland Exec Meeting yyyymmdd.docx
OUTPUT_DIR = {BASE}/Executive Meetings/{YYYY}/
PLAN_BOOK  = {BASE}/Planning/KBI Planning Book {YYYY}.xlsx
```

Replace `{YYYY}` with the 4-digit year extracted from the meeting date.

**Output filename:** `Kickboxing Ireland Exec Meeting {YYYYMMDD}.docx`

## Docx skill scripts location (always available)

```
DOCX_SCRIPTS = /sessions/sleepy-gracious-gates/mnt/.claude/skills/docx/scripts/office/
```

Use `unpack.py` and `pack.py` from this directory. They handle XML pretty-printing, run merging, and validation automatically.

---

## Step-by-step Workflow

### Step 1 — Parse the date

From the meeting date, derive:
- `YYYYMMDD` — e.g. `20260321` (used in the filename)
- `DD/MM/YYYY` — e.g. `21/03/2026` (shown in the document)
- `YYYY` — e.g. `2026` (year folder and planning book)
- `one_month_later` — default action point deadline (e.g. `21/04/2026`)

### Step 2 — Mount required directories

Request access to both directories (skip if already mounted in this session):

```
mcp__cowork__request_cowork_directory:
  path: /Users/piercemcgeough/Library/CloudStorage/GoogleDrive-secretary@kickboxingireland.ie/Shared drives/Kickboxing Ireland/00 - Executive/Executive Meetings

mcp__cowork__request_cowork_directory:
  path: /Users/piercemcgeough/Library/CloudStorage/GoogleDrive-secretary@kickboxingireland.ie/Shared drives/Kickboxing Ireland/00 - Executive/Planning
```

Once mounted, the VM paths will be reported back (e.g. `/sessions/.../mnt/Executive Meetings` and `/sessions/.../mnt/Planning`). Use those VM paths for all file operations.

### Step 3 — Look up attendance

Run the attendance script from the skill's own `scripts/` directory. Use the path reported when the skill was loaded:

```bash
python3 {SKILL_DIR}/scripts/lookup_attendance.py \
  "{VM_PATH}/Planning/KBI Planning Book {YYYY}.xlsx" \
  "{DD/MM/YYYY}"
```

The script prints JSON: `{"attendees": ["Roy Baker", ...], "apologies": ["Lisa Fox", ...]}`.

If the date column doesn't exist yet (meeting hasn't been logged in the book), ask Pierce to provide the attendance manually before continuing.

### Step 4 — Build the minutes document

#### 4a. Check for an existing draft and inventory its content (do this FIRST)

Before touching the master template, check whether a draft already exists in the output folder for this meeting date:

```
{VM_PATH}/Executive Meetings/{YYYY}/Kickboxing Ireland Exec Meeting {YYYYMMDD}.docx
```

**If a draft exists, use it as the base — not the blank template.** Pierce may have pre-created the file with correct agenda headings, and may also have typed partial notes or sentences under some of those headings. Both the headings and those notes must be preserved and used.

```bash
# Preferred: use Pierce's draft if it exists
if [ -f "{VM_PATH}/Executive Meetings/{YYYY}/Kickboxing Ireland Exec Meeting {YYYYMMDD}.docx" ]; then
  cp "{VM_PATH}/Executive Meetings/{YYYY}/Kickboxing Ireland Exec Meeting {YYYYMMDD}.docx" \
     /tmp/kbi_working_{YYYYMMDD}.docx
else
  # Fall back to the master template
  cp "{VM_PATH}/Executive Meetings/Kickboxing Ireland Exec Meeting yyyymmdd.docx" \
     /tmp/kbi_working_{YYYYMMDD}.docx
fi
```

#### 4b. Unpack

```bash
python {DOCX_SCRIPTS}/unpack.py \
  /tmp/kbi_working_{YYYYMMDD}.docx \
  /tmp/kbi_unpacked_{YYYYMMDD}/
```

**Immediately after unpacking, read the XML and build a content inventory** before writing anything. For each Heading1 section in the draft, note:

- The heading text (this is the authoritative agenda label — do not rename or remove it)
- Any existing body text already typed under that heading (keep this — it is Pierce's draft content and takes precedence)

Example inventory (internal working note, not shown to the user):
```
"All Ireland National Championships"
  → existing notes: "Venue confirmed as National Stadium. Capacity issue raised."
"Finance"
  → existing notes: (empty)
"AOB"
  → existing notes: "Dave raised online affiliation issue."
```

**Tracking orphaned draft content:** As you read through the draft, flag any text that is ambiguous — it exists in the document but you're unsure which heading it belongs to, or it doesn't appear to relate to anything in the AI summary. Collect these as a list to review with Pierce before finalising (see Step 4d).

#### 4c. Map AI minutes to headings, then edit the XML

The AI summary is informal and often non-linear — a single paragraph may touch on multiple agenda topics, or a topic may appear in the middle of a section that seems unrelated. **Do not process the AI minutes top-to-bottom and paste content sequentially.** Instead, read the entire AI summary first to understand the full picture, then work heading by heading, pulling in the relevant content from wherever it appears in the summary.

**Priority rule — Pierce's notes always come first:**  
If Pierce has typed notes under a heading, those notes are kept exactly as written — word for word, unchanged, in their original position. Do not reword, merge, or move them. After Pierce's notes, add a blank line, then insert the AI-generated content beneath. Pierce will manually edit the combined result as needed. If there are no existing notes under a heading, insert the AI content directly as normal.

For each heading in the draft:
1. Ask: "What does the AI summary say that belongs here?" Scan the whole AI text — relevant content may appear anywhere, not just in the section with a matching label.
2. If Pierce has existing notes under this heading: keep them verbatim at the top, then add the relevant AI content below them.
3. If there are no existing notes: write the AI-sourced content as the section body.

If the AI summary contains content that clearly belongs under a heading but is written as part of a larger paragraph covering multiple topics, extract just the relevant portion and place it under the right heading. You are the editor — the AI summary is raw material, not a final structure to be copied.

Only add a new Heading1 section if the AI summary clearly covers a topic that has no matching heading in the draft at all.

**Header table (always):**
- Replace `DD/MM/YYYY` → formatted meeting date
- Fill the empty `<w:p>` in the Attendees row with a `<w:r>` containing the comma-separated attendee names
- Fill the empty `<w:p>` in the Apologies row with a `<w:r>` containing the comma-separated apology names
- Both use `<w:sz w:val="20"/>` and `<w:szCs w:val="20"/>` in `<w:rPr>`

**Passing of Previous Minutes section:**
- If minutes were passed: fill in "Proposer: [Name]" and "Second: [Name]" and keep "All Approved"
- If skipped (emergency meeting or noted in the user's notes): replace the Proposer/Second/All Approved paragraphs with a single paragraph stating it was not applicable and why

**Agenda sections:**
- Keep only sections that were actually on the agenda — remove any template placeholder headings that were not discussed
- For each real agenda item: use `<w:pStyle w:val="Heading1"/>` for the section heading, then add body paragraphs with `<w:sz w:val="20"/>` text
- For subsections within a topic, use `<w:pStyle w:val="Heading2"/>`

**AOB:**
- If nothing was raised: leave the heading but add a paragraph saying "No items raised."
- If items were raised: write them up as sub-sections or a prose paragraph

**XML rules to follow:**
- Every `<w:t>` with leading or trailing spaces needs `xml:space="preserve"`
- New paragraphs don't need `w14:paraId` or `w:rsidR` — omit them
- Special characters: use `&#x2013;` (en dash), `&#x2019;` (apostrophe), `&#x201C;`/`&#x201D;` (quotes), `&#x20AC;` (euro sign), `&amp;` (ampersand)

#### 4d. Review orphaned draft notes with Pierce (before repacking)

Before repacking, check whether you collected any orphaned notes — content that was in Pierce's draft but couldn't be confidently matched to any heading or to anything in the AI summary.

If you have orphaned notes, **stop and show them to Pierce** before finalising. Present them clearly:

> "I found the following notes in your draft that I wasn't sure where to place. Can you tell me which section they belong to, or confirm they can be removed?"
>
> - *[note 1]*
> - *[note 2]*

Wait for Pierce's response before continuing. Once he directs you, place the content accordingly and then proceed to repack.

If there are no orphaned notes, proceed directly to repacking.

#### 4e. Repack and validate

```bash
python {DOCX_SCRIPTS}/pack.py \
  /tmp/kbi_unpacked_{YYYYMMDD}/ \
  /tmp/kbi_final_{YYYYMMDD}.docx \
  --original /tmp/kbi_working_{YYYYMMDD}.docx
```

If validation fails, read the error, fix the XML, and try again.

#### 4f. Save to the output location

```bash
mkdir -p "{VM_PATH}/Executive Meetings/{YYYY}/"
cp /tmp/kbi_final_{YYYYMMDD}.docx \
   "{VM_PATH}/Executive Meetings/{YYYY}/Kickboxing Ireland Exec Meeting {YYYYMMDD}.docx"
```

### Step 5 — Extract and log action points

Read the AI minutes "Next steps" section carefully, and also scan the narrative for any actions attributed to named people. For each action point, build a record:

| Field    | Value |
|----------|-------|
| Who      | First name or short form matching the Planning Book (Roy, Dave, Pierce, Natalie, Lauren, Eanna, Vida, Teresa, Karl, Paul, Lauren/Jo, etc.) |
| Action   | Concise description of what needs to be done |
| Date     | The meeting date |
| Deadline | One calendar month after the meeting date, unless the minutes specify something sooner |
| Status   | "Pending" |

Run the action points script:

```bash
python3 {SKILL_DIR}/scripts/add_action_points.py \
  "{VM_PATH}/Planning/KBI Planning Book {YYYY}.xlsx" \
  '{JSON_ARRAY_OF_ACTION_POINTS}'
```

JSON format expected:
```json
[
  {"who": "Roy", "action": "Cancel the Waterford venue booking", "date": "21/03/2026", "deadline": "21/04/2026", "status": "Pending"},
  {"who": "Dave", "action": "Email event management plan to TUI Tallaght", "date": "21/03/2026", "deadline": "21/04/2026", "status": "Pending"}
]
```

### Step 6 — Present the result

Share the output file with Pierce using `mcp__cowork__present_files` and a computer:// link. Give a brief summary: how many attendees, how many apologies, how many action points logged, and a one-line note on any issues or assumptions made.

---

## Content Writing Guide

The AI summary is informal and detailed. Your job is to turn it into clean, formal minutes — the kind that would stand as an official record.

**Do:**
- Write in the past tense ("The group agreed...", "Roy noted...", "It was confirmed that...")
- Capture decisions and their rationale, not the back-and-forth
- Group related points under logical sub-headings using Heading2
- Be specific about who is responsible for what
- Use proper Irish names and organisation names (TUI Tallaght, WAKO, IMAC, RSportz, Sport NI, CLG etc.)

**Don't:**
- Transcribe conversation verbatim
- Include speculative or tentative points unless they resulted in an action
- Write "The team discussed..." for every sentence — vary it
- Invent details not present in the summary
- Trust the AI summary's structure — it frequently jumps between topics mid-paragraph. Read it as raw material and redistribute the content by heading, not by the order it appears.

**Typical KBI agenda structure (adapt as needed):**
```
Passing of Previous Minutes
Outstanding Actions (if reviewed at the meeting)
[Main item 1 — e.g. All Ireland National Championships]
  [Subsection — e.g. Venue]
  [Subsection — e.g. Logistics]
[Main item 2]
...
AOB
  [Sub-item 1]
  [Sub-item 2]
```

---

## Year changes

When the meeting year changes (e.g. first meeting of 2027):
- The output folder changes to `.../Executive Meetings/2027/` — create it if it doesn't exist
- The Planning Book changes to `KBI Planning Book 2027.xlsx` — if it doesn't exist yet, warn Pierce and ask whether to create it or use the 2026 book temporarily
- Everything else stays the same
