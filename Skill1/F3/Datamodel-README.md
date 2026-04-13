# F3 Data Model — File Structures, Formats, and Rules

**Feature:** F3 — Assets: Data Dictionary Generation
**Project:** Synchrony Documentation Automation Skillset
**Last Updated:** 2026-04-12

---

## What This Document Is

This file defines every asset F3 contains — what each file looks like, what's inside it, what the rules are, and what happens at the edges. If you're creating, editing, or debugging an F3 asset, this is your reference for "what should this file look like?"

---

## File Inventory

F3 delivers 6 files. All live in a single flat `assets/` directory at the repository root.

| File | Format | Purpose | Size |
|------|--------|---------|------|
| `data_dictionary_template.md` | Markdown | Defines the structure of the final data dictionary output | ~60 lines |
| `qa_report_template.md` | Markdown | Defines the structure of the QA report | ~50 lines |
| `sample_schema.json` | JSON | Test input — 25 UCI Credit Card fields | ~200 lines |
| `example_data_dictionary.md` | Markdown | Gold standard — completed data dictionary (all 25 fields) | ~300 lines |
| `example_qa_report.md` | Markdown | Gold standard — completed QA report | ~80 lines |
| `glossary_template.json` | JSON | Placeholder for future P3 glossary feature | ~15 lines |

**File naming rule:** Lowercase, underscores between words, no spaces. Templates start with the document type they define. Gold standards start with `example_`. The sample schema starts with `sample_`.

---

## 1. Data Dictionary Template (`data_dictionary_template.md`)

This is the blueprint for the final data dictionary. F2's `assemble_output.py` reads this file and follows its structure to build the output.

### How the Template Works

The template has four sections. F2 handles each one differently:

| Section | How F2 Uses It |
|---------|---------------|
| **Header** | Simple placeholder substitution — replace `{table_name}`, `{source_file}`, `{generation_date}` with real values |
| **Main Table** | Read the column headers from the template, then generate one row per field using data from `timestamped_fields.json` |
| **Evidence & Citations** | Follow the section structure from the template, generate one entry per field in the same order as the main table |
| **Footer** | Copy as-is — this is static content that doesn't change between runs |

### Header Section

Contains run metadata with three placeholders:

| Placeholder | Replaced With | Example |
|-------------|--------------|---------|
| `{table_name}` | Table name from the input schema | `credit_card_clients` |
| `{source_file}` | Source file name from the input schema | `sample_schema.json` |
| `{generation_date}` | ISO 8601 timestamp of the pipeline run | `2026-05-07T14:30:00Z` |

Also includes static text:
- Skill version (hardcoded in the template)
- A note that the document is a draft for human review

### Main Table Section

A Markdown table with 8 columns in this exact order:

| # | Column | Where the Data Comes From | What to Show If the Value Is Empty |
|---|--------|--------------------------|-----------------------------------|
| 1 | `field_name` | F2 extracted metadata | Always present — required field |
| 2 | `type` | F2 extracted metadata | "Not specified" |
| 3 | `nullable` | F2 extracted metadata | "Not specified" |
| 4 | `constraints` | F2 extracted metadata | "None" |
| 5 | `enums` | F2 extracted metadata | "None" |
| 6 | `description` | F1 LLM output | "[LLM did not return a description]". Prepend "[NEEDS CLARIFICATION]" if `clarification_flag` = true. |
| 7 | `confidence` | F1 LLM output | "N/A" |
| 8 | `last_verified` | F2 timestamp | Always present — ISO 8601 |

**Column order is authoritative.** F2's `assemble_output.py` must read the column headers from this template and generate rows in the same order. The template is the single source of truth for column order (FR-F3-005).

**`[NEEDS CLARIFICATION]` rendering:** This flag is prepended to the description text in the same cell — e.g., `[NEEDS CLARIFICATION] Appears to represent repayment status`. It is NOT a separate column.

### Evidence & Citations Section

A separate section below the main table. One entry per field, in the same order as the main table. Each entry contains:

- **Field name** as a subheading (e.g., `### LIMIT_BAL`)
- **Source:** The `source_path` from F2 (format: `{source_file} → table: {table_name} → field: {field_name}`)
- **Evidence:** The `evidence_refs` from F1 — the LLM's reasoning for the description and confidence score. Rendered as a numbered list, one evidence item per line. This eliminates delimiter ambiguity — no separator character is needed.

Example:

```
### LIMIT_BAL
**Source:** sample_schema.json → table: credit_card_clients → field: LIMIT_BAL
**Evidence:**
1. field_name: 'LIMIT_BAL' suggests credit limit or balance
2. type: DECIMAL confirms numeric monetary value
3. schema_comments: 'Credit limit in NT dollars' confirms meaning
```

### Footer Section

Static content copied as-is from the template:

- **Confidence Level Legend:**
  - High — Multiple strong signals agree (field name, type, constraints, comments all point to the same meaning)
  - Medium — Some signals present but incomplete or partially ambiguous
  - Low — Few or no signals; description is a best guess
- **Generation Note:** "Descriptions are AI-generated drafts for human review."
- **[NEEDS CLARIFICATION] Note:** "Fields marked [NEEDS CLARIFICATION] require human attention before use in production."

---

## 2. QA Report Template (`qa_report_template.md`)

This is the blueprint for the QA report. F2's `generate_qa_report.py` reads this file and follows its structure.

### Report Header

Not a full section — just a few lines of metadata at the top:

| Field | Source |
|-------|--------|
| Run timestamp | ISO 8601 from the pipeline run |
| Pipeline version | Hardcoded in template |
| Input file name | From `source_file` in the input schema |
| Table name | From `table_name` in the input schema |

### Section 1: Coverage Statistics

| Stat | What It Counts | Source |
|------|---------------|--------|
| Total Fields | All fields in the schema | F2 field count |
| Fields with Descriptions | Fields where `merge_status` = `"matched"` (not placeholders) | F2 count |
| Fields with Citations | Fields where `source_path` is present | F2 count (always equals Total Fields) |
| Fields with Confidence Scores | Fields where `confidence` ≠ `"None"` | F2 count |
| Fields Flagged [NEEDS CLARIFICATION] | Fields where `clarification_flag` = `true` | F2 count |
| Completeness | (Fields with Descriptions / Total Fields) × 100 | F2 calculated |

### Section 2: Confidence Distribution

| Level | Count | Percentage |
|-------|-------|-----------|
| High | Count of `confidence` = `"High"` | (High / Total) × 100 |
| Medium | Count of `confidence` = `"Medium"` | (Medium / Total) × 100 |
| Low | Count of `confidence` = `"Low"` | (Low / Total) × 100 |
| N/A | Count of `confidence` = `"None"` | (N/A / Total) × 100 |

**Validation rule:** High + Medium + Low + N/A must equal Total Fields. If they don't add up, something is wrong in the pipeline.

### Section 3: Fields Requiring Clarification

A list of every field where `clarification_flag` = `true`. Each entry shows:

| Column | What It Shows |
|--------|--------------|
| Field name | The field that needs attention |
| Current description | Including the "[NEEDS CLARIFICATION]" prefix |
| Confidence level | High, Medium, Low, or N/A |
| Reason | Brief explanation from `evidence_refs` about why clarification is needed |

### Section 4: Merge Corrections

Cases where F2's `attach_citations.py` got conflicting information from two sources and had to make a choice. Each entry shows:

| Column | What It Shows |
|--------|--------------|
| Field name | The affected field |
| Correction type | One of the 5 types (see F2 data-model.md Section 5) |
| Original value | What the value was before correction |
| Corrected value | What F2 changed it to |
| Reason | Why the correction was made |

**Only present if corrections exist.** If the pipeline ran cleanly with no corrections, this section displays "No merge corrections."

**This is different from Warnings.** A merge correction means the system made a choice between two valid options (verify the decision). A warning means something went wrong (investigate the problem).

### Section 5: Warnings

Anything that went wrong during processing:

- Duplicate field names (from `extraction_warnings.json`, if it exists)
- LLM output validation issues (count mismatches, name mismatches)
- Missing LLM output (if graceful degradation was triggered)

**If no warnings exist:** Displays "No warnings."

### Section 6: Processing Notes

Contextual information about the run:

| Note | Example Value |
|------|--------------|
| LLM availability | "LLM available" or "LLM unavailable — all descriptions are placeholders" |
| Fields processed | "25 fields processed" |
| Batching | "All fields processed in a single pass" (always this for demo day) |
| Run timestamp | ISO 8601 timestamp |

---

## 3. Sample Schema (`sample_schema.json`)

The test input file for the pipeline. Contains 25 fields from the UCI Credit Card dataset.

### Required Top-Level Keys

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `table_name` | string | **Yes** | Name of the database table |
| `source_file` | string | **Yes** | Name of the original data source file |
| `fields` | array of objects | **Yes** | Must have at least 1 field |

### Required Per-Field Keys

| Key | Type | Required | Default If Missing |
|-----|------|----------|-------------------|
| `field_name` | string | **Yes** | — |
| `type` | string | No | `"UNKNOWN"` |
| `nullable` | boolean | No | `true` |
| `constraints` | array of strings | No | `[]` |
| `enums` | array | No | `[]` |
| `schema_comments` | string or null | No | `null` |

### Example — Full Field (All Optional Keys Present)

```json
{
  "field_name": "LIMIT_BAL",
  "type": "DECIMAL",
  "nullable": false,
  "constraints": ["CHECK (LIMIT_BAL > 0)"],
  "enums": [],
  "schema_comments": "Credit limit in NT dollars"
}
```

### Example — Minimal Field (Only Required Key)

```json
{
  "field_name": "PAY_0"
}
```

When F2's `extract_fields.py` processes this minimal field, it fills in defaults: type becomes `"UNKNOWN"`, nullable becomes `true`, constraints becomes `[]`, enums becomes `[]`, schema_comments becomes `null`.

### The 25 Fields

All fields from the UCI Credit Card dataset:

`ID`, `LIMIT_BAL`, `SEX`, `EDUCATION`, `MARRIAGE`, `AGE`, `PAY_0`, `PAY_2`, `PAY_3`, `PAY_4`, `PAY_5`, `PAY_6`, `BILL_AMT1`, `BILL_AMT2`, `BILL_AMT3`, `BILL_AMT4`, `BILL_AMT5`, `BILL_AMT6`, `PAY_AMT1`, `PAY_AMT2`, `PAY_AMT3`, `PAY_AMT4`, `PAY_AMT5`, `PAY_AMT6`, `default.payment.next.month`

### Required Variety (FR-F3-003)

The sample schema must include a mix of field types to test all pipeline paths:

| What Must Be Included | Example | Why |
|-----------------------|---------|-----|
| Fields with complete metadata | `ID`, `LIMIT_BAL` | Tests the happy path |
| Fields with missing `type` | At least 1 field with no `type` key | Tests F2's default handling (`"UNKNOWN"`) |
| Fields with missing `schema_comments` | At least 2-3 fields with `null` comments | Tests F1's Low confidence path |
| Fields with descriptive enums | A field with values like `["ACTIVE", "CLOSED"]` | Tests enum rendering |
| Fields with cryptic numeric enums | `SEX` with `[1, 2]` | Tests confidence scoring when enums aren't self-descriptive |
| Fields with constraints | `ID` with `["PRIMARY KEY"]` | Tests constraint rendering |
| Fields with oddly formatted names | `default.payment.next.month` | Tests field names with dots/special characters |
| A numbered series | `BILL_AMT1` through `BILL_AMT6` | Tests cross-field consistency |

**Minimum thresholds (SC-F3-003):** At least 3 fields with missing optional metadata, at least 1 field with cryptic enums, at least 1 field with constraints.

---

## 4. Gold Standard — Data Dictionary (`example_data_dictionary.md`)

A completed data dictionary for all 25 UCI Credit Card fields. This is the answer key.

### Structure Rules

- Must follow the **exact** structure of `data_dictionary_template.md` — same header format, same column order, same evidence section layout, same footer
- If the template changes, the gold standard must be updated to match (FR-F3-002)
- The gold standard is derived FROM the template, never the other way around

### Content Rules

| Field | Rule |
|-------|------|
| `description` | ≤25 words, plain language, accurate to what the field represents |
| `confidence` | Follows the FR-009 rubric: High (multiple strong signals), Medium (some signals), Low (few/no signals) |
| `confidence` distribution coverage | The gold standard must contain at least 1 High, 1 Medium, and 1 Low confidence field. This ensures all three confidence paths are exercised during end-to-end testing. If the gold standard contains only High and Medium fields (no Low), the Low confidence path and clarification flag behavior are never validated. |
| `evidence_refs` | Realistic reasoning explaining why the confidence score was assigned |
| `source_path` | Format: `sample_schema.json → table: credit_card_clients → field: {field_name}` |
| `last_verified` | Example ISO 8601 timestamp (e.g., `2026-05-07T14:30:00Z`) |
| `clarification_flag` | Must be `true` for all Low confidence fields. Prepend `[NEEDS CLARIFICATION]` to description. |

### Null Handling

When a field's metadata is missing, the gold standard shows the same null rendering the template defines:

| Missing Value | What the Gold Standard Shows |
|--------------|----------------------------|
| No type | "Not specified" |
| No nullable info | "Not specified" |
| No constraints | "None" |
| No enums | "None" |
| No confidence (LLM failed) | "N/A" |
| No description (LLM failed) | "[LLM did not return a description]" |

**Note:** The gold standard assumes full LLM availability — all 25 fields have real descriptions, confidence scores, and evidence. The null handling examples above are for fields where the *schema metadata* is missing, not where the LLM failed.

---

## 5. Gold Standard — QA Report (`example_qa_report.md`)

A completed QA report for the 25-field UCI Credit Card dataset. Derived directly from the gold standard data dictionary.

### Structure Rules

- Must follow the **exact** structure of `qa_report_template.md` — same sections, same stat categories, same order
- If the template changes, the gold standard must be updated to match (FR-F3-002)

### Content Rules

| Stat | Expected Value | Why |
|------|---------------|-----|
| Total Fields | 25 | UCI dataset has 25 fields |
| Fields with Descriptions | 25 | Gold standard assumes full LLM availability |
| Fields with Citations | 25 | Every field gets a `source_path` |
| Fields with Confidence Scores | 25 | Gold standard assumes full LLM availability |
| Completeness | 100% | All fields described |
| Warnings | "No warnings." | Clean run in the gold standard |
| Processing Notes | "All fields processed in a single pass. LLM available." | Demo day configuration |

### Internal Consistency Rule (SC-F3-005)

Every number in the QA report must be verifiable by counting items in the gold standard data dictionary:

- If the data dictionary has 3 Low confidence fields → the QA report must show 3 in the Low row of Confidence Distribution
- If the data dictionary has 5 fields with `clarification_flag` = `true` → the QA report must list exactly those 5 fields in Fields Requiring Clarification
- High + Medium + Low + N/A must equal 25

**If the numbers don't match, the gold standard QA report is wrong.** Fix it by recounting from the data dictionary.

---

## 6. Glossary Template (`glossary_template.json`) — P3 Placeholder

Not required for demo day. Exists as a starting point for whoever builds the glossary mapping feature later.

### Structure

```json
{
  "glossary": [
    {
      "term": "LIMIT_BAL",
      "label": "Credit Limit Balance"
    },
    {
      "term": "PAY_0",
      "label": "Repayment Status — September 2005"
    }
  ]
}
```

### Field Reference

| Key | Type | Description |
|-----|------|-------------|
| `glossary` | array of objects | List of term-label pairs |
| `glossary[].term` | string | Field name to match against (exact match) |
| `glossary[].label` | string | Standardized business label |

**Status:** P3. Not consumed by any script. Not tested. Not required for demo day.

---

## 7. Validation Rules

How to verify each F3 asset is correct:

### Template Validation (SC-F3-001)

| Check | How to Verify |
|-------|--------------|
| All 8 columns present in data dictionary template | Open the file, count the columns in the main table header |
| Columns in correct order | Compare against the order in Section 1 of this document |
| All 6 sections present in QA report template | Open the file, check section headings match Section 2 of this document |
| Header placeholders present | Search for `{table_name}`, `{source_file}`, `{generation_date}` |
| Footer content present | Confidence legend, generation note, clarification note |
| Valid Markdown syntax | Paste into a Markdown previewer — tables should render correctly |

### Gold Standard Validation (SC-F3-002, SC-F3-004)

| Check | How to Verify |
|-------|--------------|
| Structure matches template | Diff the section headings and column order against the template |
| All 25 fields present | Count the rows in the main table and entries in the evidence section |
| Descriptions ≤25 words | Count words in each description |
| Confidence scores follow rubric | High = multiple strong signals, Medium = some signals, Low = few/no signals |
| At least 1 High, 1 Medium, and 1 Low confidence field present | Count the confidence values in the main table |
| Low confidence fields have `clarification_flag` = true | Check that every Low field has `[NEEDS CLARIFICATION]` prepended |
| Evidence section field order matches main table | Compare the order of entries in both sections |
| QA report numbers match data dictionary counts | Cross-check every stat by counting in the data dictionary |

### Sample Schema Validation (SC-F3-003)

| Check | How to Verify |
|-------|--------------|
| Valid JSON | Paste into a JSON validator |
| Has `table_name`, `source_file`, and `fields` | Check top-level keys |
| 25 fields present | Count objects in the `fields` array |
| Every field has `field_name` | Check each object |
| At least 3 fields with missing optional metadata | Count fields where `type`, `nullable`, or `schema_comments` is absent or null |
| At least 1 field with cryptic enums | Check for numeric-only enum arrays |
| At least 1 field with constraints | Check for non-empty `constraints` arrays |

---

## 8. Edge Cases

| Scenario | What Happens | How to Prevent |
|----------|-------------|----------------|
| Template column order doesn't match what F2 expects | Output has misaligned columns | F2 reads column order FROM the template (FR-F3-005) — so this shouldn't happen unless F2 hardcodes the order |
| Gold standard has different structure than template | SC-F3-002 fails. Gold standard is wrong. | Always derive the gold standard FROM the template. If the template changes, update the gold standard. |
| Gold standard has different field count than sample schema | Can't compare pipeline output to gold standard | Both must have exactly 25 fields. Count them. |
| Template placeholder not filled by F2 | Output shows raw `{table_name}` instead of the actual value | F2 bug — `assemble_output.py` must replace all 3 header placeholders |
| Pipe character in table cell content | Markdown table breaks — columns shift | Escape with backslash (`\|`) or reword to avoid the pipe |
| Newline inside a Markdown table cell | Table breaks at that row | Keep all cell content on one line. Use semicolons to separate items. |
| Missing header separator row (`|---|---|`) | Table renders as plain text | Always include the separator row between header and data rows |
| Angle brackets in descriptions (`<NA>`) | Text disappears in rendered Markdown | Use backticks (`` `<NA>` ``) or reword |
| `null` vs `"null"` in sample schema JSON | F2 treats `"null"` as a real value, not as missing | Use `null` (no quotes) for empty values |
| Evidence section field order doesn't match main table | Confusing for auditors | Evidence section MUST list fields in the same order as the main table |
| QA report confidence distribution doesn't sum to total fields | Math error somewhere | High + Medium + Low + N/A must equal Total Fields |

---

*This document defines the structure of every F3 asset. For the reasoning behind these structures, see research.md. For how to use these files, see quickstart.md. For the formal agreements between F3 and other features, see contracts/.*
