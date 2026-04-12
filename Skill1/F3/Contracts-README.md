# F3 Contracts — Agreements Between Features

**Feature:** F3 — Assets: Data Dictionary Generation
**Project:** Synchrony Documentation Automation Skillset
**Last Updated:** 2026-04-10

---

## What This Document Is

This file defines the formal agreements between F3 and the features that use its files. If F3 changes a file, these contracts tell you who gets affected and what breaks.

There are four contracts:

1. **F3 → F1** — Authoring dependency (soft contract)
2. **F3 → F2** — Runtime dependency (hard contract)
3. **Template Contract** — Rules every template must follow
4. **Gold Standard Contract** — Rules the gold standards must follow

---

## Contract 1: F3 → F1 (Authoring Dependency)

**Type:** Soft contract — no runtime coupling
**Risk level:** Low

### What This Means

F1's SKILL.md does NOT read any F3 files at runtime. Claude never opens an F3 file. The only connection is:

1. The team creates the gold standard data dictionary (`example_data_dictionary.md`)
2. The team picks 3 representative fields from it (one High, one Medium, one Low confidence)
3. The team copies those 3 fields into the SKILL.md as worked examples
4. The worked examples live permanently inside the SKILL.md — they are not read from an external file

### What F1 Expects from F3

| Expectation | Details |
|-------------|---------|
| Gold standard is accurate | The 3 worked examples are copied from the gold standard. If the gold standard has wrong descriptions or confidence scores, the worked examples teach Claude the wrong thing. |
| Gold standard covers all confidence levels | The team needs at least one High, one Medium, and one Low confidence field to pick from. |

### What F3 Does NOT Need to Guarantee for F1

| Not Required | Why |
|-------------|-----|
| File paths | F1 never reads F3 files at runtime |
| Machine-readable format | F1 never parses F3 files |
| Specific file names | Renaming F3 files has zero impact on F1 |
| Automatic updates | If the gold standard changes, someone manually updates the SKILL.md examples. This is a team responsibility, not a system dependency. |

### When F1 Is Affected by F3 Changes

| F3 Change | Impact on F1 | Action Required |
|-----------|-------------|----------------|
| Reword a description in the gold standard | No impact UNLESS that field is one of the 3 worked examples | Check if the field is a worked example. If yes, update the SKILL.md to match. |
| Change a confidence score in the gold standard | No impact UNLESS that field is one of the 3 worked examples | Same — check and update if needed. |
| Change the gold standard's structure (new columns, new sections) | The SKILL.md examples may no longer match the expected output format | Update all 3 worked examples in the SKILL.md to match the new structure. |
| Rename or move a gold standard file | No impact | Nothing. F1 doesn't read the file. |
| Change the template | No impact on F1 directly, but the SKILL.md tells Claude to "follow the data dictionary template structure" — if the structure changes significantly, the SKILL.md instructions may need rewording | Review the SKILL.md instructions. Update if the template structure changed in a way that makes the instructions inaccurate. |

---

## Contract 2: F3 → F2 (Runtime Dependency)

**Type:** Hard contract — F2 scripts read F3 files during execution
**Risk level:** High — if F3 files are wrong or missing, F2 breaks

### Files F2 Reads from F3

| F2 Script | F3 File | File Path | When It's Read |
|-----------|---------|-----------|---------------|
| `assemble_output.py` | Data dictionary template | `assets/data_dictionary_template.md` | During output assembly — after F1 produces field descriptions |
| `generate_qa_report.py` | QA report template | `assets/qa_report_template.md` | During QA report generation — after output assembly |
| All scripts (testing only) | Sample schema | `assets/sample_schema.json` | During end-to-end testing — as pipeline input |

### File Path Guarantees

F3 guarantees these exact file paths. F2 scripts can hardcode them.

| File | Guaranteed Path |
|------|----------------|
| Data dictionary template | `assets/data_dictionary_template.md` |
| QA report template | `assets/qa_report_template.md` |
| Sample schema | `assets/sample_schema.json` |
| Gold standard data dictionary | `assets/example_data_dictionary.md` |
| Gold standard QA report | `assets/example_qa_report.md` |
| Glossary template (P3) | `assets/glossary_template.json` |

**If a file is renamed or moved, this contract is broken.** F2 scripts will fail.

### What F2 Expects from the Data Dictionary Template

| Expectation | Details | What Breaks If Violated |
|-------------|---------|------------------------|
| File exists at `assets/data_dictionary_template.md` | Must be present before the pipeline runs | Script stops with error: "Template not found at assets/data_dictionary_template.md" |
| Valid Markdown | Tables must render correctly — header row, separator row, data rows | `assemble_output.py` can't parse the template structure |
| 8 columns in the main table | field_name, type, nullable, constraints, enums, description, confidence, last_verified | Output has wrong columns or missing data |
| 3 header placeholders | `{table_name}`, `{source_file}`, `{generation_date}` | Output shows raw placeholder text instead of real values |
| Evidence & Citations section present | With the `### {field_name}` subheading pattern | Evidence section is missing or malformed in the output |
| Footer with confidence legend | Static content that gets copied to output | Output is missing the legend — auditors don't know what High/Medium/Low means |

### What F2 Expects from the QA Report Template

| Expectation | Details | What Breaks If Violated |
|-------------|---------|------------------------|
| File exists at `assets/qa_report_template.md` | Must be present before the pipeline runs | Script stops with error: "Template not found at assets/qa_report_template.md" |
| Report Header format | Run timestamp, pipeline version, input file name, table name | QA report has no metadata header |
| 6 sections in order | Coverage Statistics, Confidence Distribution, Fields Requiring Clarification, Merge Corrections, Warnings, Processing Notes | Sections are missing, out of order, or have wrong names |
| Section names match exactly | F2's `generate_qa_report.py` uses these names as section headers in the output | Section headers in the output don't match the template |
| Warnings section supports multiple warning types | Including extraction warnings (e.g., duplicate field names), LLM output validation issues, missing LLM output, and pipeline integrity warnings (e.g., confidence distribution sum mismatch). The section is conditional — only present if warnings exist. | New warning types are suppressed or the section structure doesn't accommodate them |

### What F2 Expects from the Sample Schema

| Expectation | Details | What Breaks If Violated |
|-------------|---------|------------------------|
| File exists at `assets/sample_schema.json` | Must be present for testing | Can't run end-to-end tests |
| Valid JSON | Parseable by Python's `json.load()` | JSON parse error — script crashes |
| Has `table_name` (string) | Top-level key | F2 can't name the output file or populate the template header |
| Has `source_file` (string) | Top-level key | F2 can't build `source_path` for each field |
| Has `fields` (array of objects) | Top-level key, at least 1 field | F2 has nothing to process |
| Every field has `field_name` (string) | Required per-field key | F2 can't identify the field |

### Missing Template Behavior

If `assemble_output.py` or `generate_qa_report.py` can't find its template file:

1. Script stops immediately
2. Prints: "Template not found at `assets/{filename}`. Pipeline cannot produce formatted output without the template. Ensure all F3 asset files are present before running."
3. No output is produced for that step
4. This is NOT graceful degradation — this is a setup error

**Why not a fallback template:** Creates two sources of truth. If someone updates the real template but forgets the backup, the pipeline silently uses the wrong format.

**Why not raw JSON output:** Raw JSON isn't a data dictionary. An auditor can't read it.

---

## Contract 3: Template Contract

Rules that every template file must follow. This applies to both `data_dictionary_template.md` and `qa_report_template.md`.

### Template Is the Single Source of Truth (FR-F3-005)

The template defines the output format. Period.

- F2 scripts read the template and follow its structure
- If the template says the columns are in a certain order, F2 produces them in that order
- If the template says the QA report has 6 sections, F2 produces 6 sections
- If there's a conflict between the template and F2's code, **the template wins** and F2's code must be updated

### Placeholder Rules

| Rule | Details |
|------|---------|
| Header placeholders use single curly braces | `{table_name}`, not `{{table_name}}` |
| Only 3 header placeholders exist | `{table_name}`, `{source_file}`, `{generation_date}` |
| Table rows are NOT placeholders | F2 generates rows dynamically based on the data — the template only defines the column headers |
| Evidence entries are NOT placeholders | F2 generates one entry per field — the template only defines the entry format |
| F2 substitutes header placeholders using `.replace()` in a fixed order, before inserting any field data | This prevents accidental substitution if field values happen to contain placeholder-like text (e.g., a schema comment containing the literal string `{table_name}`) |

### Markdown Formatting Rules

| Rule | Why |
|------|-----|
| No pipe characters (`\|`) in cell content | Breaks the table — columns shift right |
| No newlines inside table cells | Breaks the table at that row |
| Always include the header separator row (`\|---\|---\|`) | Without it, the table renders as plain text |
| No angle brackets in content (`<NA>`, `<null>`) | Disappears in rendered Markdown — browser treats it as HTML |
| Use backticks for technical values | `null`, `true`, `false` |
| Evidence items separated by semicolons, not line breaks | Line breaks inside a table cell or evidence entry break the formatting |
| F2 escapes special Markdown characters in field names before writing to tables | Field names containing `\|`, `*`, `` ` ``, `\`, `[`, `]` are escaped in the Markdown output only. Intermediate JSON files retain the raw unescaped names. |

### Template Versioning

Templates don't have version numbers. There's only one version — the current one. If the template changes:

1. Update the gold standard to match (follow the 5-step update process)
2. Verify F2 scripts still produce correct output
3. Get team sign-off

---

## Contract 4: Gold Standard Contract

Rules that the gold standard files must follow.

### Gold Standard Data Dictionary (`example_data_dictionary.md`)

| Rule | Details |
|------|---------|
| Structure matches the template exactly | Same header format, same column order, same evidence section layout, same footer |
| All 25 fields present | One row per field in the main table, one entry per field in the evidence section |
| Field order matches the sample schema | Fields appear in the same order as `sample_schema.json` |
| Descriptions ≤25 words | FR-007 compliance |
| Confidence scores follow the rubric | High = multiple strong signals, Medium = some signals, Low = few/no signals |
| Low confidence → `clarification_flag` = true | Every Low field has `[NEEDS CLARIFICATION]` prepended to its description |
| Evidence is realistic | `evidence_refs` explain the reasoning, not just "this is High confidence" |
| `source_path` format | `sample_schema.json → table: credit_card_clients → field: {field_name}` |

### Gold Standard QA Report (`example_qa_report.md`)

| Rule | Details |
|------|---------|
| Structure matches the QA report template exactly | Same sections, same order, same stat categories |
| Numbers are derived from the gold standard data dictionary | Every stat must be verifiable by counting items in `example_data_dictionary.md` |
| High + Medium + Low + N/A = 25 | Confidence distribution must sum to total fields |
| Fields Requiring Clarification list matches | Every field with `clarification_flag` = true in the data dictionary appears in this section |
| Merge Corrections | "No merge corrections." (gold standard represents a clean run) |
| Warnings | "No warnings." (gold standard represents a clean run) |

### How Gold Standards Map to Pipeline Output

When comparing pipeline output against the gold standard:

| What to Compare | Expected Result |
|----------------|----------------|
| **Structure** (columns, sections, field count) | Identical — the template enforces this |
| **Deterministic fields** (field_name, type, nullable, constraints, enums, source_path, last_verified) | Identical — these come from the schema, not the LLM |
| **LLM fields** (description, confidence, evidence_refs) | Will vary between runs — score manually against the gold standard |
| **QA report stats** | Should match if the LLM produced the same confidence distribution — but will vary if the LLM assigns different confidence levels |

### Only One QA Report Gold Standard — Ever

The QA report gold standard tests whether `generate_qa_report.py` counts and formats correctly. It tests the script, not the data.

Once the script is validated with the UCI gold standard, it works for any schema. When adding a new schema post-handoff:

- **New gold standard data dictionary:** Required (you need an answer key for the new data)
- **New gold standard QA report:** NOT required (the script already works)

---

## Cross-Reference: Alignment with F1 and F2 Contracts

### F2's Input Contract vs. F3's Sample Schema

F2's input contract requires:

| Key | F2 Expects | F3 Provides | Aligned? |
|-----|-----------|-------------|----------|
| `table_name` | Required string | Present | Yes |
| `source_file` | Required string | Present | F2's current docs list only `table_name` and `fields` — needs update to add `source_file` |
| `fields` | Required array | Present | Yes |
| `fields[].field_name` | Required string | Present | Yes |
| `fields[].type` | Optional, default `"UNKNOWN"` | Present (some fields intentionally missing) | Yes |
| `fields[].nullable` | Optional, default `true` | Present (some fields intentionally missing) | Yes |
| `fields[].constraints` | Optional, default `[]` | Present | Yes |
| `fields[].enums` | Optional, default `[]` | Present | Yes |
| `fields[].schema_comments` | Optional, default `null` | Present (some fields intentionally `null`) | Yes |

### F2's QA Report Sections vs. F3's QA Report Template

| # | F3 Template Section Name | F2 Must Use This Name | Aligned? |
|---|-------------------------|----------------------|----------|
| — | Report Header | Report Header | Yes |
| 1 | Coverage Statistics | Coverage Statistics | F2 currently says "Coverage Summary" — needs rename |
| 2 | Confidence Distribution | Confidence Distribution | F2 currently missing this section — needs addition |
| 3 | Fields Requiring Clarification | Fields Requiring Clarification | F2 currently says "Flagged Fields" — needs rename |
| 4 | Merge Corrections | Merge Corrections | Yes |
| 5 | Warnings | Warnings | F2 currently says "Extraction Warnings" — needs rename |
| 6 | Processing Notes | Processing Notes | F2 currently says "Run Summary" — needs rename |

### Tracked F2 Updates Required

These mismatches were identified during F3 planning. F2 docs need these updates:

| F2 Document | Change |
|-------------|--------|
| Input contract | Add `source_file` as required top-level key |
| Data model | Add `source_file` to Section 1 |
| All QA report references | Rename sections to match F3's template names |
| Quickstart | Add `source_file` to all input examples |
| Edge cases | Add "template file missing → hard stop" |
| Plan | Add task: "Add template file existence check to `assemble_output.py` and `generate_qa_report.py`" |

---

*This document defines the agreements between F3 and the features that consume its files. For file structure details, see data-model.md. For how to use these files, see quickstart.md. For the reasoning behind these decisions, see research.md.*
