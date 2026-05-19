---
name: kbi-executive-minutes-prep
description: >
  Create the KBI Executive Committee meeting agenda file from a list of items provided by Pierce.
  Use this skill any time Pierce mentions preparing, creating, or setting up an agenda for a KBI
  executive meeting — even if the exact words differ (e.g. "prep the agenda", "set up the meeting
  file", "create the agenda for next week", "do the agenda prep"). Takes a meeting date and a list
  of agenda items (with optional sub-items), fills the official Word template, and saves the file
  to the correct year folder in Google Drive. This skill is distinct from the minutes skill — it
  creates the blank agenda file BEFORE the meeting; the minutes skill is used AFTER.
---

# KBI Executive Meeting Agenda Prep Skill

This skill takes a meeting date and a list of agenda items from Pierce and produces a correctly
named and structured agenda `.docx` file, saved to the right folder in Google Drive, ready to be
used as the base file for the minutes after the meeting.

---

## What to collect from Pierce before starting

Both of these must be in the same message — if either is missing, ask:

1. **Meeting date** — e.g. "21st April 2026" or "21/04/2026"
2. **Agenda items** — pasted as plain text in this format:

```
Item
Item
Item
    A) Sub Item
    B) Sub Item
Item
```

Top-level items become **Heading 1** sections. Sub-items (indented with A), B), etc.) become
**Heading 2** sections nested under the item immediately above them.

---

## Hardcoded paths (Google Drive — host filesystem)

```
BASE       = /Users/piercemcgeough/Library/CloudStorage/GoogleDrive-secretary@kickboxingireland.ie/Shared drives/Kickboxing Ireland/00 - Executive

TEMPLATE   = {BASE}/Executive Meetings/Kickboxing Ireland Exec Meeting yyyymmdd.docx
OUTPUT_DIR = {BASE}/Executive Meetings/{YYYY}/
```

Replace `{YYYY}` with the 4-digit year from the meeting date.

**Output filename:** `Kickboxing Ireland Exec Meeting {YYYYMMDD}.docx`

---

## Docx skill scripts location

```
DOCX_SCRIPTS = /mnt/skills/public/docx/scripts/office/
```

Use `unpack.py` and `pack.py` from this directory.

---

## Step-by-step Workflow

### Step 1 — Parse the date

From the meeting date, derive:
- `YYYYMMDD` — e.g. `20260421` (used in the output filename)
- `YYYY` — e.g. `2026` (year folder)

The date is used in the **filename only** — it does not need to appear inside the document body.

### Step 2 — Mount the Executive Meetings directory

Request access (skip if already mounted in this session):

```
mcp__cowork__request_cowork_directory:
  path: /Users/piercemcgeough/Library/CloudStorage/GoogleDrive-secretary@kickboxingireland.ie/Shared drives/Kickboxing Ireland/00 - Executive/Executive Meetings
```

Once mounted, the VM path will be reported back (e.g. `/sessions/.../mnt/Executive Meetings`).
Use that VM path for all file operations.

### Step 3 — Copy and unpack the template

```bash
cp "{VM_PATH}/Executive Meetings/Kickboxing Ireland Exec Meeting yyyymmdd.docx" \
   /tmp/kbi_agenda_{YYYYMMDD}.docx

python {DOCX_SCRIPTS}/unpack.py \
  /tmp/kbi_agenda_{YYYYMMDD}.docx \
  /tmp/kbi_agenda_unpacked_{YYYYMMDD}/
```

### Step 4 — Inspect the template XML

After unpacking, **read `word/document.xml`** to understand the template structure before
editing. Specifically note:

- What placeholders exist in the header table (e.g. a date cell, title row, or `yyyymmdd` text)
- The styles already applied (Heading1, Heading2, body text)
- Where agenda content should begin — typically after the header table

**Do not guess the structure — read the XML first.**

### Step 5 — Edit the XML

#### 5a. Clear any placeholder agenda content

Remove any example/placeholder heading paragraphs that exist in the template body below the
header table. Keep the header table and any fixed preamble (e.g. "Passing of Previous Minutes"
if it's a fixed section) intact.

#### 5b. Insert the agenda items

For each top-level item, add a paragraph with `<w:pStyle w:val="Heading1"/>`.
For each sub-item (A), B), etc.) under it, add a paragraph with `<w:pStyle w:val="Heading2"/>`.

Strip the A), B) prefix from sub-item text — just use the label text itself as the heading.

Example input:
```
Finance
    A) Budget Review
    B) Sponsorship Update
AOB
```

Example XML output:
```xml
<w:p>
  <w:pPr><w:pStyle w:val="Heading1"/></w:pPr>
  <w:r><w:t>Finance</w:t></w:r>
</w:p>
<w:p>
  <w:pPr><w:pStyle w:val="Heading2"/></w:pPr>
  <w:r><w:t>Budget Review</w:t></w:r>
</w:p>
<w:p>
  <w:pPr><w:pStyle w:val="Heading2"/></w:pPr>
  <w:r><w:t>Sponsorship Update</w:t></w:r>
</w:p>
<w:p>
  <w:pPr><w:pStyle w:val="Heading1"/></w:pPr>
  <w:r><w:t>AOB</w:t></w:r>
</w:p>
```

**XML rules:**
- Every `<w:t>` with leading or trailing spaces needs `xml:space="preserve"`
- New paragraphs don't need `w14:paraId` or `w:rsidR` — omit them
- Special characters: `&#x2013;` (en dash), `&#x2019;` (apostrophe), `&amp;` (ampersand)

### Step 6 — Repack and validate

```bash
python {DOCX_SCRIPTS}/pack.py \
  /tmp/kbi_agenda_unpacked_{YYYYMMDD}/ \
  /tmp/kbi_agenda_final_{YYYYMMDD}.docx \
  --original /tmp/kbi_agenda_{YYYYMMDD}.docx
```

If validation fails, read the error, fix the XML, and try again.

### Step 7 — Save to the output location

```bash
mkdir -p "{VM_PATH}/Executive Meetings/{YYYY}/"
cp /tmp/kbi_agenda_final_{YYYYMMDD}.docx \
   "{VM_PATH}/Executive Meetings/{YYYY}/Kickboxing Ireland Exec Meeting {YYYYMMDD}.docx"
```

### Step 8 — Present the result

Share the output file with Pierce using `mcp__cowork__present_files` and a `computer://` link.
Briefly confirm: the filename, where it was saved, and how many agenda items (and sub-items)
were included.

---

## Relationship to the Minutes Skill

This file — once saved — becomes the **base file** that the minutes skill (`kbi-exec-minutes`)
picks up after the meeting. The minutes skill already checks for a pre-existing file in the
output folder before falling back to the blank template. So as long as this skill saves to the
correct path, the two skills chain together automatically.

**No additional linking is needed.**

---

## Year changes

When the meeting year changes (e.g. first meeting of 2027):
- The output folder changes to `.../Executive Meetings/2027/` — create it if it doesn't exist
- Everything else stays the same
