# Feature Specification: F3 — Assets: Data Dictionary Generation

---

## 1. Feature Identity

| Field | Value |
|---|---|
| **Feature Name** | F3 — Assets: Data Dictionary Generation |
| **Feature Branch** | `f3-assets-data-dictionary` |
| **Created** | 2026-04-05 |
| **Status** | Draft |
| **Owner** | Whitney Robinson (PM), Sheila Green, Molly Lowell |
| **Demo Day Deadline** | May 7, 2026 |
| **Input** | F1 (SKILL.md) and F2 (Scripts) specs define what the templates must contain. The UCI Credit Card dataset (25 fields) provides the source data for sample inputs and gold standard examples. |

---

## 2. User Scenarios & Testing

> **Note:** F3 assets are static files — they do not execute. Testing F3 means validating that the templates, sample inputs, and gold standard examples are structurally correct, internally consistent, and usable by F1 and F2.

---

### US-1: Validate Pipeline Output Against the Data Dictionary Template (Priority: P1)

**As a** documentation team member,
**I want** a blank data dictionary template that defines the exact structure, columns, and formatting of the final output,
**So that** I can verify that `assemble_output.py` (F2) produces output that matches the expected structure.

**Why this priority:** The data dictionary template is the contract between F1 (SKILL.md) and F2 (Scripts). If the template is wrong, every output the pipeline produces is wrong. This is the single most critical asset.

**Independent Test:** Open `data_dictionary_template.md` and verify it contains all required columns in the correct order, has valid markdown syntax, includes header placeholders for run metadata, and defines both the main table structure and the evidence section structure.

**Acceptance Scenarios:**

| # | Given | When | Then |
|---|---|---|---|
| 1.1 | The `data_dictionary_template.md` exists in `assets/` | A team member opens the file | The template contains: (1) a header section with `{table_name}`, `{source_file}`, `{generation_date}` placeholders, (2) a markdown table with columns in the approved order (field_name, type, nullable, constraints, enums, description, confidence, last_verified), (3) an Evidence & Citations section structure with per-field source_path and evidence_refs layout, (4) a footer with a confidence level legend. |
| 1.2 | `assemble_output.py` (F2) reads the template and processes `timestamped_fields.json` | The script runs against the UCI Credit Card test data | The output `data_dictionary.md` has the same structure as the template — same column order, same section layout, same header format — with field data populated. |
| 1.3 | The template is compared against the gold standard `example_data_dictionary.md` | A team member compares both files | The example follows the exact same structure as the template. Every section in the template has a corresponding populated section in the example. |

---

### US-2: Validate QA Reporting Against the QA Report Template (Priority: P1)

**As a** documentation team member,
**I want** a blank QA report template that defines the exact structure of the coverage and quality report,
**So that** I can verify that `generate_qa_report.py` (F2) produces a QA report with all required sections and stats.

**Why this priority:** The QA report is the auditor's first artifact — it tells them at a glance whether the data dictionary is complete or has gaps. If the template is missing sections, the QA report will be incomplete.

**Independent Test:** Open `qa_report_template.md` and verify it contains all required sections: coverage stats, confidence distribution, [NEEDS CLARIFICATION] items, warnings, and processing notes.

**Acceptance Scenarios:**

| # | Given | When | Then |
|---|---|---|---|
| 2.1 | The `qa_report_template.md` exists in `assets/` | A team member opens the file | The template contains sections for: (1) Run metadata, (2) Coverage Statistics, (3) Confidence Distribution, (4) Fields Requiring Clarification, (5) Warnings, (6) Processing Notes. |
| 2.2 | `generate_qa_report.py` (F2) reads the template and processes `timestamped_fields.json` | The script runs against the UCI Credit Card test data | The output `qa_report.md` has the same section structure as the template, with all stats populated. |
| 2.3 | The template is compared against the gold standard `example_qa_report.md` | A team member compares both files | The example follows the exact same structure as the template. Every section in the template has a corresponding populated section in the example. |

---

### US-3: Test the Full Pipeline End-to-End with Sample Data (Priority: P1)

**As a** documentation team member,
**I want** a sample JSON schema file and completed gold standard examples that I can use to test the entire pipeline from input to output,
**So that** I can verify the pipeline produces correct, audit-ready output by diffing it against the known-good examples.

**Why this priority:** Without test data and expected output, there is no way to verify the pipeline works. The sample schema is the test input; the gold standard examples are the expected output. Together they form the complete test harness.

**Independent Test:** Run the full pipeline (F2 scripts + F1 SKILL.md) with `sample_schema.json` as input. Diff the output `data_dictionary.md` against `example_data_dictionary.md` and `qa_report.md` against `example_qa_report.md`. Structure should match; LLM-generated content (descriptions, evidence_refs) may vary but must be present.

**Acceptance Scenarios:**

| # | Given | When | Then |
|---|---|---|---|
| 3.1 | `sample_schema.json` exists in `assets/` with 25 fields from the UCI Credit Card dataset | `validate_input.py` (F2) processes the file | Validation passes with no errors. The file conforms to the approved JSON schema structure (Section 4.3 of F2 spec). |
| 3.2 | `sample_schema.json` includes fields with varying metadata completeness | `extract_fields.py` (F2) processes the validated schema | All 25 fields are extracted. Fields with missing optional metadata have `null` values. Extraction warnings are generated for any duplicates. |
| 3.3 | The full pipeline runs and produces `data_dictionary.md` | A team member compares the output against `example_data_dictionary.md` | Structural match: same columns, same section layout, same number of fields. Content match: field_name, type, nullable, constraints, enums, source_path, and last_verified are identical. LLM-generated fields (description, confidence, evidence_refs) are present but may differ from the example. |
| 3.4 | The full pipeline runs and produces `qa_report.md` | A team member compares the output against `example_qa_report.md` | Structural match: same sections, same stat categories. Numeric values may differ based on LLM output but all stat fields are populated. |

---

### US-4: Glossary Template Placeholder (Priority: P3)

**As a** future developer implementing glossary mapping,
**I want** a glossary template file that defines the expected JSON structure for glossary inputs,
**So that** I have a starting point when building the glossary feature without digging through other specs.

**Priority:** P3 — Not required for demo day. Placeholder for future iteration.

**Minimal spec:** `glossary_template.json` defines the JSON structure for glossary files. Not consumed by any script for demo day.

---

### Edge Cases

| # | Edge Case | Expected Behavior |
|---|---|---|
| 1 | Template is missing a required column (e.g., `confidence` column deleted) | Pipeline output will be missing that column. Gold standard diff will catch the mismatch. Template must be validated against this spec before use. |
| 2 | Gold standard example has a different structure than the blank template | Contract mismatch. The example MUST be derived FROM the template. If they diverge, the example is wrong — re-derive it from the template. |
| 3 | Sample schema has perfect metadata for all 25 fields (no missing values) | F2's Tier 2 handling (missing optional fields → null) and F1's Low confidence path are never tested. Sample MUST include fields with missing optional metadata. |
| 4 | Template column order doesn't match what `assemble_output.py` expects | Output will have misaligned columns. The template is the single source of truth — F2 must read column order from the template, not hardcode it. |
| 5 | `[NEEDS CLARIFICATION]` flag rendering | The flag is prepended to the description text in the main table (e.g., "[NEEDS CLARIFICATION] Appears to represent..."). It is NOT a separate column. Template and example must demonstrate this. |
| 6 | Evidence section field order doesn't match main table field order | Confusing for auditors. Evidence section MUST list fields in the same order as the main table. |
| 7 | QA report confidence distribution doesn't sum to total fields | Math error in `generate_qa_report.py`. The template structure makes this easy to spot — High + Medium + Low + N/A must equal Total Fields. |

---

## 3. Asset Inventory

| # | File | Purpose | Priority | Consumers | Location |
|---|---|---|---|---|---|
| 1 | `data_dictionary_template.md` | Blank template defining the structure of the final data dictionary output | P1 | `assemble_output.py` (F2), SKILL.md (F1 references it) | `assets/` |
| 2 | `qa_report_template.md` | Blank template defining the structure of the QA report | P1 | `generate_qa_report.py` (F2) | `assets/` |
| 3 | `sample_schema.json` | Test input — UCI Credit Card dataset (25 fields) in approved JSON format | P1 | All F2 scripts, F1 SKILL.md (testing) | `assets/` |
| 4 | `example_data_dictionary.md` | Completed gold standard — what correct output looks like (25 fields) | P1 | Testing/validation benchmark | `assets/` |
| 5 | `example_qa_report.md` | Completed gold standard — what correct QA report looks like | P1 | Testing/validation benchmark | `assets/` |
| 6 | `glossary_template.json` | Minimal JSON structure for future glossary mapping | P3 | Future `glossary_mapper.py` (F2) | `assets/` |

**Deferred assets (not included for demo day):**

| File | Reason |
|---|---|
| `sample_schema.yaml` | F2 does not support YAML input for demo day (F2 Open Item #6) |
| `sample_schema.ddl` | F2 does not support DDL input for demo day (F2 Open Item #6) |

---

## 4. Asset Specifications

### 4.1 data_dictionary_template.md

**Purpose:** Defines the exact Markdown structure of the final data dictionary output. This is the contract between F1 (SKILL.md) and F2 (Scripts) — both must conform to this structure.

**Template Mechanism:** Structure-based (Decision D-001). The template is a blueprint, not a fill-in-the-blank form.

- **Header:** Simple `{placeholder}` substitution for one-time run metadata
- **Field table:** Script reads the column headers from the template and generates one row per field programmatically
- **Evidence section:** Script generates one entry per field following the section structure defined in the template
- **Footer:** Copied as-is from the template (static content)

**Structure:**

#### Header Section

Contains run metadata with placeholders:

- `{table_name}` — replaced with the table name from the schema
- `{source_file}` — replaced with the source file name
- `{generation_date}` — replaced with the ISO 8601 timestamp of the run

Also includes static metadata:

- Skill version (hardcoded in template)
- A note that the document is a draft for human review

#### Main Table Section

Markdown table with the following columns in this exact order:

| # | Column | Source | Null Handling |
|---|---|---|---|
| 1 | `field_name` | F2 extracted | Always present (Tier 1 required) |
| 2 | `type` | F2 extracted | "Not specified" if null |
| 3 | `nullable` | F2 extracted | "Not specified" if null |
| 4 | `constraints` | F2 extracted | "None" if empty |
| 5 | `enums` | F2 extracted | "None" if null |
| 6 | `description` | F1 LLM output | "[LLM did not return a description]" if unavailable. Prepend "[NEEDS CLARIFICATION]" if `clarification_flag` = true. |
| 7 | `confidence` | F1 LLM output | "N/A" if unavailable |
| 8 | `last_verified` | F2 timestamp | Always present (ISO 8601) |

**Column order is authoritative.** `assemble_output.py` (F2) must read the column headers from this template and populate rows in the same order. The template is the single source of truth for column order.

#### Evidence & Citations Section

A separate section below the main table. One entry per field, in the same order as the main table. Each entry contains:

- **Field name** as a subheading
- **Source:** The `source_path` constructed by F2 (format: `{source_file} → table: {table_name} → field: {field_name}`)
- **Evidence:** The `evidence_refs` from F1 — the LLM's reasoning for the description and confidence score

This section provides the audit trail. An auditor reads the main table for the summary, then drills into this section for the reasoning behind any field they want to verify.

#### Footer Section

Static content copied as-is from the template:

- **Confidence Level Legend:** Definitions of High, Medium, Low (from FR-009)
- **Generation Note:** Statement that descriptions are AI-generated drafts for human review
- **[NEEDS CLARIFICATION] Note:** Explanation that flagged fields require human attention

---

### 4.2 qa_report_template.md

**Purpose:** Defines the exact Markdown structure of the QA report. Consumed by `generate_qa_report.py` (F2).

**Structure:**

#### Header Section

Contains run metadata with placeholders:

- `{table_name}`, `{source_file}`, `{generation_date}` (same as data dictionary)

#### Coverage Statistics Section

| Stat | Description | Source |
|---|---|---|
| Total Fields | Count of all fields in the schema | F2 count |
| Fields with Descriptions | Fields where description ≠ placeholder | F2 count |
| Fields with Citations | Fields where source_path is present | F2 count |
| Fields with Confidence Scores | Fields where confidence ≠ "N/A" | F2 count |
| Fields Flagged [NEEDS CLARIFICATION] | Fields where clarification_flag = true | F2 count |
| Completeness | (Fields with Descriptions / Total Fields) × 100 | F2 calculated |

#### Confidence Distribution Section (Decision D-005)

Breakdown of confidence scores across all fields:

| Level | Count | Percentage |
|---|---|---|
| High | Count of High confidence fields | (High / Total) × 100 |
| Medium | Count of Medium confidence fields | (Medium / Total) × 100 |
| Low | Count of Low confidence fields | (Low / Total) × 100 |
| N/A | Count of fields with no confidence score | (N/A / Total) × 100 |

**Validation rule:** High + Medium + Low + N/A must equal Total Fields.

**Note:** This section is an F3 addition (Decision D-005). F2's `generate_qa_report.py` must implement the confidence count. Estimated effort: ~5 lines of Python using existing data.

#### Fields Requiring Clarification Section

List of all fields where `clarification_flag` = true, showing:

- Field name
- Current description (including the "[NEEDS CLARIFICATION]" prefix)
- Confidence level
- Brief reason (from evidence_refs) why clarification is needed

#### Warnings Section

Includes any issues detected during the pipeline run:

- **Duplicate field names** (from `extraction_warnings.json`, if it exists)
- **LLM output validation issues** (count mismatches, name mismatches from `attach_citations.py`)
- **Missing LLM output** (if graceful degradation was triggered — note which fields are missing LLM-generated content)

If no warnings exist, this section displays "No warnings."

#### Processing Notes Section

Contextual information about the run:

- Whether the LLM was available or unavailable
- Number of fields processed
- Any batching notes (future scope — for demo day, always "All fields processed in a single pass")
- Timestamp of the run

---

### 4.3 sample_schema.json

**Purpose:** Test input file for the entire pipeline. Contains the UCI Credit Card dataset (25 fields) in the approved JSON schema format defined in F2 spec Section 4.3.

**Requirements:**

- Must conform to the F2 input contract:
  - Required: `table_name` (string), `source_file` (string), `fields` (non-empty array)
  - Required per field: `field_name` (string)
  - Optional per field: `type`, `nullable`, `constraints`, `enums`, `schema_comments`

- **Must include realistic variety** (Assumption IA-5):
  - Fields with complete metadata (all optional fields present) — e.g., `ID`, `AGE`
  - Fields with missing optional metadata (some fields set to null) — e.g., at least 2–3 fields missing `type` or `schema_comments`
  - Fields with descriptive enums — e.g., a field with `["ACTIVE", "CLOSED"]`-style values
  - Fields with cryptic numeric enums — e.g., `SEX` with `[1, 2]`
  - Fields with constraints — e.g., `ID` with `["PRIMARY KEY"]`
  - Fields with schema comments — e.g., `ID` with `"Unique client identifier"`
  - Fields with no comments — e.g., `PAY_0`
  - Fields with oddly formatted names — e.g., `default.payment.next.month`
  - A numbered series — e.g., `BILL_AMT1` through `BILL_AMT6`

- `table_name`: `"credit_card_clients"`
- `source_file`: `"sample_schema.json"`
- `fields`: 25 objects representing all columns from the UCI Credit Card dataset

**Creation approach:** AI-assisted generation from the UCI Credit Card CSV column headers, then team reviews and corrects. Team intentionally sets some optional fields to null to ensure test coverage of F2's Tier 2 handling and F1's Low confidence path.

**All 25 fields from the UCI Credit Card dataset:**

`ID`, `LIMIT_BAL`, `SEX`, `EDUCATION`, `MARRIAGE`, `AGE`, `PAY_0`, `PAY_2`, `PAY_3`, `PAY_4`, `PAY_5`, `PAY_6`, `BILL_AMT1`, `BILL_AMT2`, `BILL_AMT3`, `BILL_AMT4`, `BILL_AMT5`, `BILL_AMT6`, `PAY_AMT1`, `PAY_AMT2`, `PAY_AMT3`, `PAY_AMT4`, `PAY_AMT5`, `PAY_AMT6`, `default.payment.next.month`

---

### 4.4 example_data_dictionary.md

**Purpose:** Completed gold standard example showing what a correct, audit-ready data dictionary looks like when the template is populated with all 25 UCI Credit Card fields. This is the validation benchmark — if pipeline output matches this structure, it passes.

**Requirements:**

- Must follow the exact structure of `data_dictionary_template.md` — same header format, same column order, same evidence section layout, same footer
- Must contain all 25 fields from the UCI Credit Card dataset
- Must include realistic, high-quality content for all fields:
  - Descriptions: ≤25 words, plain language, accurate to what the field represents
  - Confidence scores: Correctly assigned per the FR-009 rubric (High/Medium/Low based on signal strength)
  - Evidence refs: Realistic reasoning explaining the confidence score
  - Source paths: Correctly formatted (`sample_schema.json → table: credit_card_clients → field: {field_name}`)
  - Timestamps: Example ISO 8601 timestamp
  - [NEEDS CLARIFICATION] flags: Correctly applied to Low confidence fields
- Must demonstrate the split layout (Decision D-002): main table + separate evidence section
- Must demonstrate null handling: "Not specified", "None", "N/A" where appropriate
- Must demonstrate `[NEEDS CLARIFICATION]` prepended to description for flagged fields

**Expected confidence distribution (approximate — finalized during creation):**

- High confidence: Fields like `ID`, `AGE`, `LIMIT_BAL` (clear names, informative types/constraints)
- Medium confidence: Fields like `PAY_0`, `SEX`, `EDUCATION` (ambiguous names or cryptic enums)
- Low confidence: At least 1–2 fields intentionally made opaque in `sample_schema.json` (missing metadata)

**Creation approach:** AI-assisted drafting, then team reviews and corrects every field. The team must sign off that the example is accurate and audit-ready before it is used as a benchmark (per Definition of Done).

---

### 4.5 example_qa_report.md

**Purpose:** Completed gold standard example showing what a correct QA report looks like for the 25-field UCI Credit Card dataset. Validates that the QA report template is correct.

**Requirements:**

- Must follow the exact structure of `qa_report_template.md` — same sections, same stat categories
- Must contain accurate stats derived from `example_data_dictionary.md`:
  - Total fields: 25
  - Fields with descriptions: 25 (assuming full LLM availability in the example)
  - Fields with citations: 25
  - Fields with confidence scores: 25
  - Fields flagged [NEEDS CLARIFICATION]: Count of Low confidence fields in the example
  - Completeness: 100% (assuming full LLM availability)
- Must include the confidence distribution breakdown (Decision D-005)
- Must list all [NEEDS CLARIFICATION] fields by name with their descriptions and reasons
- Warnings section: "No warnings" (clean run in the example)
- Processing notes: "All fields processed in a single pass. LLM available."

**Internal consistency rule:** Every number in the QA report must be verifiable by counting the corresponding items in `example_data_dictionary.md`. If the data dictionary says 3 fields are Low confidence, the QA report must say 3 fields flagged [NEEDS CLARIFICATION].

**Creation approach:** Derived directly from `example_data_dictionary.md` by counting. Can be automated or done manually — but must be cross-checked.

---

### 4.6 glossary_template.json (P3 — Placeholder)

**Purpose:** Defines the expected JSON structure for glossary input files. Not consumed by any script for demo day. Exists so that the future developer building glossary mapping (FR-011) has a starting point.

**Minimal structure (from F2 spec Section 3.7):**

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
**Fields per entry:**

- `term` (string): The field name to match against (exact match by `glossary_mapper.py`)
- `label` (string): The standardized business label

**Status:** P3. Not required for demo day. Not consumed by any script in the current pipeline. Included as a reference for future development.

---

## 5. Requirements

### 5.1 Functional Requirements

#### FR-012: Data Dictionary Output Template (Asset Side)

- The `data_dictionary_template.md` MUST define the complete structure of the final data dictionary output.
- The template MUST include all required columns: field_name, type, nullable, constraints, enums, description, confidence, last_verified.
- The template MUST define the split layout: main table for core fields, separate Evidence & Citations section for source_path and evidence_refs.
- The template MUST include header placeholders for run metadata: `{table_name}`, `{source_file}`, `{generation_date}`.
- The template MUST include a footer with confidence level legend and generation notes.
- **Boundary:** The template defines the structure. `assemble_output.py` (F2) populates it. The template does not execute.

#### FR-013: QA Report Output Template (Asset Side)

- The `qa_report_template.md` MUST define the complete structure of the QA report.
- The template MUST include sections for: Coverage Statistics, Confidence Distribution, Fields Requiring Clarification, Warnings, and Processing Notes.
- The Confidence Distribution section MUST include High, Medium, Low, and N/A counts with percentages.
- **Boundary:** The template defines the structure. `generate_qa_report.py` (F2) populates it. The template does not execute.

### 5.2 F3-Specific Requirements

#### FR-F3-001: Template Completeness

- Every column required by the project-level spec MUST be present in the data dictionary template.
- Every stat required by the F2 spec (Section 4.1, FR-013) MUST have a corresponding section in the QA report template.
- If a required column or section is missing from a template, the pipeline cannot produce valid output.

#### FR-F3-002: Example-to-Template Consistency

- `example_data_dictionary.md` MUST follow the exact structure of `data_dictionary_template.md`.
- `example_qa_report.md` MUST follow the exact structure of `qa_report_template.md`.
- If the template changes, the example MUST be updated to match. The example is derived FROM the template, never the other way around.

#### FR-F3-003: Sample Input Variety

- `sample_schema.json` MUST include fields with varying levels of metadata completeness to test both F2's Tier 2 handling (missing optional fields → null) and F1's confidence rubric across signal strengths.
- At minimum: fields with complete metadata, fields with missing `type`, fields with missing `schema_comments`, fields with cryptic enums, fields with descriptive constraints, and fields with oddly formatted names.

#### FR-F3-004: Gold Standard Accuracy

- Gold standard examples MUST contain realistic, high-quality content that an auditor would accept.
- Descriptions MUST be ≤25 words, plain language, and accurate.
- Confidence scores MUST follow the FR-009 rubric (High/Medium/Low based on signal strength).
- Evidence refs MUST include reasoning that explains the confidence score.
- [NEEDS CLARIFICATION] flags MUST be correctly applied to all Low confidence fields.
- All numbers in `example_qa_report.md` MUST be verifiable by counting items in `example_data_dictionary.md`.

#### FR-F3-005: Template as Contract

- The data dictionary template defines the column order. `assemble_output.py` (F2) MUST read the column headers from the template and populate rows in that order.
- The QA report template defines the section structure. `generate_qa_report.py` (F2) MUST follow the section order defined in the template.
- Both templates are the single source of truth for output structure. F2 scripts conform to the templates, not the other way around.

---

## 6. Success Criteria

| SC ID | What We Measure | Threshold | How We Measure |
|---|---|---|---|
| SC-F3-001 | **Template Completeness** — All required columns/sections are present in both templates | 100% of required columns and sections present | Manual: check templates against this spec (Section 4.1, 4.2) |
| SC-F3-002 | **Example-to-Template Structural Match** — Gold standard examples follow the exact structure of their corresponding templates | 100% structural match | Manual: diff example structure against template structure |
| SC-F3-003 | **Sample Input Variety** — `sample_schema.json` includes fields with varying metadata completeness | At least 3 fields with missing optional metadata, at least 1 field with cryptic enums, at least 1 field with constraints | Manual: review sample schema |
| SC-F3-004 | **Gold Standard Quality** — Example descriptions are accurate, ≤25 words, and confidence scores follow the FR-009 rubric | Team consensus that all 25 fields are accurate | Manual: team reviews all 25 fields and signs off |
| SC-F3-005 | **Internal Consistency** — QA report example stats match actual counts in the data dictionary example | All stats match exactly | Manual: cross-check every number |
| SC-F3-006 | **Pipeline Compatibility** — F2 scripts can read and populate both templates without errors | Both scripts produce valid output using the templates | Integration test: run full pipeline with F3 assets |

**Relationship to project-level success criteria:**

| Project SC | How F3 Supports It |
|---|---|
| SC-002 (≥95% field extraction) | `sample_schema.json` provides the test input. `example_data_dictionary.md` provides the expected output to verify against. |
| SC-004 (≥90% template sections populated) | Templates define what "sections" exist. If the template is complete (SC-F3-001), F2 can measure population rate against it. |
| SC-009 (≥90% structural match across runs) | Templates enforce consistent structure. If F2 reads column order from the template (FR-F3-005), structural consistency across runs is guaranteed. |
| SC-010 (≥80% vs human ground truth) | Gold standard examples ARE the human ground truth for structural comparison. |

---

## 7. Constitution Check

| Constitution Principle | F3 Compliance | How |
|---|---|---|
| **Accuracy Over Speed** | ✅ Compliant | Gold standard examples are hand-reviewed by the full team. Templates enforce all required fields — no shortcuts that omit columns for simplicity. |
| **Graceful Degradation** | ✅ Compliant | Templates handle null/placeholder values gracefully ("Not specified", "None", "N/A", "[LLM did not return a description]"). The QA report template includes sections for LLM unavailability notes. |
| **Simplicity First** | ✅ Compliant | Static Markdown and JSON files. No code, no dependencies, no databases, no UI. Assets exist before any code runs and do not change at runtime. |
| **Audit-Ready by Default** | ✅ Compliant | Split layout (D-002) separates scannable summary from detailed evidence trail. Templates include citations (source_path), confidence scores, timestamps, and [NEEDS CLARIFICATION] flags. Footer includes confidence legend for auditor reference. |
| **Never send raw PII to LLM** | ✅ Compliant | `sample_schema.json` contains only field metadata (names, types, constraints). No actual data values from the UCI dataset are included. |
| **Never send API keys or credentials** | ✅ Compliant | No credentials in any asset file. |
| **Never send full source code or raw datasets** | ✅ Compliant | Sample schema contains only structural metadata, not the raw CSV data. |
| **Never commit secrets to the repo** | ✅ Compliant | Assets contain no secrets. |
| **Never opt into LLM provider training** | ✅ Compliant | Assets are static files — they do not interact with LLM providers. |

**No violations detected.** F3's design is fully aligned with the Constitution.

---

## 8. Cross-Feature Dependencies

### How F1 and F2 Depend on F3

| Consumer | F3 Asset | Dependency Type |
|---|---|---|
| F1 — SKILL.md | `data_dictionary_template.md` | F1 references the template in its instructions ("populate the data dictionary using the template in assets/") |
| F2 — `assemble_output.py` | `data_dictionary_template.md` | F2 reads the template to determine output structure and column order |
| F2 — `generate_qa_report.py` | `qa_report_template.md` | F2 reads the template to determine QA report section structure |
| F2 — all scripts (testing) | `sample_schema.json` | F2 uses this as the test input for the entire pipeline |
| F1 + F2 (testing) | `example_data_dictionary.md`, `example_qa_report.md` | Gold standard benchmarks for validating pipeline output |

### Cross-Feature Impact from F3 Decisions

| Decision | Origin | Impacts | Action Required |
|---|---|---|---|
| D-005: Confidence distribution in QA report | F3 spec | F2 — `generate_qa_report.py` | F2 must add ~5 lines of Python to count fields by confidence level (High/Medium/Low/N/A) and calculate percentages. Uses existing data from `timestamped_fields.json`. |
| D-001: Structure-based template mechanism | F3 spec | F2 — `assemble_output.py` | F2 must read column order from the template's table header row, not hardcode it. |
| D-002: Split layout | F3 spec | F2 — `assemble_output.py` | F2 must generate two sections: the main table AND the evidence section. Slightly more complex than a single-table approach. |

### F2 Open Items Resolved by F3

| F2 Open Item | F3 Deliverable | Status |
|---|---|---|
| #1: `data_dictionary_template.md` design | Defined in Section 4.1 | ✅ Resolved by this spec |
| #2: `qa_report_template.md` design | Defined in Section 4.2 | ✅ Resolved by this spec |
| #3: `sample_schema.json` creation | Defined in Section 4.3 | ✅ Resolved by this spec |

---

## 9. Approved Decisions & Assumptions

### Decisions

| # | Decision | Rationale | Date |
|---|---|---|---|
| D-001 | **Template mechanism: Structure-based** | Template is a blueprint. Script reads structure and generates rows programmatically. Simple `{placeholder}` for header metadata only. Avoids fragility of full placeholder-based approach for repeating field rows. | 2026-04-05 |
| D-002 | **Data dictionary layout: Split layout** | Main table for core fields (scannable). Separate Evidence & Citations section for source_path and evidence_refs (detailed reasoning). Matches auditor workflow: scan first, drill down second. Avoids unreadable 12-column markdown table. | 2026-04-05 |
| D-003 | **Sample inputs: JSON only for demo day** | F2 only supports JSON for demo day. YAML and DDL samples deferred to match F2's scope. Avoids delivering files nobody can use. | 2026-04-05 |
| D-004 | **Gold standard examples: Full 25 fields** | Complete benchmark enables definitive validation. Team diffs pipeline output against the example. A subset would leave ambiguity about untested fields. | 2026-04-05 |
| D-005 | **QA report: Add confidence distribution** | Auditors need to see at a glance how much of the output they can trust. Trivial to implement (~5 lines of Python in F2). Uses existing data. | 2026-04-05 |
| D-006 | **Glossary template: Define minimal JSON structure** | Gives future P3 developer a starting point. Two extra lines in the spec. Low effort, high future value. | 2026-04-05 |
| D-007 | **Omit `glossary_label` column for demo day** | Empty column in a demo looks like a bug. Add when P3 is built. | 2026-04-05 |

### Approved Assumptions

| # | Assumption | Mitigation |
|---|---|---|
| IA-1 | Gold standard examples are AI-assisted drafts, reviewed and corrected by the full team | Definition of Done requires all team members to review and sign off. AI drafts speed creation; human review ensures accuracy. |
| IA-2 | Template defines column order; F2 reads and respects it | FR-F3-005 explicitly states this. If F2 hardcodes column order, integration test (SC-F3-006) will catch the mismatch. |
| IA-3 | `[NEEDS CLARIFICATION]` is prepended to description text, not a separate column | Consistent with F2 spec US-4 scenario 4.3. Template and example must demonstrate this rendering. |
| IA-4 | QA report is a standalone file, not appended to the data dictionary | F2 has separate scripts producing separate files. Auditor receives two documents. |
| IA-5 | Sample schema includes realistic variety (missing optional fields) | FR-F3-003 requires this. SC-F3-003 measures it. Without variety, F2 Tier 2 and F1 Low confidence paths are untested. |

---

## 10. Open Items

| # | Item | Status | Notes |
|---|---|---|---|
| 1 | **Gold standard benchmark source** | ⏳ PARKING LOT | Whitney wants to compare output against real-world data dictionary examples to ensure quality is competitive with industry standards. Questions: Where to find professional data dictionary examples? Should the gold standard be validated against an external benchmark? Goal: ensure F3 output quality exceeds expectations. To be revisited after spec is drafted. |
| 2 | **F2 integration testing** | ⏳ PENDING | F3 assets must be tested with F2 scripts to confirm compatibility (SC-F3-006). Cannot be completed until both F2 scripts and F3 assets are built. |
| 3 | **Success criteria thresholds for SC-F3-004** | ⏳ TBD | "Team consensus" is qualitative. May need a more specific rubric after baseline testing. |
| 4 | **Exact confidence distribution in gold standard** | ⏳ TBD | Approximate distribution noted in Section 4.4. Finalized when `sample_schema.json` and `example_data_dictionary.md` are created. |
