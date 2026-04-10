# F3 — Assets: Data Dictionary Generation — Research

**Feature Branch**: `f3-assets-data-dictionary`
**Created**: 2026-04-10
**Status**: Draft
**Owner**: Whitney Robinson (PM), Sheila Green, Molly Lowell

---

## What This Document Is

This is a record of what we learned and decided while planning F3. It's organized by topic so you can find answers quickly. If you want to know *why* something in F3 is designed the way it is, the answer should be here.

This is NOT a literature review. It's the team's notes from the planning process.

---

## 1. What F3 Provides and Why

F3 is 6 files in a folder. That's it. No code, no scripts, no instructions for Claude. Just files that the other two features (F1 and F2) need to do their jobs.

### The 6 Files

| File | What It Is | Who Uses It | Why It Exists |
|------|-----------|-------------|---------------|
| `data_dictionary_template.md` | Blank form that defines what the final data dictionary looks like | F2's `assemble_output.py` reads it to build the output | Without this, F2 doesn't know what format to produce |
| `qa_report_template.md` | Blank form that defines what the QA report looks like | F2's `generate_qa_report.py` reads it to build the report | Without this, F2 doesn't know what sections the QA report needs |
| `sample_schema.json` | Test input — 25 fields from the UCI Credit Card dataset | F2 scripts use it as test data for the whole pipeline | You can't test a pipeline without test data |
| `example_data_dictionary.md` | The answer key — what correct output looks like for all 25 fields | The team uses it to check if the pipeline got the right answer | If you don't know what "correct" looks like, you can't tell if the output is wrong |
| `example_qa_report.md` | The answer key for the QA report | The team uses it to check if the QA report has the right numbers | Same reason — you need an answer key |
| `glossary_template.json` | Placeholder for a future feature (glossary mapping) | Nobody right now — it's for whoever builds the P3 glossary feature later | Gives the future developer a starting point so they don't have to dig through specs |

### Three Roles

These 6 files serve three purposes:

1. **Structural contracts** — The templates are the single source of truth for what the output looks like. F2 scripts read the templates and follow their structure. If you want to change the output format, you change the template — not the code.

2. **Test harness** — The sample schema is the test input. The gold standards are the expected output. Together they form a complete test: "feed this in, check if what comes out matches."

3. **LLM teaching material** — The team pulls 3 example fields from the gold standard and pastes them into F1's SKILL.md as worked examples. This teaches Claude what good output looks like. (This is a copy-paste during authoring, not a runtime file read — see Section 3 for details.)

---

## 2. How F3 Relates to F1 and F2

### F3 → F2 (Runtime Dependency)

F2 scripts directly read F3 files during pipeline execution:

| F2 Script | F3 File It Reads | What It Does With It |
|-----------|-----------------|---------------------|
| `assemble_output.py` | `data_dictionary_template.md` | Reads the template structure, fills in field data, produces the final `data_dictionary.md` |
| `generate_qa_report.py` | `qa_report_template.md` | Reads the template structure, fills in stats, produces the final `qa_report.md` |
| All scripts (testing) | `sample_schema.json` | Uses it as test input to run the pipeline end-to-end |

**If a template file is missing, F2 stops with an error.** This is a setup problem, not a runtime problem. The fix is "put the file back." (See Section 5 for the full reasoning.)

### F3 → F1 (Authoring Reference Only)

F1's SKILL.md does NOT read any F3 files at runtime. The connection is:

1. The team creates the gold standard (`example_data_dictionary.md`) with all 25 fields
2. The team picks 3 representative fields (one High, one Medium, one Low confidence)
3. The team copies those 3 fields into the SKILL.md as worked examples
4. At runtime, Claude reads ONLY the SKILL.md — it never opens any F3 file

**What this means:** Renaming or moving F3 files has zero impact on F1. But if the gold standard changes, someone needs to check whether the 3 worked examples in the SKILL.md need updating too. This is a manual step.

### F3 → Testing (Gold Standards as Answer Keys)

The gold standards are used for end-to-end testing (SC-010 in the project spec):

1. Run the pipeline with `sample_schema.json` as input
2. Compare the output `data_dictionary.md` against `example_data_dictionary.md`
3. Compare the output `qa_report.md` against `example_qa_report.md`
4. Structure should match exactly. LLM-generated content (descriptions, evidence) may vary but must be present.

---

## 3. Template Design Decisions

### D-001: Structure-Based Templates (Not Fill-in-the-Blank)

**What we chose:** The template is a blueprint. F2 reads the structure (column headers, section layout) and generates rows programmatically. Only the header metadata uses simple `{placeholder}` substitution (like `{table_name}`, `{source_file}`, `{generation_date}`).

**What we rejected:** A full placeholder-based approach where every cell in the template has a placeholder like `{{field_1_name}}`, `{{field_1_type}}`, etc.

**Why:** The fill-in-the-blank approach breaks when the number of fields changes. If the template has placeholders for 25 fields and someone runs it on a 50-field schema, the template can't handle it. The structure-based approach works for any number of fields because F2 generates rows dynamically based on the data.

### D-002: Split Layout (Main Table + Evidence Section)

**What we chose:** The data dictionary has two parts:
- A **main table** with the core fields (field_name, type, nullable, constraints, enums, description, confidence, last_verified) — designed for scanning
- A separate **Evidence & Citations section** below the table with per-field source_path and evidence_refs — designed for drill-down

**What we rejected:** A single wide table with all 12+ columns in one row.

**Why:** A 12-column Markdown table is unreadable. The columns get squished, text wraps awkwardly, and nobody can scan it. The split layout matches how auditors actually work: scan the summary table first, then drill into the evidence for specific fields they want to verify.

### D-005: Confidence Distribution in QA Report

**What we chose:** The QA report includes a Confidence Distribution section showing counts and percentages of High, Medium, Low, and N/A fields.

**Why:** Auditors need to see at a glance how much of the output they can trust. If 80% of fields are High confidence, the data dictionary is probably solid. If 60% are Low, it needs significant human review. This section gives that picture in 4 lines.

**Impact on F2:** `generate_qa_report.py` needs ~5 lines of Python to count fields by confidence level. Uses data that already exists in `timestamped_fields.json`.

### D-007: No Glossary Column for Demo Day

**What we chose:** The data dictionary template does NOT include a `glossary_label` column.

**Why:** An empty column in a demo looks like a bug. The glossary feature is P3 (not built for demo day). When P3 is built, the column gets added to the template at that point.

---

## 4. Gold Standard Design Decisions

### What "Gold Standard" Means

The gold standard is the answer key. It's a completed data dictionary and QA report that represent what correct output looks like for the UCI Credit Card dataset. The team writes it by hand, reviews every field, and signs off that it's accurate.

### D-004: Full 25 Fields (Not a Subset)

**What we chose:** The gold standard includes all 25 fields from the UCI Credit Card dataset.

**What we rejected:** A subset (e.g., 10 representative fields).

**Why:** A subset leaves ambiguity. If the pipeline produces wrong output for a field that's not in the gold standard, you can't catch it. With all 25 fields, every pipeline output can be compared against the answer key. No gaps.

### How the Gold Standard Is Created

1. **AI-assisted drafting** — Use Claude to generate a first draft of descriptions, confidence scores, and evidence
2. **Team reviews every field** — Check each description against the Kaggle dataset documentation. Is it accurate? Is it ≤25 words? Does the confidence score follow the rubric?
3. **Team corrects mistakes** — Fix anything the AI got wrong
4. **Team signs off** — All three team members confirm the gold standard is accurate (Definition of Done)

**Important:** The gold standard is NOT just whatever Claude produces. Claude writes the first draft to save time, but the team is responsible for the final content. If Claude writes a bad description, the team fixes it.

### How the Gold Standard Is Used for Scoring

The gold standard supports SC-010 from the project spec: "AI-generated documentation scores ≥80% on an evaluation rubric when compared against human-written ground truth documentation."

The comparison works like this:
- **Structural match:** Does the pipeline output have the same columns, sections, and field count as the gold standard? (Should be 100% — the template enforces this.)
- **Content match for deterministic fields:** field_name, type, nullable, constraints, enums, source_path, last_verified should be identical between pipeline output and gold standard.
- **Content match for LLM fields:** description, confidence, evidence_refs will vary between runs. The team scores these manually against the gold standard using a rubric (accuracy, completeness, appropriate confidence level).

### Expected Confidence Distribution

Based on the UCI Credit Card dataset's field characteristics:

| Level | Expected Fields | Why |
|-------|----------------|-----|
| High | `ID`, `AGE`, `LIMIT_BAL`, and other clear fields | Unambiguous names + informative types/constraints |
| Medium | `PAY_0`, `SEX`, `EDUCATION`, `MARRIAGE` | Ambiguous names or cryptic numeric enums |
| Low | At least 1-2 fields intentionally made opaque in the sample schema | Missing metadata — tests the Low confidence path |

Exact distribution finalized when the gold standard is created.

### Gold Standard QA Report — Only One Needed, Ever

The QA report gold standard (`example_qa_report.md`) tests whether F2's `generate_qa_report.py` counts and formats correctly. It tests the script, not the data.

Once you've proven the script works with the UCI gold standard, it works for any schema. You do NOT need a new QA report gold standard when you add a new schema. The UCI version stays permanently as the regression test.

---

## 5. Sample Schema Decisions

### Why the UCI Credit Card Dataset

**Dataset:** UCI Default of Credit Card Clients (Kaggle)
**Fields:** 25 columns
**Table name:** `credit_card_clients`

**Why this dataset:**

It has the right mix of easy, medium, and hard fields:

| What It Has | Example | What It Tests |
|-------------|---------|---------------|
| Clear, unambiguous field names | `AGE`, `ID` | High confidence path |
| Ambiguous field names | `PAY_0`, `SEX` | Medium confidence path |
| Cryptic numeric enums | `SEX` with values `[1, 2]` | Enum handling when values aren't self-descriptive |
| Numbered series | `BILL_AMT1` through `BILL_AMT6` | Cross-field consistency (FR-F1-002) |
| Oddly formatted names | `default.payment.next.month` | Field names with dots/special characters |
| Fields with schema comments | Some fields have comments | Tests the "comment agrees with name" signal |
| Fields without schema comments | Some fields have `null` comments | Tests the "no comment" signal |

**Why not a different dataset:** We needed something public (no NDA issues), financial (relevant to Synchrony), and small enough to create a full gold standard by hand (25 fields is manageable). The UCI Credit Card dataset checks all three boxes.

### Intentional Metadata Gaps

The sample schema intentionally leaves some optional fields as `null` for at least 2-3 fields. This is on purpose — it tests:

- F2's handling of missing optional fields (fills in defaults like `"UNKNOWN"` for type)
- F1's Low confidence path (fewer signals → lower confidence)
- The template's null rendering ("Not specified", "None", "N/A")

If every field had perfect metadata, we'd never test the paths that handle incomplete data. Real-world schemas almost always have gaps.

### `source_file` as a Required Field

The sample schema includes a top-level `source_file` key (e.g., `"sample_schema.json"`). This tells the pipeline where the data originally came from and gets used to build the `source_path` for every field in the output.

**Why it's required (not derived from the filename):** If someone renames the file on disk, the source_path would be wrong. Writing it inside the file is more reliable and supports the audit trail (Constitution Principle 4).

**Note:** This is a contract mismatch with F2's current input contract, which only requires `table_name` and `fields`. F2's docs need to be updated to add `source_file` as a third required top-level key.

---

## 6. Alternative Approaches We Considered

### Template Mechanism — Fill-in-the-Blank vs. Structure-Based

| Approach | How It Works | Why We Rejected It |
|----------|-------------|-------------------|
| **Fill-in-the-blank** | Every cell has a placeholder: `{{field_1_name}}`, `{{field_1_type}}`, etc. | Breaks when field count changes. A 25-field template can't handle a 50-field schema. |
| **Structure-based** (chosen) | Template defines the column headers and section layout. F2 generates rows dynamically. | Works for any number of fields. |

### Data Dictionary Layout — Single Table vs. Split

| Approach | How It Works | Why We Rejected It |
|----------|-------------|-------------------|
| **Single wide table** | All columns (including source_path and evidence_refs) in one table | 12+ columns in Markdown is unreadable. Evidence_refs can be long text — it destroys the table layout. |
| **Split layout** (chosen) | Main table for scanning + separate Evidence section for drill-down | Matches auditor workflow. Keeps the main table clean and readable. |

### Gold Standard Scope — Subset vs. Full

| Approach | How It Works | Why We Rejected It |
|----------|-------------|-------------------|
| **Subset** (10-15 fields) | Gold standard covers representative fields only | Leaves gaps — can't verify fields not in the subset |
| **Full** (all 25 fields, chosen) | Gold standard covers every field | Complete verification. No ambiguity. |

### Template Fallback — Hardcoded Backup vs. Hard Stop

| Approach | How It Works | Why We Rejected It |
|----------|-------------|-------------------|
| **Hardcoded fallback** | F2 has a built-in backup template if the real one is missing | Creates two sources of truth. If someone updates the real template but forgets the backup, the pipeline silently uses the wrong format. |
| **Dump raw JSON** | F2 outputs the raw data instead of formatted Markdown | Raw JSON isn't a data dictionary. An auditor can't read it. |
| **Hard stop with error** (chosen) | F2 prints "Template not found at [path]" and stops | Instantly fixable. Missing template is a setup problem, not a runtime problem. |

---

## 7. F3's Role in Testing

### What F3 Provides for Testing

| Asset | Testing Role |
|-------|-------------|
| `sample_schema.json` | **Test input** — feed this into the pipeline to run an end-to-end test |
| `example_data_dictionary.md` | **Expected output** — compare pipeline output against this to check correctness |
| `example_qa_report.md` | **Expected QA output** — compare QA report against this to check the script's counting logic |
| Templates | **Structural contract** — if F2 reads the template correctly, output structure is guaranteed consistent across runs |

### How End-to-End Testing Works (SC-010)

1. Feed `sample_schema.json` into the pipeline
2. Pipeline produces `data_dictionary.md` and `qa_report.md`
3. Compare structure: same columns? Same sections? Same field count? (Should be identical.)
4. Compare deterministic content: field_name, type, nullable, constraints, enums, source_path, last_verified (Should be identical.)
5. Score LLM content: description, confidence, evidence_refs (Will vary — team scores manually against gold standard.)

### What F3 Does NOT Test

F3 assets don't test themselves. There's no automated test framework for "is this template correct?" The team verifies F3 assets manually using the 6 success criteria (SC-F3-001 through SC-F3-006). The most important integration test — SC-F3-006, "can F2 scripts actually read these files?" — is blocked until F2 scripts are built.

---

## 8. What We Learned from External Research

During planning, we looked at real-world data dictionaries and regulatory frameworks to make sure our design wasn't missing anything obvious. Here's what we found and how it influenced our decisions.

### FDIC BankFind Data Dictionary (Primary Benchmark)

We analyzed the FDIC's BankFind Suite variable definitions — 2,643 variables from a real federal financial dataset.

**Key findings that shaped our design:**

| What We Found | What It Means for F3 |
|---------------|---------------------|
| Median definition length is 36 words; 73.5% of definitions exceed 25 words | Our 25-word limit (FR-007) is intentionally stricter. The extra context goes in `evidence_refs` instead of longer descriptions. |
| Cryptic abbreviations are common (`ABCUBKR`, `INSTCRCD`) | Validates our confidence scoring — opaque field names are a real-world problem, not just an edge case we invented. |
| Definitions are self-contained (no cross-references needed) | Our descriptions must stand alone too. Each field's description should make sense without reading other fields. |
| FDIC has no confidence scoring, no clarification flags, no source traceability | These are features we ADD beyond what even a federal benchmark provides. Our system is more transparent than the FDIC's. |

### Regulatory Validation

We checked our design against financial regulations to confirm we're not missing required fields:

| Regulation | What It Requires | How Our Design Covers It |
|-----------|-----------------|-------------------------|
| **BCBS 239** (Risk Data Aggregation) | Consistent definitions, completeness tracking, timeliness | `description` + `evidence_refs`, QA report coverage stats, `last_verified` timestamp |
| **12 CFR 360.9** (Resolution Planning) | Unique identifier, plain-language definition, source system | `field_name`, `description`, `source_path` |
| **OCC SR 11-7** (Model Risk Management) | Model inputs must be auditable | `confidence`, `evidence_refs`, `clarification_flag` make LLM reasoning transparent |

**What we're NOT covering (post-handoff):** Business owner, privacy classification, data lineage — these require organizational knowledge that doesn't exist in schema metadata. Synchrony adds these after handoff.

---

*This document captures what we learned and decided during F3 planning. If a decision isn't explained here, check the F3 spec (specs/f3-assets-data-dictionary/spec.md) for the formal requirements.*
