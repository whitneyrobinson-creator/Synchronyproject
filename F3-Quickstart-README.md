# F3 Quickstart — How to Use the Asset Files

**Feature:** F3 — Assets: Data Dictionary Generation
**Project:** Synchrony Documentation Automation Skillset
**Last Updated:** 2026-04-10

---

## What's in the Assets Folder

All F3 files live in one folder: `assets/` at the repository root.

```
assets/
├── data_dictionary_template.md          ← Template: defines the data dictionary format
├── qa_report_template.md                ← Template: defines the QA report format
├── sample_schema.json                   ← Test input: 25 UCI Credit Card fields
├── example_data_dictionary.md           ← Gold standard: the answer key for the data dictionary
├── example_qa_report.md                 ← Gold standard: the answer key for the QA report
└── glossary_template.json               ← Placeholder for future glossary feature (ignore for now)
```

### Quick Summary

| File | What It Is | Who Uses It |
|------|-----------|-------------|
| `data_dictionary_template.md` | Blank form — defines what the final data dictionary looks like | F2's `assemble_output.py` reads it during pipeline execution |
| `qa_report_template.md` | Blank form — defines what the QA report looks like | F2's `generate_qa_report.py` reads it during pipeline execution |
| `sample_schema.json` | Test data — 25 fields from the UCI Credit Card dataset | F2 scripts use it to test the whole pipeline |
| `example_data_dictionary.md` | Answer key — what correct output looks like for all 25 fields | Team uses it to check if the pipeline got the right answer |
| `example_qa_report.md` | Answer key — what correct QA report numbers look like | Team uses it to check if the QA report is accurate |
| `glossary_template.json` | Future feature placeholder — not used for demo day | Nobody right now |

---

## How to Use a Template

Templates are blueprints. F2 scripts read them and follow their structure to build the final output files. You don't fill in templates by hand — the scripts do it.

### What the Data Dictionary Template Contains

The template has four sections:

| Section | What's In It | What F2 Does With It |
|---------|-------------|---------------------|
| **Header** | Run metadata with placeholders: `{table_name}`, `{source_file}`, `{generation_date}` | Replaces the placeholders with real values |
| **Main Table** | Column headers: field_name, type, nullable, constraints, enums, description, confidence, last_verified | Reads the headers, generates one row per field |
| **Evidence & Citations** | Format for per-field evidence entries | Generates one entry per field with source_path and evidence_refs |
| **Footer** | Confidence legend, generation note, clarification note | Copies it as-is into the output |

### What the QA Report Template Contains

| Section | What's In It |
|---------|-------------|
| **Report Header** | Run timestamp, pipeline version, input file name, table name |
| **Coverage Statistics** | Total fields, fields with descriptions, completeness percentage |
| **Confidence Distribution** | Counts of High, Medium, Low, and N/A confidence fields |
| **Fields Requiring Clarification** | List of fields that need human review |
| **Merge Corrections** | Any fixes the pipeline made automatically |
| **Warnings** | Anything that went wrong during processing |
| **Processing Notes** | LLM availability, fields processed, batching info |

### What the Placeholders Mean

The templates use three placeholders in the header. These are the only placeholders in the entire template — everything else is structural.

| Placeholder | What It Gets Replaced With | Example |
|-------------|---------------------------|---------|
| `{table_name}` | The table name from the input schema | `credit_card_clients` |
| `{source_file}` | The source file name from the input schema | `sample_schema.json` |
| `{generation_date}` | The timestamp of the pipeline run | `2026-05-07T14:30:00Z` |

---

## How to Use the Sample Schema

The sample schema (`sample_schema.json`) is the test input for the pipeline. It contains 25 fields from the UCI Credit Card dataset.

### Running an End-to-End Test

1. Copy `assets/sample_schema.json` to `output/intermediate/user_input.json`
2. Run the F2 pipeline (all 6 scripts in order)
3. The pipeline produces `output/data_dictionary.md` and `output/qa_report.md`
4. Compare the output against the gold standards (see next section)

### What's Inside the Sample Schema

```json
{
  "table_name": "credit_card_clients",
  "source_file": "sample_schema.json",
  "fields": [
    {
      "field_name": "ID",
      "type": "INTEGER",
      "nullable": false,
      "constraints": ["PRIMARY KEY"],
      "enums": [],
      "schema_comments": "Unique client identifier"
    },
    {
      "field_name": "PAY_0"
    }
  ]
}
```

Some fields have complete metadata (like `ID` above). Others have only a `field_name` (like `PAY_0`). This is intentional — it tests how the pipeline handles missing information.

### Three Required Top-Level Keys

| Key | What It Is | Example |
|-----|-----------|---------|
| `table_name` | Name of the database table | `"credit_card_clients"` |
| `source_file` | Name of the original data source file | `"sample_schema.json"` |
| `fields` | Array of field objects (at least 1) | `[{"field_name": "ID"}, ...]` |

---

## How to Use the Gold Standard

The gold standards are the answer keys. After running the pipeline, you compare the output against these files to check if the pipeline got the right answer.

### Comparing Pipeline Output to the Gold Standard

**Step 1: Structure check**

Open the pipeline output (`output/data_dictionary.md`) and the gold standard (`assets/example_data_dictionary.md`) side by side. Check:

- Same number of columns in the main table?
- Same column order?
- Same number of fields (25)?
- Evidence section has the same layout?
- Footer is present?

These should be identical. The template enforces the structure.

**Step 2: Deterministic content check**

These fields come from the schema, not the LLM. They should be identical between the pipeline output and the gold standard:

- field_name
- type
- nullable
- constraints
- enums
- source_path
- last_verified (format matches, but the actual timestamp will differ)

**Step 3: LLM content check**

These fields are generated by Claude and will vary between runs:

- description
- confidence
- evidence_refs

Score these manually against the gold standard. Ask:
- Is the description accurate? (Does it mean the same thing as the gold standard's description?)
- Is the confidence level appropriate? (Does it match the signal strength?)
- Does the evidence make sense? (Does it explain why the confidence was assigned?)

**Step 4: QA report check**

Compare `output/qa_report.md` against `assets/example_qa_report.md`:

- Same sections present?
- Coverage stats make sense? (If the LLM described all 25 fields, completeness should be 100%)
- Confidence distribution adds up? (High + Medium + Low + N/A = 25)

---

## How to Add a New Template

If someone wants to create a template for a different document type (e.g., an RCSA control narrative template for Skill 2):

**Step 1: Define the structure**

Decide what sections and columns the new document needs. Write them out on paper or in a scratch file first.

**Step 2: Create the Markdown file**

Follow the same patterns as the existing templates:
- Header section with `{placeholder}` substitution for run metadata
- Main content section with column headers that F2 can read
- Footer with any legends or notes

**Step 3: Follow the formatting rules**

| Rule | Why |
|------|-----|
| No pipe characters (`|`) in cell content | Breaks Markdown tables |
| No newlines inside table cells | Breaks the table at that row |
| Always include the header separator row (`|---|---|`) | Without it, the table renders as plain text |
| No angle brackets (`<NA>`) in content | Disappears in rendered Markdown |
| Use semicolons to separate multiple items in a cell | Line breaks break the formatting |

**Step 4: Create a matching gold standard**

Every template needs an answer key. Create a completed example that follows the template's structure exactly.

**Step 5: Update F2 scripts**

The F2 script that assembles the output needs to know about the new template — where to find it, how to read it, and how to fill it in. This is F2 work, not F3 work.

**Step 6: Get team sign-off**

Someone other than the creator reviews the template and gold standard.

---

## How to Update the Gold Standard

The gold standard is NOT an isolated file. It connects to two other files:

```
example_data_dictionary.md (gold standard)
    ↓
example_qa_report.md (QA report stats are derived from gold standard)
    ↓
SKILL.md (3 worked examples may be copied from gold standard fields)
```

If you change the gold standard without checking the chain, the files become inconsistent.

### The 5-Step Update Process

**Step 1: Make the edit**

Edit the field in `example_data_dictionary.md`.

**Step 2: Did the confidence level or clarification_flag change?**

- If you just reworded a description (same confidence, same flag) → skip to Step 3
- If the confidence changed (e.g., Medium → High) → update the Confidence Distribution counts in `example_qa_report.md`
- If the clarification_flag changed (true → false or vice versa) → update the Fields Requiring Clarification list in `example_qa_report.md`

**Step 3: Is this field one of the 3 worked examples in SKILL.md?**

The SKILL.md has 3 fields embedded as worked examples (one High, one Medium, one Low). Check if the field you edited is one of those 3.

- If yes → update the corresponding example in SKILL.md to match your edit exactly
- If no → no SKILL.md change needed

**Step 4: Verify the numbers add up**

Quick sanity check on `example_qa_report.md`:
- High + Medium + Low + N/A = 25 (total fields)
- Every field with `clarification_flag` = true in the gold standard appears in the "Fields Requiring Clarification" section
- Coverage Statistics reflect the correct counts

**Step 5: Team review and sign-off**

Someone other than the person who made the edit reviews the change. No solo edits go live without a second pair of eyes.

### Quick Reference Checklist

```
□ Edit the field in example_data_dictionary.md
□ Did confidence or flag change? → Update example_qa_report.md
□ Is this a SKILL.md worked example? → Update SKILL.md
□ Verify: High + Medium + Low + N/A = 25
□ Get team sign-off
```

---

## How to Add a New Sample Schema

If someone wants to test with a different dataset (e.g., Synchrony's real data post-handoff):

### What You Need

| File | Needed? | Why |
|------|---------|-----|
| New schema JSON | Yes | This is the pipeline input |
| New gold standard data dictionary | Yes | You need an answer key for the new data |
| New gold standard QA report | **No** | The QA report script was already validated with UCI — it works for any schema |

### The Process

**Step 1: Create the new schema JSON**

Must follow the input contract:
- Top-level keys: `table_name` (string), `source_file` (string), `fields` (array)
- Per field: `field_name` (required), plus optional `type`, `nullable`, `constraints`, `enums`, `schema_comments`

**Step 2: Create a gold standard data dictionary**

- Use `data_dictionary_template.md` as the format (same template — it works for any schema)
- Fill in fields by hand with correct descriptions, confidence levels, and evidence_refs
- **Do NOT use the LLM to generate the answer key** — that defeats the purpose
- Two approaches:
  - **Full gold standard** (all fields) — use when schema has <50 fields
  - **Sample gold standard** (15-20 representative fields) — use when schema has 50+ fields. Must include at least 3 High, 3 Medium, 3 Low confidence candidates, plus fields with enums, nulls, and missing comments.

**Step 3: Run the pipeline with the new schema as input**

**Step 4: Compare pipeline output against the new gold standard**

**Step 5: Do NOT delete or modify the original UCI files**

Keep them permanently for:
- Regression testing — re-run UCI after code changes to make sure nothing broke
- Demo day artifact — part of your project history
- Known good test — every field has been verified by hand

### What the Folder Looks Like After Adding a New Schema

```
assets/
├── data_dictionary_template.md              ← shared (all schemas use this)
├── qa_report_template.md                    ← shared (all schemas use this)
├── glossary_template.json                   ← shared (deferred)
│
├── sample_schema.json                       ← UCI input (keep forever)
├── example_data_dictionary.md               ← UCI answer key (keep forever)
├── example_qa_report.md                     ← UCI QA answer key (keep forever
│                                               — ONLY QA gold standard needed)
│
├── synchrony_accounts_schema.json           ← NEW input
└── example_synchrony_accounts_dictionary.md ← NEW answer key
                                               (no QA gold standard needed)
```

---

## Troubleshooting

### JSON Problems (sample_schema.json)

| # | Problem | Symptom | Fix |
|---|---------|---------|-----|
| 1 | Trailing comma in JSON array or object (e.g., `[1, 2, 3,]`) | F2 crashes with JSON parse error | Remove the comma after the last item |
| 2 | `null` vs `"null"` — using `"null"` (with quotes) instead of `null` (without) | F2 treats `"null"` as a real schema comment. Field gets Medium/High confidence instead of Low. The Low confidence path is never tested. | Use `null` (no quotes) for empty values |
| 3 | Single quotes instead of double quotes (`'field_name'` instead of `"field_name"`) | JSON parse error — JSON requires double quotes only | Replace all single quotes with double quotes |
| 4 | Enums as a string instead of an array (`"enums": "[1, 2]"` instead of `"enums": [1, 2]`) | F2 tries to iterate over an array but gets a string. Script crashes or produces wrong output. | Remove the outer quotes so it's a real JSON array |
| 5 | Copy-paste from Google Docs or Word — invisible smart quotes (" " instead of " ") | JSON parse error on a line that looks perfectly fine | Paste into a plain text editor first to strip formatting, then paste into the JSON file |
| 6 | File saved as UTF-16 or with a BOM (byte order mark) instead of UTF-8 | JSON parser chokes on invisible characters at the start of the file | Re-save as UTF-8 without BOM |

### Markdown Problems (templates and gold standard)

| # | Problem | Symptom | Fix |
|---|---------|---------|-----|
| 7 | Pipe character in table cell content (e.g., "True\|False flag") | Table columns shift right. Row has wrong number of columns. | Escape with backslash (`True\|False`) or reword: "Boolean flag (True or False)" |
| 8 | Newline inside a Markdown table cell (e.g., line break in evidence_refs) | Table breaks at that row. Everything below shifts. | Keep all cell content on one line. Use semicolons to separate multiple items instead of line breaks. |
| 9 | Missing header separator row (the `|---|---|` row between header and data) | Table renders as plain text with pipes, not as an actual table. F2's parser can't find the table. | Add the separator row: `| --- | --- | --- |` |
| 10 | Angle brackets in descriptions (`<NA>`, `<null>`) | Text between angle brackets disappears in rendered Markdown — browser treats it as an HTML tag | Use backticks: `` `<NA>` `` or reword: "NA value" |

### Cross-File Consistency Problems

| # | Problem | Symptom | Fix |
|---|---------|---------|-----|
| 11 | Field count mismatch — schema has 24 fields but gold standard has 25 | Pipeline output can't be compared to gold standard. Diff shows extra or missing fields. | Count fields in both files. Must be identical. |
| 12 | Field name typo — schema has PAY_0 but gold standard has PAY_1 | That field can't be matched between pipeline output and gold standard | Verify field names match exactly between `sample_schema.json` and `example_data_dictionary.md` |
| 13 | Description exceeds 25 words | Fails FR-007 validation | Tighten the wording. Every word must earn its place. |
| 14 | Confidence = Low but clarification_flag = false | QA report won't list the field in "Fields Requiring Clarification" even though it should be there | Low confidence always means clarification_flag = true. Fix the flag. |
| 15 | Updated template but didn't update gold standard | Gold standard no longer matches template structure. SC-F3-002 fails. | After any template change, update the gold standard to match (follow the 5-step update process). |

---

## What You Can Do Without F2

F3 assets are static files. You can create, edit, and review them without any F2 scripts being built. Here's what you can do right now:

| Task | Needs F2? |
|------|----------|
| Create or edit templates | No |
| Create or edit the sample schema | No |
| Create or edit gold standards | No |
| Validate templates against the spec (SC-F3-001) | No — manual review |
| Validate gold standard structure matches template (SC-F3-002) | No — manual review |
| Validate sample schema variety (SC-F3-003) | No — manual review |
| Validate gold standard accuracy (SC-F3-004) | No — manual review |
| Cross-check QA report numbers against data dictionary (SC-F3-005) | No — manual counting |
| Test that F2 scripts can read the files (SC-F3-006) | **Yes** — blocked until F2 scripts are built |

5 out of 6 success criteria can be checked right now. Only SC-F3-006 (integration test) requires F2.

---

*This guide covers how to use, update, and troubleshoot F3's asset files. For file structure details, see data-model.md. For the formal agreements between F3 and other features, see contracts/. For the reasoning behind these designs, see research.md.*
