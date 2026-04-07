# research.md
## F3 · Assets & Data Dictionary — Phase 0 Research
**Status:** Draft — Pending Approval | **Date:** 2026-04-07

---

## 1. Purpose of This Document

This document records the Phase 0 research that informed the design of F3's static asset files:
the data dictionary template, sample schema, and gold standard examples. It answers
three questions:

1. **What does a high-quality financial data dictionary look like?** (benchmark analysis)
2. **What regulatory frameworks require or validate our data dictionary approach?** (compliance grounding)
3. **What did analysis of the actual FDIC dataset reveal?** (empirical grounding)

This research is the factual foundation for `data-model.md`, `quickstart.md`, and `contracts/`.

---

## 2. What F3 Actually Is

F3 is **not** a governance platform. It is a set of static asset files that support the
data dictionary generation pipeline:

- **`data_dictionary_template.md`** — Defines the Markdown structure for the final data dictionary output (consumed by F2's `assemble_output.py`)
- **`qa_report_template.md`** — Defines the Markdown structure for the QA report (consumed by F2's `generate_qa_report.py`)
- **`sample_schema.json`** — Provides realistic test input for the pipeline (consumed by F2's `validate_input.py`)
- **Gold standard examples** — Hand-curated reference entries used as few-shot anchors in F1's SKILL.md to ensure LLM consistency

F3 does not manage workflows, approvals, SLAs, or change control. Those are post-handoff
enhancements documented in the Constitution (Section 4: Handoff Boundary).

---

## 3. Primary Benchmark: FDIC BankFind Data Dictionary

### 3.1 Source

- **Publisher:** Federal Deposit Insurance Corporation (FDIC)
- **Dataset:** BankFind Suite — Variable Definitions
- **File Analyzed:** `Definitions_4_7_2026.csv` (exported April 7, 2026)
- **Analysis Method:** Primary data analysis using Python (pandas) on the actual exported file
- **Reference URL:** https://banks.data.fdic.gov/docs/

### 3.2 Dataset Characteristics

| Metric | Value |
|--------|-------|
| Total variables | 2,643 |
| Variables with definitions | 2,643 (100%) |
| Variables without definitions | 0 (0%) |
| Mean definition length | 47 words |
| Median definition length | 36 words |
| Min definition length | 2 words |
| Max definition length | 858 words |
| 25th percentile | 25 words |
| 75th percentile | 55 words |

### 3.3 Definition Length Distribution

| Range | Count | % of Total |
|-------|-------|-----------|
| 1–10 words | 124 | 4.7% |
| 11–25 words | 576 | 21.8% |
| 26–50 words | 1,170 | 44.3% |
| 51–100 words | 550 | 20.8% |
| 101+ words | 223 | 8.4% |

**Definitions ≤ 25 words:** 700 (26.5%)
**Definitions > 25 words:** 1,943 (73.5%)

**Key finding:** Only 26.5% of FDIC definitions are ≤ 25 words. The median definition is
36 words. Our system enforces a ≤25 word hard ceiling on the `description` field (F1 spec
FR-007). This is intentionally stricter than the FDIC's practice — our design captures
additional context in the separate `evidence_refs` field rather than writing longer
descriptions. The FDIC data validates that `evidence_refs` is necessary: complex financial
variables often require more context than a single concise sentence can provide.

### 3.4 Report Type Coverage

| Report Type | Variables | % Coverage |
|-------------|-----------|-----------|
| R&C (Reports of Condition & Income) | 2,345 | 88.7% |
| BSC (Balance Sheet & Capital) | 228 | 8.6% |
| FIB (Financial Institution Bulletin) | 145 | 5.5% |
| SOD (Summary of Deposits) | 79 | 3.0% |
| ADS (Application Data System) | 73 | 2.8% |
| F&A (Failures & Assistance) | 6 | 0.2% |

**Key finding:** R&C (Call Report) variables dominate the dataset at 88.7%. This aligns
with our project's focus on financial institution data and validates using the FDIC
BankFind dataset as the primary benchmark for a Synchrony-facing prototype.

### 3.5 Structural Patterns Observed in FDIC Definitions

| FDIC Pattern | Example | How It Informed Our Design |
|--------------|---------|---------------------------|
| Definitions begin with the concept, not the variable name | "The sum of..." not "ABCUBKR is the sum of..." | Our gold standard template requires descriptions to explain the concept, not echo the field name |
| Units and scope are explicit | "(YTD, %)" prefix in title | Our system captures type and constraints as separate metadata fields (F2 input contract) |
| Multi-part variables use consistent phrasing | "Includes / Excludes" structure | F1 spec FR-F1-002 requires cross-field consistency for related fields |
| Definitions are self-contained | No cross-references required to understand a single entry | Our ≤25 word descriptions must stand alone; additional context goes in `evidence_refs` |
| Cryptic abbreviations are common | `ABCUBKR`, `INSTCRCD`, `INSSAVE` | Validates our confidence scoring approach — opaque names are a real-world problem, not an edge case |

### 3.6 Field Coverage Comparison: FDIC vs. Our System

| What FDIC Includes | What Our System Includes | Comparison |
|--------------------|--------------------------|-----------|
| Variable name | `field_name` | Match |
| Title (human-readable label) | `description` (≤25 words) | Match — ours has a word limit for consistency |
| Definition (detailed) | `evidence_refs` (reasoning + evidence) | Our `evidence_refs` serves a similar role but also explains *why* the description was chosen |
| Report type flags (FIB, BSC, SOD, etc.) | Not in scope | FDIC has this; ours doesn't (post-handoff enhancement) |
| Data type | `type` | Match |
| Enum/coded values | `enums` | Match |
| Constraints | `constraints` | We include this; FDIC doesn't explicitly |
| Nullable | `nullable` | We include this; FDIC doesn't explicitly |
| **Confidence scoring** | `confidence` (High/Medium/Low) | **We have this; FDIC doesn't** |
| **Clarification flags** | `clarification_flag` | **We have this; FDIC doesn't** |
| **Source traceability** | `source_path` | **We have this; FDIC doesn't** |
| **Verification timestamp** | `last_verified` | **We have this; FDIC doesn't** |

---

## 4. Secondary Benchmarks

### 4.1 FDIC Community Banking Research — Variable Reference Notes

**Source:** FDIC Community Banking Study, Technical Appendix
**URL:** https://www.fdic.gov/resources/community-banking/

**Relevant finding:** The FDIC's own community banking researchers annotate variables with
calculation notes, data source references, and known data quality caveats. This practice
validates our `evidence_refs` field (F1 spec FR-008) — the idea that definitions alone are
insufficient and that reasoning/provenance must accompany each entry is established practice
at the federal level.

### 4.2 12 CFR Part 360 — Resolution Planning Data Standards

**Source:** 12 CFR Part 360.9 — Large-Bank Resolution Plans
**URL:** https://www.ecfr.gov/current/title-12/part-360

**Relevant finding:** Part 360 requires covered institutions to maintain data dictionaries
for their Critical Operations and Core Business Lines. The regulation explicitly requires:

- A unique identifier for each data element
- A plain-language definition
- The source system and business owner
- Mapping to regulatory reports

**Design impact:** This validates our core field set (`field_name`, `description`,
`source_path`). The regulation also requires `business owner` and `source system` fields —
these are acknowledged as post-handoff enhancements because our prototype uses schema
metadata rather than live institutional data (Constitution Section 4: Handoff Boundary).

---

## 5. Regulatory Framework Mapping

These regulations don't define our requirements — they validate that the features we
already designed address real regulatory expectations.

### 5.1 BCBS 239 — Principles for Effective Risk Data Aggregation and Reporting

**Source:** Basel Committee on Banking Supervision, January 2013
**URL:** https://www.bis.org/publ/bcbs239.pdf

| BCBS 239 Principle | What It Requires | Which Feature Addresses It | Spec Location |
|--------------------|------------------|---------------------------|---------------|
| Principle 3: Accuracy and Integrity | "Maintain a dictionary of the concepts used" with consistent definitions | Gold standard examples establish accuracy baseline; `evidence_refs` provides traceable reasoning | F1 spec FR-008, FR-009 |
| Principle 4: Completeness | All material risk data must be captured | QA report tracks completeness percentage; every input field gets an output entry | F2 spec FR-013, SC-F2-001 |
| Principle 5: Timeliness | Documentation must be current | `last_verified` timestamp on every field; one timestamp per pipeline run | F2 spec FR-010 |

**Boundary note:** BCBS 239 Principles 1–2 (Governance, Data Architecture) and Principles
6–14 (Reporting, Supervisory Review) are enterprise-level concerns beyond this prototype's
scope.

### 5.2 OCC SR 11-7 — Model Risk Management

**Source:** Board of Governors of the Federal Reserve System
**URL:** https://www.federalreserve.gov/supervisionreg/srletters/sr1107.htm

**Relevant guidance:** Model inputs must be well-defined, consistently applied, and subject
to validation.

**Design impact:** Validates our requirement that every field processed by the LLM (F1)
must produce traceable output — `description`, `confidence`, `evidence_refs`, and
`clarification_flag` — so that the LLM's reasoning can be audited. This is the "white box"
principle: the LLM's work must be explainable, not opaque.

### 5.3 FFIEC Information Technology Examination Handbook

**Source:** Federal Financial Institutions Examination Council
**URL:** https://ithandbook.ffiec.gov/

**Relevant guidance:** Financial institutions must maintain metadata repositories that
describe data elements, their sources, and their usage.

**Design impact:** Validates our decision to include `source_path` (tracing every field
back to its origin file, table, and position) and the QA report (providing coverage
statistics that an examiner could review).

---

## 6. Cross-Sector Context

The following standards were examined for structural patterns only — not as primary
benchmarks for our financial services project.

| Standard | What It Does Well | What Pattern We Observed |
|----------|-------------------|------------------------|
| **HL7 FHIR** (Healthcare) | Separates "short" definitions from "full" definitions for different audiences | Validates our split layout: concise `description` (≤25 words) for scanning + detailed `evidence_refs` for drill-down |
| **FEMA Data Lexicon** | Includes "reason for requested term" — provenance for why a data element exists | Conceptually similar to our `evidence_refs`; validates that reasoning-behind-definitions is practiced in government |
| **NASA PDS4** (Planetary Data System) | Version-controlled metadata with unique identifiers and timestamps | Validates our `last_verified` timestamp and the principle that metadata must be versioned |
| **HHS TAGGS** (Health & Human Services) | Plain-language descriptions for 500K+ financial assistance awards | Validates that plain-language descriptions work at scale in government financial data |
| **Frictionless Data** (Open Source) | JSON-based schema inference and description | Validates our JSON-first input format for schema files |

---

## 7. Design Decisions Traceable to This Research

| Decision | Research Source | Location in Specs |
|----------|----------------|-------------------|
| `description` field: ≤25 words, hard ceiling | FDIC median is 36 words, but 73.5% exceed 25 words → validates need for separate `evidence_refs` to capture overflow context | F1 spec FR-007 |
| `evidence_refs` field required for every entry | FDIC Community Banking annotations + BCBS 239 Principle 3 (traceable definitions) + SR 11-7 (auditable model inputs) | F1 spec FR-008 |
| Confidence scoring: High / Medium / Low based on signal count | BCBS 239 Principle 3 (accuracy standard) + OCC expectation that uncertain items are flagged | F1 spec FR-009 |
| `source_path` citation on every field | 12 CFR 360.9 (source system required) + FFIEC (metadata must trace to source) | F2 spec FR-006 |
| `last_verified` timestamp on every field | BCBS 239 Principle 5 (timeliness) + NASA PDS4 (version-controlled metadata) | F2 spec FR-010 |
| QA report with completeness percentage | BCBS 239 Principle 4 (completeness) | F2 spec FR-013 |
| Gold standard examples as few-shot anchors | FDIC dataset provides real-world variety across clear, ambiguous, and opaque field names | F1 spec Section 3 (Worked Examples) |
| Post-handoff: steward, privacy classification, lineage | 12 CFR 360.9 requires these for covered institutions; outside prototype scope because they require organizational knowledge not present in schema metadata | Constitution Section 4 |

---

## 8. What This Research Does Not Cover

The following are explicitly out of scope for this prototype and are documented as
Synchrony's post-handoff responsibility in the Constitution (Section 4: Handoff Boundary):

- Data stewardship assignment (requires organizational knowledge)
- Privacy/PCI classification (requires knowledge of actual data values)
- Business lineage tracking (requires enterprise dependency mapping)
- Change management workflows
- SLA monitoring
- Approval routing

---

## 9. Sources

| # | Source | Type | URL |
|---|--------|------|-----|
| 1 | FDIC BankFind Suite — Variable Definitions | Primary benchmark (dataset analyzed) | https://banks.data.fdic.gov/docs/ |
| 2 | FDIC Community Banking Study — Technical Appendix | Secondary benchmark | https://www.fdic.gov/resources/community-banking/ |
| 3 | 12 CFR Part 360.9 — Resolution Planning | Secondary benchmark (regulation) | https://www.ecfr.gov/current/title-12/part-360 |
| 4 | BCBS 239 — Risk Data Aggregation Principles | Regulatory framework | https://www.bis.org/publ/bcbs239.pdf |
| 5 | OCC SR 11-7 — Model Risk Management | Regulatory guidance | https://www.federalreserve.gov/supervisionreg/srletters/sr1107.htm |
| 6 | FFIEC IT Examination Handbook | Regulatory guidance | https://ithandbook.ffiec.gov/ |
| 7 | HL7 FHIR R4 — ElementDefinition | Cross-sector context | https://hl7.org/fhir/R4/ |
| 8 | FEMA Data Lexicon | Cross-sector context | https://catalog.data.gov/dataset/data-lexicon-data-dictionary |
| 9 | NASA PDS4 Data Standards | Cross-sector context | https://pds.nasa.gov/datastandards/documents/im/current/ |
| 10 | HHS TAGGS Data Dictionary | Cross-sector context | https://taggs.hhs.gov/About/Data_Dictionary |
| 11 | Frictionless Data — Table Schema | Cross-sector context | https://specs.frictionlessdata.io/table-schema/ |

---

*End of research.md — F3 Phase 0*
