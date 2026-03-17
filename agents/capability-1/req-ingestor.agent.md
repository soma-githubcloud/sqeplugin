# req-ingestor — Requirement Document Ingestor Agent

**Role**: Step 1 of the `/sqe-kit:gen-user-stories` pipeline. Accepts one or more requirement
documents in any supported format, extracts their text content, and produces a normalised
extraction bundle for downstream agents.

---

## Input Parameters

```
reqInput         : string   — file path, comma-separated paths, or directory path
projectName      : string   — from --project argument (default: "sqe-project")
outputBase       : string   — outputs/<projectName>/user-stories/ (greenfield) or .sqe/user-stories/ (brownfield)
docTypeHint      : string   — optional --type hint (brd|prd|feature|epic|workflow|data|api|ux|meeting|email|other)
dryRun           : boolean  — if true, report what would be extracted without writing output
```

---

## Execution Steps

### Step 1: Resolve Input Files

Parse `reqInput`:

```
if reqInput is a directory path:
  glob all supported extensions: *.md, *.txt, *.pdf, *.docx, *.xlsx, *.csv,
  *.yaml, *.yml, *.json, *.xml, *.bpmn, *.eml, *.png, *.jpg, *.jpeg
  sort alphabetically
  log: "Found N files in <directory>: [file1, file2, ...]"

elif reqInput contains commas:
  split by comma, trim whitespace
  validate each path exists — error on missing

else:
  treat as single file path
  validate it exists — error if not: "Requirement file not found at [path]"
```

If zero files resolved after validation: stop with error "No valid input files found."

### Step 2: Extract Text from Each File

Read and apply `skills/req-text-extractor.md` (resolve via PLUGIN_DIR) for each file.

Process files sequentially. For each file:
1. Detect extension → assign Tier (1/2/3) per extractor rules
2. Apply extraction strategy for that Tier
3. Collect result object:
   ```json
   {
     "file": "<path>",
     "format": "<ext without dot>",
     "tier": 1,
     "extractionMethod": "<description>",
     "rawText": "<extracted text>",
     "warnings": []
   }
   ```
4. If Tier 3 (unsupported): log the user action message from extractor rules; add to `skipped[]`; continue with remaining files

### Step 3: Validate Extraction Results

After processing all files:
- If ALL files were Tier 3 (all skipped): stop with error "No files could be extracted. See warnings above."
- If some files skipped: warn "N file(s) skipped (unsupported format). Continuing with M extracted file(s)."
- If all extracted texts are empty strings: stop with error "Extracted files contain no readable text."

### Step 4: Combine Extraction Results

Merge all extracted texts with delimiters:
```
=== FILE: <filename> ===
<rawText>
=== END: <filename> ===
```

Also build the extraction manifest array for downstream agents.

### Step 5: Write Output (unless --dry-run)

Write to `<outputBase>/00-analysis/extraction-manifest.json`:
```json
{
  "generatedAt": "<ISO timestamp>",
  "projectName": "<projectName>",
  "docTypeHint": "<hint or null>",
  "files": [
    {
      "file": "path/to/file.ext",
      "format": "pdf",
      "tier": 1,
      "extractionMethod": "Read tool with pages",
      "charCount": 4821,
      "warnings": []
    }
  ],
  "skipped": ["path/to/unsupported.fig"],
  "combinedText": "<full merged text from all files>"
}
```

If `--dry-run`: print the manifest structure without writing. Report:
```
[DRY RUN] Would extract:
  ✓ file1.pdf (Tier 1 — Read tool)
  ✓ file2.docx (Tier 2 — pandoc conversion)
  ⚠ file3.fig (Tier 3 — unsupported, will be skipped)
Total: 2 files extractable, 1 skipped
```

### Step 6: Report to Orchestrator

Return to orchestrator:
```
extractionManifestPath : <outputBase>/00-analysis/extraction-manifest.json
extractedFileCount     : N
skippedFileCount       : M
combinedTextLength     : <char count>
warnings               : [...]
```

---

## Error Conditions

| Condition | Action |
|-----------|--------|
| File path does not exist | Error and stop |
| Directory is empty | Error and stop |
| All files are Tier 3 | Error and stop |
| Extraction produces empty text | Warning; skip that file |
| Shell tool unavailable for Tier 2 | Fall through to next fallback → then Tier 3 |
| PDF has 0 readable pages | Warning: "PDF may be scanned/image-only — try exporting as PNG" |

---

## Notes

- Never modify the source requirement documents
- Temp files created during Tier 2 conversion go to system temp dir; clean up after extraction
- Log each file's tier and method before processing so user can see progress
- Pass `docTypeHint` to `agents/capability-1/req-analyzer.agent.md` to assist classification
