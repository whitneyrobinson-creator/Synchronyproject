# F1 — SKILL.md: Data Dictionary Generation — Plan

**Feature Branch**: `f1-skill.md-data-dictionary`

**Created**: 2026-04-03

**Status**: Draft

**Owner**: Sheila Green (PM), Whitney Robinson, Molly Lowell

**Demo Day Deadline**: May 7, 2026

---

## Section 1: Summary

F1 is a SKILL.md file — a self-contained LLM instruction document that tells Claude how to generate data dictionary entries from structured field metadata.

**What F1 does:** Receives a JSON object from F2 containing the table name and an array of field metadata (field names, types, constraints, comments, enums). For each field, it generates a plain-language description (≤25 words), a confidence score (High/Medium/Low), evidence citations explaining the score, and a clarification flag for fields needing human review.

**What F1 produces:** A JSON array of objects — one per input field — each containing 5 fields: `field_name`, `description`, `confidence`, `evidence_refs`, `clarification_flag`. If the SKILL.md can't process a field, it still returns the standard 5-field object with Low confidence and a clarification flag — no special error format.

**What F1 does NOT do:** Parse schema files (F2), assemble final output documents (F2), or format templates (F3). F1 only reasons over metadata it receives and returns structured output.

**Key design decisions:**

- Three worked examples (High/Medium/Low confidence) are embedded directly in the SKILL.md

- Confidence scoring follows a defined rubric based on signal strength (not LLM intuition)

- The SKILL.md is batch-size agnostic — instructions don't change regardless of field count

**Demo day target:** Run the SKILL.md against the UCI Credit Card dataset (25 fields) and demonstrate accurate, consistent, audit-ready output.

---

## Section 2: Technical Context

**Language/Version**: Natural language (Markdown) — F1 is a SKILL.md instruction file, not executable code. Python 3.11+ applies to the broader project (F2 scripts).

**Primary Dependencies**: Claude Agent Skills architecture (SKILL.md format); Claude LLM as the reasoning engine that interprets the instructions. No direct code dependencies for F1 itself.

**Storage**: File-based — SKILL.md stored in `skills/data-dictionary/`. LLM outputs returned as structured JSON (format pending F2 confirmation). Final assembled outputs (.md) are F2/F3's responsibility.

**Testing**: Baseline testing by running SKILL.md against UCI Credit Card dataset (25 fields). Six success criteria measured:

- **SC-F1-001** — Completeness: LLM produces all 5 output fields for every input field

- **SC-F1-002** — Description Quality: Descriptions are understandable, accurate, ≤25 words

- **SC-F1-003** — Confidence Calibration: High/Medium/Low scores match actual signal strength

- **SC-F1-004** — Citation Coverage: Every description includes evidence_refs with reasoning

- **SC-F1-005** — Clarification Flag Accuracy: All Low confidence fields flagged [NEEDS CLARIFICATION]

- **SC-F1-006** — Consistency: Same input produces structurally similar output across runs

Quality and calibration scored manually by team; completeness, coverage, and flag accuracy checked via automated scripts (owned by F2).

F2 validates every SKILL.md response against 13 rules (see data-model.md Section 5). Critical failures trigger automatic retry (max 3 attempts). Non-critical issues are flagged for manual review.

Iteration guidance for improving SKILL.md output quality is documented in quickstart.md, including a 4-step debugging order and common failure modes.

**Target Platform**: Claude Agent Skills runtime — local execution, no cloud deployment for demo day.

**Project Type**: Agent skill (SKILL.md — LLM instruction document within Claude Agent Skills architecture)

**Performance Goals**: Accuracy over speed (Constitution Principle 1). No latency SLA for demo day. The SKILL.md must produce correct output for all fields received in a single pass, regardless of count.

**Constraints**:

- ≤25 words per description (FR-007)

- Only metadata sent to LLM — no raw PII, datasets, or credentials (Constitution Never-Ever Rules)

- File-based I/O only

- No microservices, no UI, no cloud deployment (Constitution Principle 3)

**Scale/Scope**: Demo day: 25 fields (UCI Credit Card dataset). The SKILL.md instructions are batch-size agnostic — if F2 sends 500 fields or 10, the instructions don't change. Batching logic for large schemas is F2's responsibility.

---

## Section 3: Constitution Check

| Constitution Principle | F1 Compliance | Notes |
|---|---|---|
| **Accuracy Over Speed** | ✅ Compliant | No latency SLA. Confidence rubric and evidence citations prioritize correctness over generation speed. |
| **Graceful Degradation** | ✅ Compliant | If Claude is offline, F2 scripts still extract raw metadata. F1's SKILL.md simply doesn't run — no data is lost. |
| **Simplicity First** | ✅ Compliant | Single Markdown file. No code dependencies, no database, no UI, no cloud. |
| **Audit-Ready by Default** | ✅ Compliant | Every description includes evidence_refs tracing back to input signals. Confidence scores follow a defined rubric. Clarification flags direct reviewers to uncertain items. |
| **Never send raw PII to LLM** | ✅ Compliant | F1 receives only metadata (field names, types, constraints). Actual data values are never sent. |
| **Never send API keys or credentials** | ✅ Compliant | No credentials in SKILL.md or its inputs. |
| **Never send full source code or raw datasets** | ✅ Compliant | Only structured field metadata is sent. |
| **Never commit secrets to the repo** | ✅ Compliant | SKILL.md contains no secrets — it's an instruction document. |
| **Never opt into LLM provider training** | ✅ Compliant | No opt-in. Anthropic does not train on API inputs by default. |

**No violations detected.** F1's design is fully aligned with the constitution.

---

## Section 4: Project Structure

### Documentation (this feature)

    specs/f1-skill.md-data-dictionary/
    ├── plan.md              # This file
    ├── research.md          # Phase 0 — prompt engineering patterns, alternative approaches considered, existing tools landscape, testing strategy
    ├── data-model.md        # Phase 1 — input metadata schema, 5-field output structure, confidence rubric definitions
    ├── quickstart.md        # Phase 1 — how to run, test, and iterate on the SKILL.md
    ├── contracts/
    │   ├── input-contract.md    # What F2 sends to the SKILL.md
    │   └── output-contract.md   # What the SKILL.md sends back to F2
    └── tasks.md             # Phase 2 — implementation checklist (created separately)

### Source Code (repository root)

    skills/                                  # Lives at repository root, NOT inside specs/
    └── data-dictionary/
        └── SKILL.md                         # Main deliverable — LLM instruction file with confidence rubric,
                                             # signal-strength definitions, and 3 embedded worked examples
                                             # (High/Medium/Low confidence)

**Structure Decision**: Single `SKILL.md` file inside `skills/data-dictionary/` at the repository root. No subdirectories needed — F1's entire deliverable is one self-contained Markdown instruction file with all rules, rubrics, and worked examples embedded directly. This is the simplest structure that works (Constitution Principle 3: Simplicity First). F2 scripts invoke this skill at runtime and pass it structured field metadata as JSON. F1 has no `tests/` directory — validation of the SKILL.md happens through baseline testing against the UCI Credit Card dataset (SC-F1-001 through SC-F1-006), with automated checks owned by F2's scripts and manual quality scoring done by the team.

---

## Section 5: Complexity Tracking

| # | Complexity Added | Justification | Source |
|---|---|---|---|
| 1 | Confidence rubric with signal strength definitions | Makes confidence scores meaningful and repeatable for auditors. Without it, Claude's scores would be arbitrary and inconsistent across runs. | Constitution Principle 1 (Accuracy Over Speed), FR-F1-003 |
| 2 | Three worked examples embedded in SKILL.md | Few-shot examples anchor LLM behavior and dramatically improve output consistency. Without them, Claude interprets rules abstractly and produces more variable results. | Feature spec Section 3, SC-F1-006 |
| 3 | Conflict detection for disagreeing signals | Prevents confident-but-wrong descriptions when metadata contradicts itself. Real-world schemas commonly have mismatched names and comments. Red teaming after skill creation verifies this works. | Feature spec Scenario 2.4, Constitution Principle 4 (Audit-Ready by Default) |

**All three are justified by either the constitution or the feature spec. None are gold-plating.**

---

*This document is the central plan for F1. For detailed schemas, see data-model.md. For contracts, see contracts/. For testing and iteration guidance, see quickstart.md. For research rationale, see research.md.*
