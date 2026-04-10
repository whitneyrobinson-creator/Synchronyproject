# F2 Quickstart — How to Use the Pipeline

**Feature:** F2 — Orchestration Scripts (Data Dictionary)
**Project:** Synchrony Documentation Automation Skillset
**Last Updated:** April 10, 2026

---

## Two Ways to Run the Pipeline

| Mode | Who Uses It | How It Works |
|------|------------|--------------|
| **Demo Day** | End user | Upload a schema to Claude. SKILL.md runs everything automatically. You never open a terminal. |
| **Development** | Team members | Run individual scripts from the command line to build and test them. |

---

## Demo Day (Production)

### Who This Is For

Anyone using the Data Dictionary skill through Claude. No Python knowledge needed. No terminal. You just talk to Claude.

### How to Use It

1. Open Claude
2. Attach your JSON schema file
3. Ask Claude to generate a data dictionary

Example prompt:

> Here's my database schema. Please generate a data dictionary for it.
>
> [attach sample_schema.json]

That's it. Claude takes it from there.

### What Happens Behind the Scenes

You don't need to know any of this to use the skill. This is here so the team understands what Claude is doing under the hood.

SKILL.md tells Claude to run these 9 steps in order:

| Step | What Claude Does |
|------|-----------------|
| 1 | Saves the uploaded file as `output/intermediate/user_input.json` |
| 2 | Runs `validate_input.py` — checks the schema is valid |
| 3 | Runs `extract_fields.py` — pulls out individual fields |
| 4 | Reads the extracted fields and writes descriptions (the LLM step) |
| 5 | Runs `attach_citations.py` — merges descriptions with metadata |
| 6 | Runs `add_timestamps.py` — adds timestamps to every field |
| 7 | Runs `assemble_output.py` — builds the final data dictionary |
| 8 | Runs `generate_qa_report.py` — builds the quality report |
| 9 | Delivers both output files to the user |

### What the User Gets Back

Two files:

- **`data_dictionary.md`** — The data dictionary. One entry per field with descriptions, confidence levels, evidence, and flags for anything that needs human review.
- **`qa_report.md`** — The quality report. Coverage stats, flagged fields, any corrections the pipeline made, and an overall run summary.

### If Something Goes Wrong

- **Bad schema:** Claude will tell you what's wrong with your file. Fix it and try again.
- **LLM struggles with some fields:** You'll still get both files. Fields the LLM couldn't describe will have placeholder text and be flagged for review. The QA report will show which ones.
- **LLM completely fails:** You'll still get both files, but with placeholder descriptions everywhere. The QA report will show 0% coverage. Re-upload and try again, or review the placeholders manually.

The skill never silently fails. You always get output, and the QA report always tells you how the run went.

---

## Development (Building and Testing)

### Who This Is For

Team members building, testing, and debugging the scripts. You run each script one at a time so you can check its work before moving to the next one.

### What You Need

- **Python 3.11+**
- No third-party packages. Everything uses the standard library (`json`, `os`, `datetime`, `pathlib`).
- No virtual environment needed.

### Directory Structure

```
synchrony-doc-automation/
├── scripts/
│   ├── validate_input.py
│   ├── extract_fields.py
│   ├── attach_citations.py
│   ├── add_timestamps.py
│   ├── assemble_output.py
│   └── generate_qa_report.py
├── assets/
│   ├── data_dictionary_template.md    (from F3)
│   └── qa_report_template.md          (from F3)
├── output/
│   └── intermediate/
└── test_inputs/
```

If `output/` and `output/intermediate/` don't exist, create them:

```bash
mkdir -p output/intermediate
```

---

## Preparing Test Input

You need a JSON schema file to feed the pipeline. Here's a minimal one you can copy:

```json
{
  "table_name": "credit_card_clients",
  "source_file": "sample_schema.json",
  "fields": [
    {
      "field_name": "ID"
    },
    {
      "field_name": "LIMIT_BAL",
      "type": "DECIMAL",
      "nullable": false,
      "constraints": ["CHECK (LIMIT_BAL > 0)"],
      "schema_comments": "Credit limit in NT dollars"
    },
    {
      "field_name": "AGE",
      "type": "INTEGER",
      "nullable": true
    }
  ]
}
```

Save this as `test_inputs/sample_schema.json`.

**Rules for the input file:**
- Must be valid JSON
- Must have a `table_name` (non-empty string)
- Must have a `source_file` (non-empty string)
- Must have a `fields` array with at least one field
- Every field must have a `field_name`
- Everything else is optional

---

## Running the Pipeline (Step by Step)

Run each script from the project root directory. They go in order — each one reads the previous script's output.

### Step 1: Copy your test file into position

```bash
cp test_inputs/sample_schema.json output/intermediate/user_input.json
```

On demo day, SKILL.md does this automatically. During development, you do it manually.

### Step 2: Validate the input

```bash
python scripts/validate_input.py
```

**Reads:** `output/intermediate/user_input.json`
**Writes:** `output/intermediate/validated_schema.json`

If the input is bad, you'll see error messages listing every problem. Fix the input and run again.

### Step 3: Extract fields

```bash
python scripts/extract_fields.py
```

**Reads:** `output/intermediate/validated_schema.json`
**Writes:** `output/intermediate/extracted_fields.json` (+ `extraction_warnings.json` if duplicates found)

### Step 4: LLM step (three options)

This is where Claude reads the extracted fields and writes descriptions. During development, you have three choices:

**Option A: Skip it.** Don't create `llm_output.json`. The rest of the pipeline will fill placeholders. Good for testing graceful degradation.

**Option B: Fake it.** Create a `llm_output.json` by hand with test descriptions. Good for testing the merge logic without waiting for Claude.

Example fake LLM output for the 3-field test schema:

```json
[
  {
    "field_name": "ID",
    "description": "Unique identifier for each client record",
    "confidence": "High",
    "evidence_refs": ["field_name: 'ID' is a standard identifier pattern"],
    "clarification_flag": false
  },
  {
    "field_name": "LIMIT_BAL",
    "description": "Amount of credit available to the cardholder in NT dollars",
    "confidence": "High",
    "evidence_refs": ["field_name: 'LIMIT_BAL' suggests credit limit", "schema_comments confirms meaning"],
    "clarification_flag": false
  },
  {
    "field_name": "AGE",
    "description": "Age of the cardholder in years",
    "confidence": "High",
    "evidence_refs": ["field_name: 'AGE' is self-descriptive"],
    "clarification_flag": false
  }
]
```

Save as `output/intermediate/llm_output.json`.

**Option C: Use Claude.** Paste the contents of `extracted_fields.json` into Claude and ask it to respond in the expected format. This is the closest to the demo day flow.

For most testing, **skipping or faking is fine.** You're testing whether your scripts work, not whether Claude writes good descriptions.

### Step 5: Merge LLM output with extracted fields

```bash
python scripts/attach_citations.py
```

**Reads:** `output/intermediate/extracted_fields.json` + `output/intermediate/llm_output.json`
**Writes:** `output/intermediate/merged_fields.json`

If `llm_output.json` doesn't exist, all fields get placeholder values. The pipeline keeps going.

### Step 6: Add timestamps

```bash
python scripts/add_timestamps.py
```

**Reads:** `output/intermediate/merged_fields.json`
**Writes:** `output/intermediate/timestamped_fields.json`

### Step 7: Build the data dictionary

```bash
python scripts/assemble_output.py
```

**Reads:** `output/intermediate/timestamped_fields.json` + `assets/data_dictionary_template.md`
**Writes:** `output/data_dictionary.md`

**Note:** This step needs the F3 template. If the template is missing, the script stops with an error message telling you the exact file path it expected. Steps 1–6 work without F3.

### Step 8: Build the QA report

```bash
python scripts/generate_qa_report.py
```

**Reads:** `output/intermediate/timestamped_fields.json` + `output/intermediate/extraction_warnings.json` (if exists)
**Writes:** `output/qa_report.md`

**Note:** Same as Step 7 — needs the F3 template (`assets/qa_report_template.md`). If missing, the script stops with an error message.

---

## Reading the Output

### data_dictionary.md

This is the main deliverable. Open it and check:

- Is there one entry per field?
- Do the descriptions make sense?
- Are flagged fields marked for review?
- Is the source path correct for each field?

### qa_report.md

This is the quality check. Open it and look at:

| Section | What to Look For |
|---------|-----------------|
| Report Header | Correct table name and field count? |
| Coverage Statistics | What percentage of fields got real descriptions? |
| Confidence Distribution | How many fields are High, Medium, Low, N/A? Do they add up to the total? |
| Fields Requiring Clarification | Which fields need human review? |
| Merge Corrections | Did F2 auto-fix anything? Does the fix make sense? |
| Warnings | Any duplicate field names? Any LLM output issues? |
| Processing Notes | LLM available or unavailable? Clean run, partial, or failed? |

---

## Testing Specific Scenarios

### Test 1: Happy Path

Use the sample schema with 3 fields and a fake `llm_output.json`. Run all steps. Everything should work cleanly — no corrections, no warnings, 100% coverage.

### Test 2: Graceful Degradation (LLM Offline)

Run Steps 1–3, skip Step 4 (don't create `llm_output.json`), then run Steps 5–8. Both deliverables should still be produced with placeholder content. QA report should show 0% coverage.

### Test 3: Bad Input

Feed `validate_input.py` a broken file:
- Empty file
- Valid JSON but missing `table_name`
- Valid JSON but missing `source_file`
- Valid JSON but `fields` is empty
- A field missing `field_name`

The script should reject each one and tell you exactly what's wrong.

### Test 4: Duplicate Field Names

Create a schema with two fields both named `STATUS`. Run Steps 1–3. Check that `extraction_warnings.json` exists and lists the duplicate.

### Test 5: Determinism

Run the same input through the pipeline 3 times. Diff the outputs. Everything should match except `last_verified` timestamps.

---

## Debugging — When Something Looks Wrong

**The rule: Find the last file that looks right. The next script is where the bug is.**

### Step-by-Step Debugging Order

| Order | File to Check | Script That Created It | Most Common Problem |
|-------|--------------|----------------------|-------------------|
| 1 | `output/intermediate/user_input.json` | (user provided) | Schema is malformed or missing fields |
| 2 | `output/intermediate/validated_schema.json` | `validate_input.py` | Validation let something bad through |
| 3 | `output/intermediate/extracted_fields.json` | `extract_fields.py` | Wrong field count, missing metadata |
| 4 | `output/intermediate/llm_output.json` | Claude (F1) | Missing fields, wrong names, bad descriptions |
| 5 | `output/intermediate/merged_fields.json` | `attach_citations.py` | Wrong matches, unexpected placeholders |
| 6 | `output/intermediate/timestamped_fields.json` | `add_timestamps.py` | Missing timestamps (rare) |
| 7 | Template files in `assets/` | F3 | Rendering issue, not a data issue |

### Example

- `extracted_fields.json` looks good ✅
- `llm_output.json` looks good ✅
- `merged_fields.json` has wrong descriptions ❌

→ The problem is in `attach_citations.py`. That's where you focus.

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| Script says "file not found" | You're in the wrong folder, or you skipped a step | Make sure you're in the project root and ran the steps in order |
| `validate_input.py` says "not valid JSON" | File has a syntax error (missing comma, extra bracket) | Paste the file into a JSON validator online and fix the errors |
| `validate_input.py` says "missing table_name" | Top-level `table_name` key is missing or empty | Add `"table_name": "your_table"` to the JSON |
| `validate_input.py` says "missing source_file" | Top-level `source_file` key is missing or empty | Add `"source_file": "your_file.json"` to the JSON |
| `validate_input.py` says "missing field_name" | One or more fields don't have a `field_name` key | Check every object in the `fields` array |
| `extract_fields.py` shows wrong field count | Input has unexpected structure | Open `validated_schema.json` and count the fields manually |
| All descriptions are placeholders | `llm_output.json` is missing or invalid | Check if the file exists. If it does, open it and check if it's valid JSON with the right structure. |
| Some descriptions are placeholders | LLM skipped those fields | Open `llm_output.json` and check if those field names are present. Check for casing mismatches. |
| `assemble_output.py` says "Template not found" | F3 template is missing | Check that `assets/data_dictionary_template.md` exists. Steps 1–6 work without it. |
| `generate_qa_report.py` says "Template not found" | F3 template is missing | Check that `assets/qa_report_template.md` exists. Steps 1–6 work without it. |
| Timestamps are missing | `add_timestamps.py` didn't run or errored | Re-run it. Check that `merged_fields.json` exists. |
| QA report shows corrections you didn't expect | `attach_citations.py` auto-fixed something | Open `merged_fields.json` and check the `corrections` array on the affected fields. |

---

## What You Can Test Without F3

Steps 1 through 6 (`validate_input.py` through `add_timestamps.py`) don't need F3 templates. You can build and test the entire pipeline up to `timestamped_fields.json` right now.

Steps 7 and 8 (`assemble_output.py` and `generate_qa_report.py`) need the F3 templates to produce the final Markdown files. If the template is missing, the script stops with a clear error message telling you the exact path it expected. Once F3 delivers those templates, you can test the full pipeline end to end.
