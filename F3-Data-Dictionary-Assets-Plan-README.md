# Implementation Plan: F3 — Assets: Data Dictionary Generation

**Branch**: `f3-assets-data-dictionary` | **Date**: 2026-04-07 | **Spec**: `specs/f3-assets-data-dictionary/spec.md`

**Input**: Feature specification from `/specs/f3-assets-data-dictionary/spec.md`

---

## Summary

F3 delivers the static asset files that the data dictionary generation pipeline depends on: two Markdown templates (`data_dictionary_template.md`, `qa_report_template.md`), one test input file (`sample_schema.json`), two gold standard examples (`example_data_dictionary.md`, `example_qa_report.md`), and one placeholder (`glossary_template.json` — deferred, not required for demo day).

**Demo day deadline: May 7, 2026.**

These assets serve three roles:
1. **Structural contracts** — Templates define the exact output format that F2 scripts populate. Column order, section layout, and null handling are authoritative in the templates, not hardcoded in scripts.
2. **Test harness** — The sample schema provides realistic input (25 UCI Credit Card fields with intentional metadata gaps), and gold standard examples provide the expected output for pipeline validation.
3. **LLM anchoring** — Gold standard examples serve as few-shot worked examples in F1's SKILL.md to ensure consistent LLM output quality.

All assets are static files (Markdown and JSON). No code executes. F3 has no runtime dependencies — assets are created offline and consumed by F1 (SKILL.md) and F2 (scripts) at pipeline execution time. Creation is AI-assisted with mandatory team review and sign-off before use.

---

## Technical Context

**Language/Version**: Markdown and JSON — both are text-based file formats, not programming languages. F3 contains no executable code. Markdown (.md) is a simple formatting language for documents (headings, tables, bold text). JSON (.json) is a structured data format that scripts can read. Python 3.11+ is used by F2's scripts, which consume F3's files — but F3 itself has nothing to run.

**Primary Dependencies**: None — F3 has no incoming dependencies. These files are standalone and require no software, libraries, or tools to create beyond a text editor. However, other parts of the pipeline read these files:
- F1 (SKILL.md) — references the templates and gold standard examples so the LLM knows what format to produce
- F2 (`assemble_output.py`) — reads `data_dictionary_template.md` to know how to structure the final output document
- F2 (`generate_qa_report.py`) — reads `qa_report_template.md` to know how to structure the quality report
- F2 (testing) — uses `sample_schema.json` as test input to run the pipeline end-to-end

**Storage**: File-based. All assets live in a single `assets/` folder. No database, no cloud storage — just files on disk.

**Testing**: Manual review by the team, checked against the F1, F2, and F3 specs. There is no automated test framework for F3 because these are documents, not code. The team verifies correctness using six success criteria (a checklist):
- SC-F3-001 — Does the template include every required column and section?
- SC-F3-002 — Does the gold standard example follow the template's structure exactly?
- SC-F3-003 — Does the sample schema include a mix of easy, medium, and hard fields?
- SC-F3-004 — Are the gold standard descriptions actually correct (verified against the Kaggle dataset documentation)?
- SC-F3-005 — Do the numbers in the QA report match the data dictionary? (e.g., if 3 fields are Low confidence in the dictionary, the QA report should also say 3)
- SC-F3-006 — Can F2's scripts actually read and use these files without errors? (Blocked until F2 scripts are built)

**Target Platform**: Runs locally on your computer during demo day. Claude Agent Skills reads the files from disk. Nothing is deployed to the internet or a server.

**Project Type**: Static assets — files that are created once by the team and don't change while the pipeline runs. The pipeline reads them as-is. Think of them as reference documents the system consults.

**Performance Goals**: Not applicable for speed — these are files in a folder, not running software. The only goal is accuracy: the content must be correct and match the specs.

**Constraints** (rules F3 must follow):
- Templates must define the exact same columns and sections that F1 and F2 expect to produce — if there's a mismatch, the pipeline breaks
- `sample_schema.json` must use the exact field structure F2's scripts expect to read: `field_name`, `type`, `nullable`, `constraints`, `enums`, `schema_comments`
- Every description in the gold standard must be 25 words or fewer (FR-007)
- No actual data values from the UCI dataset — only metadata (field names, types, constraints), never the real numbers or personal information
- JSON format only for demo day — YAML and DDL (SQL-based schema format) are future extensions, not supported now

**Scale/Scope**: 6 asset files covering 25 fields from the UCI Credit Card dataset. That's the full scope for demo day. After handoff, Synchrony creates their own versions of the sample schema and gold standard using their real data — but the templates stay the same.

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Constitution Principle | F3 Compliance | How |
|---|---|---|
| **Principle 1: Accuracy Over Speed** | ✅ Pass | Gold standard examples are manually verified against Kaggle documentation before use. Templates include `confidence` and `evidence_refs` columns to ensure every generated description is traceable. No shortcuts — team reviews and signs off on every asset (Definition of Done). |
| **Principle 2: Graceful Degradation** | ✅ Pass | F3 assets are static files — they don't depend on the LLM being online. If Claude goes down, the templates and sample schema still exist and can be used manually. The gold standard examples also serve as a "what correct output looks like" reference if the team needs to write descriptions by hand during an outage. |
| **Principle 3: Simplicity First** | ✅ Pass | F3 is the simplest feature in the project: 6 files in a folder. No code, no dependencies, no services. Markdown and JSON only. |
| **Principle 4: Audit-Ready by Default** | ✅ Pass | Templates are designed with audit reviewers in mind — the data dictionary template includes `confidence`, `evidence_refs`, and `clarification_flag` columns so every entry can be traced back to its evidence. The QA report template includes coverage statistics and flagged items. |
| **Never-Ever Rules** | ✅ Pass | `sample_schema.json` contains only field metadata (names, types, constraints) — no actual data values, no PII, no API keys, no credentials. Enum values like `[1, 2]` for the SEX field are structural metadata describing allowed values, not personal data. No raw dataset content is included in any F3 asset. |
| **Security: What Gets Sent to LLM** | ✅ Pass | F3 assets are read locally by F2 scripts. The only F3 content that reaches the LLM is the gold standard examples embedded in F1's SKILL.md as few-shot anchors — these contain only field metadata and sample descriptions, not real data. |
| **Out of Scope Boundaries** | ✅ Pass | F3 does not include: workflows, approvals, SLAs, change control, repository scanning, deployment features, or a frontend UI. All correctly deferred per Constitution Section 4. |
| **Handoff Boundary** | ✅ Pass | F3 assets are designed to be replaceable. Synchrony creates their own sample schema and gold standard using their real data post-handoff. Templates remain the same. Governance fields (steward, privacy classification, lineage) are documented as post-handoff enhancements, not included in F3. |

**Gate result: ALL PASS.** No violations. No complexity tracking needed.

---

## Project Structure

### Documentation (this feature)

    specs/f3-assets-data-dictionary/
    ├── plan.md              # This file
    ├── research.md          # Phase 0 — benchmark analysis, regulatory mapping, FDIC dataset findings
    ├── data-model.md        # Phase 1 — structures for templates, sample schema, and gold standard
    ├── quickstart.md        # Phase 1 — how to use, update, and validate F3 assets
    ├── contracts/           # Phase 1 — input/output agreements between F3 → F1 and F3 → F2
    └── tasks.md             # Phase 2 — implementation checklist (created separately)

### Source Code (repository root)

    assets/                                  # Lives at repository root, NOT inside specs/
    ├── data_dictionary_template.md          # Markdown template — defines output structure for the data dictionary
    ├── qa_report_template.md                # Markdown template — defines output structure for the QA report
    ├── sample_schema.json                   # 25-field UCI Credit Card test input conforming to F2 input contract
    ├── example_data_dictionary.md           # Gold standard — completed data dictionary for sample_schema.json
    ├── example_qa_report.md                 # Gold standard — completed QA report for sample_schema.json
    └── glossary_template.json               # Placeholder for P3 glossary feature (deferred — not required for demo day)

**Structure Decision**: Single flat `assets/` directory at the repository root. No subdirectories needed — F3 has only 6 files, all at the same level. This is the simplest structure that works (Constitution Principle 3: Simplicity First). F2 scripts reference these files by path (e.g., `assets/data_dictionary_template.md`). F3 has no `tests/` directory — validation of these assets happens through F2's integration tests (SC-F3-006). If Synchrony adds more asset types post-handoff, they can introduce subdirectories at that point.

---

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

No violations were detected in the Constitution Check. All 8 principles passed cleanly. F3 is the simplest feature in the project — 6 static files in a folder with no code, no dependencies, and no runtime behavior.

**No complexity entries required.**

**Note:** F3's Decision D-005 (adding confidence distribution to the QA report template) creates a small cross-feature impact on F2 — `generate_qa_report.py` needs ~5 lines of Python to count fields by confidence level. This is tracked in the F3 spec (Section 8: Cross-Feature Dependencies), not here, because it's F2's implementation work, not an F3 constitution violation.
