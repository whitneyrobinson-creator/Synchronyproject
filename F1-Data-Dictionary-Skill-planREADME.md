# F1 — SKILL.md: Data Dictionary Generation — Plan

**Feature Branch**: `f1-skill.md-data-dictionary`
**Created**: 2026-04-03
**Status**: Draft
**Owner**: Whitney Robinson (PM), Sheila Green, Molly Lowell
**Demo Day Deadline**: May 7, 2026

---

## Section 1: Summary

F1 is a SKILL.md file — a self-contained LLM instruction document that tells Claude how to generate data dictionary entries from structured field metadata.

**What F1 does:** Receives a JSON array of field metadata from F2 (field names, types, constraints, comments, enums). For each field, it generates a plain-language description (≤25 words), a confidence score (High/Medium/Low), evidence citations explaining the score, and a clarification flag for fields needing human review.

**What F1 produces:** A JSON array of objects — one per input field — each containing 5 fields: `field_name`, `description`, `confidence`, `evidence_refs`, `clarification_flag`.

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

### Documentation

```text
specs/f1-skill.md-data-dictionary/
├── spec.md              ← Feature-level spec (already written)
└── plan.md              ← This plan
```

### Source Code

```text
skills/
└── data-dictionary/
    └── SKILL.md          ← ⭐ Main deliverable — LLM instruction file
                             (3 worked examples embedded directly)
```

**What F1 owns:**
- `SKILL.md` — The only file F1 creates. Self-contained instruction document with all rules, rubrics, and worked examples inside it.

**What F1 uses but doesn't own:**
- Field metadata (JSON) — Passed to the SKILL.md at runtime by F2. F1 doesn't control where this lives or how it's created.
- `spec.md` — Referenced during development, not modified.

**What F1 doesn't touch:**
- Scripts, templates, and output files — all owned by F2/F3.

**Note:** This folder structure is proposed. Teammates building F2 and F3 may adjust the broader project layout as they create their own plans.

---

## Section 5: Complexity Tracking

| # | Complexity Added | Justification | Source |
|---|---|---|---|
| 1 | Confidence rubric with signal strength definitions | Makes confidence scores meaningful and repeatable for auditors. Without it, Claude's scores would be arbitrary and inconsistent across runs. | Constitution Principle 1 (Accuracy Over Speed), FR-F1-003 |
| 2 | Three worked examples embedded in SKILL.md | Few-shot examples anchor LLM behavior and dramatically improve output consistency. Without them, Claude interprets rules abstractly and produces more variable results. | Feature spec Section 3, SC-F1-006 |
| 3 | Conflict detection for disagreeing signals | Prevents confident-but-wrong descriptions when metadata contradicts itself. Real-world schemas commonly have mismatched names and comments. Red teaming after skill creation verifies this works. | Feature spec Scenario 2.4, Constitution Principle 4 (Audit-Ready by Default) |

**All three are justified by either the constitution or the feature spec. None are gold-plating.**
