---
name: kbi-letter
description: >
  Write an official letter as Secretary of Kickboxing Ireland (KBI). Use this skill any time Pierce
  asks for an official letter, formal reply, or secretary correspondence — phrased in any of these
  ways: "write a letter", "draft a letter", "official letter", "reply to this email as a letter",
  "use SKILL to reply to this email", "do me a letter", "write this up on letterhead", or anything
  similar. Pierce will usually provide the context, recipient, and key points inline in the prompt
  (often pasted emails or bullet-point notes). The skill fills the official KBI letterhead template,
  saves the finished .docx to the correct year folder in Google Drive Official Correspondence, and
  returns a link. This skill is distinct from the minutes/agenda skills — those are for meetings.
---

# KBI Official Letter Skill

This skill turns Pierce's notes, a pasted email, or a short brief into a finished official letter
on KBI letterhead — filed in the right place in Google Drive, ready to send.

---

## What the user will usually give you

Pierce normally puts most of the information directly in the prompt. Typical triggers include:

- "Use SKILL to reply to this email: ..." (email pasted below)
- "Draft a letter to X about Y" (with bullet points for the key content)
- "Write up a formal letter declining this request"

From the prompt, extract:

1. **Recipient details** — name, title, organisation, address
2. **What the letter is about** — purpose, tone, key points to make, any decisions, deadlines, or next steps
3. **Date** — default to today's date unless Pierce specifies otherwise
4. **Tone** — inferred from the situation (formal by default; firmer if it's a complaint response, warmer if it's a thank-you)

**Only ask Pierce for more info if something essential is missing** (usually the recipient's address
or name). Do not ask routine questions — the skill should make sensible judgements from the prompt.

---

## Hardcoded paths (Google Drive — host filesystem)

```
BASE       = /Users/piercemcgeough/Library/CloudStorage/GoogleDrive-secretary@kickboxingireland.ie/Shared drives/Kickboxing Ireland/10 - Official Correspondence

TEMPLATE   = {BASE}/Official Letterhead With Address.docx
OUTPUT_DIR = {BASE}/{YYYY}/
```

Replace `{YYYY}` with the 4-digit year from the letter date. If the folder for that year does not
exist yet (e.g. first letter of 2027), create it.

**Output filename:** `{YYYYMMDD} Letter to {Recipient} - {Short Subject}.docx`

Keep the subject short (3–6 words). Strip punctuation that would be invalid in a filename
(`/`, `:`, `?`, etc.).

Example: `20260422 Letter to John Murphy - Grading Seminar Dates.docx`

---

## Docx skill scripts location

Use whichever path is available in the current session:

```
{SKILL_DIR}/../docx/scripts/office/      # preferred — same skills folder
/mnt/skills/public/docx/scripts/office/  # fallback if public skills are mounted
```

Use `unpack.py` and `pack.py` from this directory. They handle XML pretty-printing, run merging,
and validation automatically.

---

## Template structure (already inspected — do not re-inspect unless it has changed)

The letterhead is a single-page template with a pre-built header (logo + KBI address) and a fixed
footer (signature block with scanned signature + Pierce's details). The body of `document.xml`
contains the following editable placeholders, in order:

| Placeholder                | Replace with                                                     |
|----------------------------|------------------------------------------------------------------|
| `[Date]`                   | Letter date, formatted e.g. `22nd April 2026`                    |
| `[Recipient Name]`         | Recipient's name                                                 |
| `[Recipient Title]`        | Recipient's title/role (delete paragraph if none)                |
| `[Recipient Organization]` | Recipient's organisation (delete paragraph if none)              |
| `[Recipient Address]`      | Recipient address — one paragraph per line                       |
| `Dear [Recipient],`        | Salutation, e.g. `Dear John,` or `Dear Mr Murphy,`               |
| Lorem ipsum body paragraph | The body of the letter — replace with one or more paragraphs     |

**Do NOT touch** the closing (`Yours sincerely,`), the signature image, or the signature block
(`Pierce McGeough / Secretary / Kickboxing Ireland / secretary@kickboxingireland.ie / ...`).
These are already in the template.

---

## Step-by-step Workflow

### Step 1 — Parse the date

From the letter date (default: today), derive:
- `YYYYMMDD` — e.g. `20260422` (used in filename)
- Long form — e.g. `22nd April 2026` (used in the letter body)
- `YYYY` — e.g. `2026` (year folder)

### Step 2 — Mount the Official Correspondence directory

Request access (skip if already mounted in this session):

```
mcp__cowork__request_cowork_directory:
  path: /Users/piercemcgeough/Library/CloudStorage/GoogleDrive-secretary@kickboxingireland.ie/Shared drives/Kickboxing Ireland/10 - Official Correspondence
```

Once mounted, the VM path will be reported back (e.g.
`/sessions/.../mnt/10 - Official Correspondence`). Use that VM path for all file operations.

### Step 3 — Draft the letter content

Before touching the template, draft the full letter body in plain text. The body should:

- Open with a clear statement of purpose (e.g. "I am writing on behalf of Kickboxing Ireland regarding ...")
- Use short, formal paragraphs — one idea per paragraph
- Use British/Irish English spelling (`organisation`, `recognise`, `behaviour`)
- Avoid contractions (`do not` not `don't`)
- Avoid bullet points unless the content genuinely needs them — formal letters read best as prose
- Close with a forward-looking sentence (e.g. "I look forward to your response in due course." or
  "Please do not hesitate to contact me if you require any further information.")

Do NOT write a closing like "Yours sincerely" — that's already in the template.

Consider starting the body with a short "Re: ..." line if the letter has a clear subject. If used,
make it its own paragraph in bold — see the XML notes below.

### Step 4 — Copy and unpack the template

```bash
cp "{VM_PATH}/Official Letterhead With Address.docx" \
   /tmp/kbi_letter_{YYYYMMDD}.docx

python3 {DOCX_SCRIPTS}/unpack.py \
  /tmp/kbi_letter_{YYYYMMDD}.docx \
  /tmp/kbi_letter_unpacked_{YYYYMMDD}/
```

### Step 5 — Edit `word/document.xml`

Make these replacements in order:

#### 5a. Date
Replace the run text `[Date]` with the long-form date (e.g. `22nd April 2026`).

#### 5b. Recipient block
Replace `[Recipient Name]`, `[Recipient Title]`, `[Recipient Organization]`, and
`[Recipient Address]` with the appropriate text.

- If the recipient has no separate title, delete the whole `[Recipient Title]` paragraph.
- If there's no organisation, delete the `[Recipient Organization]` paragraph.
- For a multi-line address, the `[Recipient Address]` paragraph should be **replaced with multiple
  paragraphs** — one per address line — all sharing the same styling (`<w:sz w:val="21"/>`,
  `<w:spacing w:after="60"/>` except the last line which keeps `<w:spacing w:after="240"/>`).

#### 5c. Salutation
Replace `Dear [Recipient],` with the correct salutation. Rules:
- First-name basis (`Dear John,`) if Pierce clearly knows them or the original email is on first-name terms
- Otherwise `Dear Mr/Ms {Surname},`
- For unknown recipients or generic bodies, `Dear Sir/Madam,`

#### 5d. Body
Replace the Lorem ipsum paragraph with the drafted body. Structure the XML like this for each
paragraph:

```xml
<w:p>
  <w:pPr>
    <w:spacing w:after="240"/>
    <w:rPr><w:sz w:val="21"/><w:szCs w:val="21"/></w:rPr>
  </w:pPr>
  <w:r>
    <w:rPr><w:sz w:val="21"/><w:szCs w:val="21"/></w:rPr>
    <w:t>Paragraph text here.</w:t>
  </w:r>
</w:p>
```

For a bold "Re:" line, add `<w:b/>` to the `<w:rPr>`:

```xml
<w:r>
  <w:rPr><w:b/><w:sz w:val="21"/><w:szCs w:val="21"/></w:rPr>
  <w:t>Re: Grading Seminar 2026</w:t>
</w:r>
```

**XML rules:**
- Every `<w:t>` with leading/trailing spaces needs `xml:space="preserve"`
- New paragraphs don't need `w14:paraId` or `w:rsidR` — omit them
- Special characters: `&#x2013;` (en dash), `&#x2019;` (apostrophe), `&amp;` (ampersand)

#### 5e. Do NOT modify
Leave these exactly as they are:
- The `Yours sincerely,` paragraph
- The signature drawing (`<w:drawing>` block)
- The Pierce McGeough signature block paragraph

### Step 6 — Repack and validate

```bash
python3 {DOCX_SCRIPTS}/pack.py \
  /tmp/kbi_letter_unpacked_{YYYYMMDD}/ \
  /tmp/kbi_letter_final_{YYYYMMDD}.docx \
  --original /tmp/kbi_letter_{YYYYMMDD}.docx
```

If validation fails, read the error, fix the XML, and try again.

### Step 7 — Save to the output location

```bash
mkdir -p "{VM_PATH}/{YYYY}/"
cp /tmp/kbi_letter_final_{YYYYMMDD}.docx \
   "{VM_PATH}/{YYYY}/{YYYYMMDD} Letter to {Recipient} - {Short Subject}.docx"
```

### Step 8 — Present the result

Share the final file with Pierce using `mcp__cowork__present_files` and a `computer://` link.
Briefly confirm:
- Recipient
- Subject/purpose in one line
- Where it was saved

Keep the message short — Pierce will open the file to review it.

---

## Tone guidance

Default register is **formal but warm**. Kickboxing Ireland is a volunteer-run national governing
body; letters go to clubs, coaches, parents, government bodies (Sport Ireland, Irish Martial Arts
Commission), and international federations (WAKO). Match the register to the audience:

- **To clubs / members** — formal but approachable; first names acceptable if the relationship is established.
- **To Sport Ireland / government / legal matters** — strictly formal; use titles and surnames; be precise and factual.
- **To WAKO and international bodies** — formal; assume English is not the reader's first language and keep sentences short.
- **Disciplinary / complaint responses** — firm, factual, unemotional; stick to the facts, cite relevant rules, avoid adjectives.
- **Thank-you / congratulations** — warmer, but still on letterhead and still structured as a proper letter.

Never sign off with anything other than what's already in the template.

---

## Year changes

When the letter year changes (e.g. first letter of 2027):
- The output folder changes to `.../10 - Official Correspondence/2027/` — create it if it doesn't exist
- Everything else stays the same
