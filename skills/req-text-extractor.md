# req-text-extractor — Requirement Document Text Extraction Skill

Apply these rules when extracting text content from requirement documents of any format.
Used by `agents/capability-1/req-ingestor.agent.md` to normalise all input files into plain text before analysis.

---

## Tier Classification

Classify each file by extension before extracting:

| Tier | Extension(s) | Strategy |
|------|-------------|----------|
| 1 | `.md`, `.txt`, `.eml` | Read directly with `Read` tool |
| 1 | `.pdf` | Read with `Read` tool, pages parameter |
| 1 | `.yaml`, `.yml`, `.json` | Read with `Read` tool → parse structure |
| 1 | `.xml`, `.bpmn` | Read with `Read` tool → parse XML |
| 1 | `.csv` | Read with `Read` tool → parse rows |
| 1 | `.png`, `.jpg`, `.jpeg` | Read with `Read` tool (multimodal) |
| 2 | `.docx` | Shell conversion → temp markdown |
| 2 | `.xlsx` | Shell conversion → CSV text |
| 3 | `.fig` | Unsupported — user action required |

---

## Tier 1 Extraction Rules

### `.md` / `.txt` / `.eml`
- Read file with `Read` tool
- Return full text as-is
- For `.eml`: extract Subject, From, Date as metadata; body is the raw text

### `.pdf`
- Read with `Read` tool
- If file > 10 pages: read in 10-page chunks (`pages: "1-10"`, `"11-20"`, etc.) and concatenate
- Strip repeated headers/footers (lines appearing identically on consecutive pages)
- Preserve section headings and numbered lists

### `.yaml` / `.yml`
- Read with `Read` tool
- Serialize the full YAML to a readable text block:
  - Top-level keys become section headings
  - Nested keys become bullet points with indentation
- Look for keys: `story`, `userStory`, `feature`, `acceptanceCriteria`, `epic`, `description`, `requirements`, `constraints`
- If found, extract those sections first; include full serialized YAML below as "Full Document"

### `.json`
- Read with `Read` tool
- Pretty-print the JSON
- Look for keys: `title`, `description`, `requirements`, `features`, `capabilities`, `acceptanceCriteria`, `actors`, `roles`
- Serialize to readable key: value text block; nested arrays become bulleted lists

### `.xml` / `.bpmn`
- Read with `Read` tool
- Extract text content from element bodies (strip XML tags for readability)
- Preserve element names as section labels
- For `.bpmn`: extract `<process>`, `<task>`, `<gateway>`, `<event>` elements as workflow steps
  - Format as: `[TaskType] ElementName: documentation text`

### `.csv`
- Read with `Read` tool
- Parse as tabular data: first row = headers
- Format each row as: `Row N: header1=value1 | header2=value2 | ...`
- If row count > 100: extract first 100 rows + summary count warning

### `.png` / `.jpg` / `.jpeg`
- Read with `Read` tool (Claude is multimodal — image is processed visually)
- Describe visual content in structured text:
  - For whiteboard/diagram photos: describe workflow, boxes, arrows, labels
  - For UI mockups: describe screens, components, user interactions visible
  - For scanned forms: transcribe field labels and any visible content
  - For charts/graphs: describe data relationships and labels
- Prepend description with: `[IMAGE ANALYSIS - <filename>]`

---

## Tier 2 Extraction Rules

### `.docx` — Conversion Required

**Step 1**: Check for `pandoc`:
```bash
pandoc --version
```

If available:
```bash
pandoc --to markdown "<input_file>" -o "<tmp_dir>/<slug>.md"
```
Then read the generated `.md` file with `Read` tool.

**Step 2 (fallback)**: If pandoc not found, check for Python:
```bash
python --version || python3 --version
```

If available:
```bash
python -c "import docx2txt; print(docx2txt.process('<input_file>'))"
```

If `docx2txt` not installed:
```bash
pip install docx2txt -q && python -c "import docx2txt; print(docx2txt.process('<input_file>'))"
```

**Step 3 (fallback)**: If Python also unavailable → Tier 3 (inform user).

**Output**: Full markdown text of the document.

### `.xlsx` — Conversion Required

**Step 1**: Check for Python + pandas:
```bash
python -c "import pandas; print('ok')"
```

If available:
```bash
python -c "
import pandas as pd
xl = pd.ExcelFile('<input_file>')
for sheet in xl.sheet_names:
    df = xl.parse(sheet)
    print(f'=== Sheet: {sheet} ===')
    print(df.to_csv(index=False))
"
```

**Step 2 (fallback)**: If pandas not installed:
```bash
pip install pandas openpyxl -q && python -c "..."
```

**Step 3 (fallback)**: If Python unavailable → Tier 3.

**Output**: CSV text for each sheet, prefixed with `=== Sheet: <name> ===`.

---

## Tier 3 — Unsupported Formats

When a file cannot be extracted, report:

```
⚠️  UNSUPPORTED FORMAT: <filename> (<extension>)

This file format cannot be processed automatically.

ACTION REQUIRED — choose one of the following:
  Option A: Export as PDF and re-run:
            /sqe-kit:gen-user-stories --req <path-to-pdf>
  Option B: Export as Markdown/plain text and re-run:
            /sqe-kit:gen-user-stories --req <path-to-md>
  Option C: Copy the relevant content and pass it inline.

Skipping this file and continuing with remaining inputs.
```

Tier 3 formats:
- `.fig` — Figma files: export as PDF or PNG from Figma
- `.docx` with no pandoc/python → export from Word as PDF
- `.xlsx` with no Python → export from Excel as CSV

---

## Multi-File Inputs

When `--req` receives a comma-separated list or directory path:

```
1. Split by comma OR glob directory for: *.md, *.txt, *.pdf, *.docx, *.xlsx, *.csv,
   *.yaml, *.yml, *.json, *.xml, *.bpmn, *.eml, *.png, *.jpg, *.jpeg
2. Sort files alphabetically
3. Log discovered files: "Found N files: [file1, file2, ...]"
4. Process each file independently using rules above
5. Combine extracted texts with separator:
   === FILE: <filename> ===
   <extracted text>
   === END: <filename> ===
```

---

## Extraction Output Contract

For each file, produce:

```json
{
  "file": "path/to/file.ext",
  "format": "pdf",
  "tier": 1,
  "extractionMethod": "Read tool with pages",
  "rawText": "... full extracted text ...",
  "pageCount": 12,
  "warnings": ["Large document — 12 pages extracted", "..."]
}
```

All extraction results are passed to `agents/capability-1/req-ingestor.agent.md` as an array.
