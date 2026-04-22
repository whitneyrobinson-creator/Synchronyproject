# Data Dictionary Skill — Master Reference
 
**Project**: Synchrony Documentation Automation Skillset
**Skill**: Data Dictionary Generation (Skill 1)
**Features**: F1 (SKILL.md), F2 (Scripts), F3 (Assets)
**Last Updated**: 2026-04-22
**Demo Day Deadline**: May 7, 2026
 
> This document is the home base for the Data Dictionary skill. It tells you what the skill does, how the three features work together, and what every file in the pipeline is. For deep technical detail on any specific feature, go to that feature's plan.
 
---
 
## What This Skill Does
 
A user uploads a JSON schema file describing a database table. The skill reads every field in that schema and produces two documents:
 
1. **`data_dictionary.md`** — A complete data dictionary with a plain-language description, confidence score, evidence citations, and a clarification flag for every field.
2. **`qa_report.md`** — A quality report showing coverage statistics, confidence distribution, any fields that need human review, and a summary of how the pipeline ran.
Optionally, when the input schema is `sample_schema.json` (the UCI Credit Card dataset), the skill also produces:
 
3. **`comparison_report.md`** — A side-by-side comparison of the skill's output descriptions against the Kaggle ground truth, showing metadata availability per field.
The goal is to take a task that currently takes 60–120 minutes by hand and reduce it to 5–10 minutes, while producing output that is more consistent, more traceable, and more audit-ready than a human-written first draft.
 
---
 
## The Three Features
 
The skill is built from three features. Each owns a distinct layer.
 
| Feature | Name | What It Owns | Simple Version |
|---------|------|-------------|----------------|
| **F1** | SKILL.md | The LLM reasoning layer — reads field metadata and writes descriptions, confidence scores, and evidence citations | The LLM thinks |
| **F2** | Scripts | The deterministic pipeline — validates input, extracts metadata, merges LLM output, timestamps fields, assembles final documents | The code works |
| **F3** | Assets | The static files — templates that define what the output looks like, the sample schema for testing, and gold standard examples as the answer key | The blank forms |
 
**The rule of thumb:** If it requires judgment or language, F1 owns it. If it's deterministic logic, F2 owns it. If it's a static file the pipeline reads, F3 owns it.
 
---
 
## Repository Layout
 
All files for this skill live here. Nothing in `specs/` runs at runtime — it's planning documentation only.
 
```
synchrony-doc-automation/
│
├── .gitignore                               # Excludes output/, __pycache__/, *.pyc, .env
│
├── skills/
│   └── data-dictionary/
│       ├── SKILL.md                         ← F1: LLM instruction file
│       ├── scripts/                         ← F2: Python pipeline scripts
│       │   ├── validate_input.py
│       │   ├── extract_fields.py
│       │   ├── attach_citations.py
│       │   ├── add_timestamps.py
│       │   ├── assemble_output.py
│       │   ├── generate_qa_report.py
│       │   └── compare_output.py            (optional — runs when input is sample_schema.json)
│       └── output/                          ← Created at runtime, not checked in
│           ├── data_dictionary.md
│           ├── qa_report.md
│           ├── comparison_report.md         (optional — only when input is sample_schema.json)
│           └── intermediate/
│               ├── user_input.json
│               ├── validated_schema.json
│               ├── extracted_fields.json
│               ├── extraction_warnings.json  (conditional)
│               ├── llm_output.json
│               ├── merged_fields.json
│               └── timestamped_fields.json
│
├── assets/                                  ← F3: static files, checked in
│   ├── data_dictionary_template.md
│   ├── qa_report_template.md
│   ├── sample_schema.json
│   ├── example_data_dictionary.md
│   ├── example_qa_report.md
│   ├── kaggle_ground_truth.md               ← Ground truth for comparison (UCI dataset)
│   └── glossary_template.json               (P3 placeholder, not used for demo)
│
└── specs/                                   ← Planning docs only, not runtime
    ├── f1-skill.md-data-dictionary/
    ├── f2-scripts-data-dictionary/
    └── f3-assets-data-dictionary/
```
 
**Note on Cursor local paths:** During local development in Cursor, all skill files are located under `.cursor/skills/data_dictionary/` rather than `skills/data-dictionary/`. The canonical repository structure documented above reflects the final handoff layout for Synchrony. The `.cursor/` prefix is a Cursor-specific workspace convention and does not affect the skill's behavior or output.
 
---
 
## End-to-End Workflow
 
Here is exactly what happens from the moment a user uploads a schema to the moment they receive their files.
 
### Step 1 — User Uploads a Schema
 
The user attaches their JSON schema file to Claude and asks it to generate a data dictionary.
 
**What the user provides:**
 
```json
{
  "table_name": "credit_card_clients",
  "source_file": "sample_schema.json",
  "fields": [
    {
      "field_name": "AGE",
      "type": "INTEGER",
      "nullable": false,
      "constraints": [],
      "enums": [],
      "schema_comments": "Age of the credit card holder in years"
    },
    {
      "field_name": "PAY_0",
      "type": "INTEGER",
      "nullable": true,
      "constraints": [],
      "enums": [-2, -1, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9],
      "schema_comments": null
    },
    {
      "field_name": "col_x7",
      "type": "VARCHAR",
      "nullable": true,
      "constraints": [],
      "enums": [],
      "schema_comments": null
    }
  ]
}
```
 
The file is saved as `output/intermediate/user_input.json`.
 
---
 
### Step 2 — Validate Input (`validate_input.py`)
 
F2 checks that the schema is valid before anything else runs. If anything is wrong, the pipeline stops here with a clear error message listing every problem.
 
**What it checks:**
- Is it valid JSON?
- Does it have `table_name`, `source_file`, and `fields`?
- Does every field have a `field_name`?
**Output:** `output/intermediate/validated_schema.json` (identical to input if valid)
 
---
 
### Step 3 — Extract Fields (`extract_fields.py`)
 
F2 reads the validated schema and builds a standardized metadata record for every field. Missing optional fields get default values filled in. Every field gets a `source_path` tracing it back to the original file and table.
 
**Output:** `output/intermediate/extracted_fields.json`
 
```json
[
  {
    "field_name": "AGE",
    "type": "INTEGER",
    "nullable": false,
    "constraints": [],
    "enums": [],
    "source_path": "sample_schema.json → credit_card_clients → AGE",
    "schema_comments": "Age of the credit card holder in years",
    "table_name": "credit_card_clients"
  },
  {
    "field_name": "PAY_0",
    "type": "INTEGER",
    "nullable": true,
    "constraints": [],
    "enums": [-2, -1, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9],
    "source_path": "sample_schema.json → credit_card_clients → PAY_0",
    "schema_comments": null,
    "table_name": "credit_card_clients"
  },
  {
    "field_name": "col_x7",
    "type": "VARCHAR",
    "nullable": true,
    "constraints": [],
    "enums": [],
    "source_path": "sample_schema.json → credit_card_clients → col_x7",
    "schema_comments": null,
    "table_name": "credit_card_clients"
  }
]
```
 
If any duplicate field names exist, a warning is logged to `extraction_warnings.json`.
 
---
 
### Step 4 — LLM Generates Descriptions (F1 / SKILL.md)
 
This is the only step that involves the LLM. Claude reads `extracted_fields.json`, applies the confidence rubric defined in the SKILL.md, and returns a description, confidence score, evidence citations, and clarification flag for every field.
 
**What the LLM receives:** The extracted fields array above.
 
**What the LLM returns:** `output/intermediate/llm_output.json`
 
```json
[
  {
    "field_name": "AGE",
    "description": "The age of the credit card holder, in years.",
    "confidence": "High",
    "evidence_refs": [
      "field_name: 'AGE' is specific and unambiguous",
      "schema_comments: 'Age of the credit card holder in years' confirms meaning",
      "type: INTEGER is consistent with age as a whole number"
    ],
    "clarification_flag": false
  },
  {
    "field_name": "PAY_0",
    "description": "Repayment status for a billing period, represented as a coded integer.",
    "confidence": "Medium",
    "evidence_refs": [
      "field_name: 'PAY_0' suggests payment-related, but '_0' suffix is ambiguous",
      "enums: numeric codes [-2 through 9] — not self-descriptive, meaning of each code unclear",
      "schema_comments: absent — no additional context provided"
    ],
    "clarification_flag": false
  },
  {
    "field_name": "col_x7",
    "description": "Purpose unknown. Field name is non-descriptive and no metadata provides meaningful context.",
    "confidence": "Low",
    "evidence_refs": [
      "field_name: 'col_x7' is opaque — no semantic meaning can be inferred",
      "type: VARCHAR is broad and does not narrow the field's purpose",
      "schema_comments: absent",
      "Insufficient evidence to generate a meaningful description."
    ],
    "clarification_flag": true
  }
]
```
 
**If the LLM fails or goes offline:** F2 fills placeholder values for every field and the pipeline continues. The QA report notes that LLM content is missing.
 
**Retry logic:** F2 owns retry logic. If the LLM output fails validation, F2 retries up to 2 times (3 total attempts). If all attempts fail, graceful degradation kicks in.
 
---
 
### Step 5 — Merge and Validate (`attach_citations.py`)
 
F2 takes the LLM output and merges it with the extracted metadata. It validates the LLM's work — checking field counts, field name matching, and rubric compliance — and logs any corrections it makes automatically.
 
**What it checks and fixes:**
 
| Issue | What F2 Does |
|-------|-------------|
| LLM skipped a field | Fills placeholder values for that field |
| LLM returned wrong field name casing | Matches case-insensitively, keeps original casing |
| `"Low"` confidence but `clarification_flag` is `false` | Overrides flag to `true` |
| Description over 25 words | Keeps it, logs for awareness |
| Fields returned in wrong order | Matches by name, not position |
 
Every correction is logged in the field's `corrections` array. Nothing is silently fixed.
 
**Output:** `output/intermediate/merged_fields.json`
 
---
 
### Step 6 — Add Timestamps (`add_timestamps.py`)
 
F2 stamps every field with a `last_verified` timestamp. Every field in the same run gets the same timestamp — this records when the pipeline ran, not when the field was created in the database.
 
**Output:** `output/intermediate/timestamped_fields.json` (same as merged, plus `last_verified` on every field)
 
---
 
### Step 7 — Assemble Data Dictionary (`assemble_output.py`)
 
F2 reads the timestamped fields and fills in F3's `data_dictionary_template.md` to produce the final deliverable. The template defines the column order and layout — F2 follows it exactly.
 
**Output:** `output/data_dictionary.md`
 
---
 
### Step 8 — Generate QA Report (`generate_qa_report.py`)
 
F2 counts everything — how many fields got real descriptions, how confidence scores are distributed, which fields need human review — and produces a quality report.
 
**Output:** `output/qa_report.md`
 
---
 
### Step 9 — Compare Against Ground Truth (`compare_output.py`) — Optional
 
This step runs automatically when the input schema is `sample_schema.json`. It reads the generated `data_dictionary.md` and compares it field by field against `assets/kaggle_ground_truth.md` — the verified Kaggle UCI Credit Card dataset ground truth. It produces a side-by-side comparison showing metadata availability for each field.
 
**When it runs:** Only when input is `sample_schema.json`. Skipped for all other input schemas.
 
**Output:** `output/comparison_report.md`
 
| Column | What It Shows |
|--------|--------------|
| Field | The field name |
| Ground Truth | The Kaggle-verified description |
| Skill Output | The description generated by the skill |
| Metadata Available | ✅ Full, ⚠️ Limited, or ❌ None — based on whether the skill had enough schema metadata to produce a complete description |
 
**Metadata Available logic:**
- `✅ Full` — skill produced a detailed description with no clarification flag
- `⚠️ Limited` — skill produced a correct but brief description with no clarification flag
- `❌ None` — skill flagged the field [NEEDS CLARIFICATION] due to missing metadata
---
 
## Every File in the Pipeline
 
### Files the User Provides
 
| File | What It Is | Required Keys |
|------|-----------|---------------|
| `user_input.json` | The user's schema — saved from their upload | `table_name`, `source_file`, `fields[]`, `fields[].field_name` |
 
### Intermediate Files (Created at Runtime by F2)
 
These live in `output/intermediate/` and are never checked into the repo. They exist so you can open any one and see exactly what the pipeline did at that step.
 
| File | Created By | Read By | What It Contains |
|------|-----------|---------|-----------------|
| `validated_schema.json` | `validate_input.py` | `extract_fields.py` | Confirmed-clean copy of the input schema |
| `extracted_fields.json` | `extract_fields.py` | SKILL.md (F1), `attach_citations.py` | Standardized 8-key metadata record per field |
| `extraction_warnings.json` | `extract_fields.py` | `generate_qa_report.py` | Duplicate field name warnings (only exists if duplicates found) |
| `llm_output.json` | F1 (LLM step) | `attach_citations.py` | 5-field LLM response per field |
| `merged_fields.json` | `attach_citations.py` | `add_timestamps.py` | 14-key merged record per field — metadata + LLM output combined |
| `timestamped_fields.json` | `add_timestamps.py` | `assemble_output.py`, `generate_qa_report.py` | 15-key final record per field — everything plus `last_verified` |
 
### Final Deliverables (Created at Runtime by F2)
 
| File | Created By | What It Is |
|------|-----------|-----------|
| `output/data_dictionary.md` | `assemble_output.py` | The data dictionary — one entry per field |
| `output/qa_report.md` | `generate_qa_report.py` | The quality report — coverage stats, flags, corrections |
| `output/comparison_report.md` | `compare_output.py` | Optional — side-by-side comparison against Kaggle ground truth. Only generated when input is `sample_schema.json`. |
 
### Static Asset Files (F3 — Checked Into Repo)
 
| File | What It Is | Who Uses It |
|------|-----------|------------|
| `assets/data_dictionary_template.md` | Blueprint for the data dictionary output | `assemble_output.py` reads it at runtime |
| `assets/qa_report_template.md` | Blueprint for the QA report output | `generate_qa_report.py` reads it at runtime |
| `assets/sample_schema.json` | 25-field UCI Credit Card test input | Used for end-to-end testing |
| `assets/example_data_dictionary.md` | Gold standard — the answer key for the data dictionary | Used to verify pipeline output is correct |
| `assets/example_qa_report.md` | Gold standard — the answer key for the QA report | Used to verify QA report is correct |
| `assets/kaggle_ground_truth.md` | Kaggle-verified field descriptions for all 25 UCI fields | `compare_output.py` reads it at runtime |
| `assets/glossary_template.json` | Placeholder for future glossary feature | Not used for demo day |
 
---
 
## Key Rules That Govern the Whole Skill
 
### Confidence Levels
 
Confidence is always one of exactly three values. These are the only allowed strings.
 
| Value | Meaning | When It's Assigned |
|-------|---------|-------------------|
| `"High"` | 2+ strong signals that agree | Clear field name + schema comment, or clear name + informative type |
| `"Medium"` | 1 strong signal, OR multiple conflicting signals | Decent name but nothing to confirm it, or signals contradict each other |
| `"Low"` | 0 strong signals | Opaque field name, no comments, generic type, no constraints, no enums |
 
**Rule:** `"Low"` confidence always sets `clarification_flag` to `true`. Always.
 
### Placeholder Values
 
When the LLM fails to describe a field, F2 fills these exact values. The types never change between real and placeholder values — downstream scripts process everything the same way.
 
| Field | Placeholder Value |
|-------|------------------|
| `description` | `"[LLM did not return a description]"` |
| `confidence` | `"N/A"` |
| `evidence_refs` | `["No evidence available — LLM did not process this field"]` |
| `clarification_flag` | `true` |
| `merge_status` | `"placeholder"` |
 
### Template Authority
 
F3's templates are the single source of truth for output format. F2 scripts read the templates and follow their structure — they never hardcode column order or section names. If you want to change the output format, change the template. The QA report section names defined in `qa_report_template.md` are authoritative — `generate_qa_report.py` must use them exactly.
 
### Graceful Degradation
 
If the LLM goes offline, F2 still runs. Every field gets placeholder values, both output files are still produced, and the QA report notes that LLM content is missing. The team gets a structured starting point to manually write descriptions, or they can re-run when the LLM is back.
 
A **missing template is not** graceful degradation — it's a setup error. If `assets/data_dictionary_template.md` or `assets/qa_report_template.md` is missing, the pipeline stops with a clear error. The fix is to put the file back.
 
### Retry Logic
 
F2 owns retry logic. If the LLM output fails validation, F2 retries up to 2 times (3 total attempts). If all attempts fail, F2 fills placeholders and continues. The SKILL.md is a prompt — it does not trigger or control retries.
 
---
 
## Feature Ownership Summary
 
| What | Who Owns It | Where to Find More |
|------|------------|-------------------|
| LLM instructions and confidence rubric | F1 | `specs/f1-skill.md-data-dictionary/plan.md` |
| Input/output contract between F1 and F2 | F1 + F2 | `specs/f1-skill.md-data-dictionary/contracts/` |
| All Python pipeline scripts including compare_output.py | F2 | `specs/f2-scripts-data-dictionary/plan.md` |
| Intermediate JSON file schemas | F2 | `specs/f2-scripts-data-dictionary/data-model.md` |
| Template format and placeholder handling | F2 + F3 | `specs/f2-scripts-data-dictionary/contracts.md` |
| All files in `assets/` including kaggle_ground_truth.md | F3 | `specs/f3-assets-data-dictionary/plan.md` |
| Gold standard update process | F3 | `specs/f3-assets-data-dictionary/quickstart.md` |
| Full repository layout | Project spec | `project-spec.md` |
 
---
 
*This document is the starting point for understanding the Data Dictionary skill. It does not replace the feature-level plans — it points you to them. If something here conflicts with a feature plan, the feature plan wins.*
