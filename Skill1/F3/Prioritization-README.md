# F3 Demo Day Prioritization Report

**Feature:** F3 — Assets: Data Dictionary Generation  
**Date:** April 12, 2026  
**Demo Day Deadline:** May 7, 2026  
**Owner:** Sheila Green (PM), Whitney Robinson, Molly Lowell

---

## Prioritization Criteria

**Tier 1 (Demo)** — Must be working for the live demo. Includes: happy path requirements, assets the pipeline hard-stops without, security/infrastructure requirements, and assets explicitly required by the spec's primary user story.

**Tier 2 (Future)** — Documented but not built. Can be shown as roadmap. Includes: edge case handlers, nice-to-have enhancements, complex items not essential for the core demo, and anything that can be described as "in a production version, we would also..."

---

## Tier 1: Demo (Must Build)

| # | Feature/Asset | Why It's Tier 1 |
|---|---|---|
| 1 | `data_dictionary_template.md` | The demo's primary deliverable (`data_dictionary.md`) is built from this template. Without it, `assemble_output.py` hard-stops with an error. No template = no data dictionary = no demo. |
| 2 | `qa_report_template.md` | The QA report is the second deliverable shown to the audience. Without it, `generate_qa_report.py` hard-stops. Losing the QA report cuts the demo's audit-readiness story in half. |
| 3 | `sample_schema.json` | This is the demo input — the file uploaded to Claude to start the pipeline. Must have all 25 UCI Credit Card fields with intentional metadata gaps to exercise High/Medium/Low confidence paths. Without it, there's nothing to demo. |
| 4 | `example_data_dictionary.md` (gold standard) | Non-negotiable for three reasons: (1) The team needs it to verify pipeline output is correct before demo day. (2) Three worked examples are copied from this file into SKILL.md — without them, Claude's output quality drops significantly. (3) SC-010 scoring (≥80% against ground truth) cannot be measured without it. |
| 5 | `example_qa_report.md` (gold standard) | Required to verify that `generate_qa_report.py` counts correctly. If the QA report shows wrong numbers during the demo, it undermines the entire audit-readiness pitch. Must be internally consistent with the data dictionary gold standard (High + Medium + Low + N/A = 25). |
| 6 | Template validation (SC-F3-001) | Manual check that templates have all required columns, sections, and placeholders. If the template is wrong, every pipeline run produces wrong output. Must pass before any end-to-end testing begins. |
| 7 | Gold standard validation (SC-F3-002, SC-F3-004, SC-F3-005) | Manual checks that the gold standard matches the template structure, descriptions are accurate against Kaggle docs, and QA report numbers match data dictionary counts. If the answer key is wrong, you can't tell if the pipeline is right. |
| 8 | Sample schema validation (SC-F3-003) | Manual check that the sample schema has the required variety — at least 3 fields with missing metadata, at least 1 with cryptic enums, at least 1 with constraints. Without this variety, the demo only shows the happy path and never demonstrates Medium/Low confidence handling. |
| 9 | Gold standard update process (5-step) | The team will iterate on the gold standard during testing. Without a defined update process, edits to the gold standard silently break the QA report numbers or SKILL.md worked examples. This process prevents inconsistencies that would surface embarrassingly during the demo. |
| 10 | Cross-feature contract alignment (F3 → F2) | F2 scripts hardcode paths to F3 files. If file names, column orders, or section names don't match what F2 expects, the pipeline breaks. The contracts must be verified before integration testing begins. |

### Tier 1 Asset Summary

| Asset | Tier | Non-Negotiable? |
|---|---|---|
| `data_dictionary_template.md` | **Tier 1** | Yes — pipeline hard-stops without it |
| `qa_report_template.md` | **Tier 1** | Yes — pipeline hard-stops without it |
| `sample_schema.json` | **Tier 1** | Yes — nothing to demo without test input |
| `example_data_dictionary.md` | **Tier 1** | Yes — SKILL.md examples, SC-010 scoring, pipeline verification |
| `example_qa_report.md` | **Tier 1** | Yes — QA report verification |
| `glossary_template.json` | **Tier 2** | No — P3 placeholder, nobody reads it |

**5 out of 6 F3 files are Tier 1. Only the glossary placeholder is Tier 2.**

---

## Tier 2: Future (Document Only)

| # | Feature/Asset | Why It's Tier 2 | How to Present in Demo |
|---|---|---|---|
| 1 | `glossary_template.json` | P3 feature — no script reads it, no test uses it. Exists as a placeholder for a future glossary mapping feature. | "We've also designed a glossary mapping capability. The template is ready — when Synchrony builds the glossary feature, it plugs right in." |
| 2 | YAML/DDL input format support | Constitution and spec explicitly defer non-JSON formats. Demo uses JSON only. | "For the prototype, we support JSON schemas. The architecture is format-agnostic — adding YAML or DDL parsing is a future extension that doesn't require changing the templates or SKILL.md." |
| 3 | Post-handoff governance fields (steward, privacy classification, lineage) | Require organizational knowledge that doesn't exist in schema metadata. Synchrony adds these after handoff. | "Our templates are designed to be extensible. Synchrony can add columns like data steward, privacy classification, and lineage without changing the pipeline logic — just update the template." |
| 4 | Multiple schema support (adding new datasets beyond UCI) | The quickstart documents the full process for adding new schemas. But for demo day, only the UCI dataset is needed. | "We've documented the process for onboarding new schemas. Synchrony creates their own sample schema and gold standard using their real data — the templates stay the same." |
| 5 | Automated validation of F3 assets | All 6 success criteria (SC-F3-001 through SC-F3-006) are checked manually for demo day. Automated checks would be nice but aren't necessary for 6 files. | "For the prototype, validation is manual — the team reviews each asset against the spec. In production, these checks could be automated as part of a CI pipeline." |
| 6 | SC-F3-006: F2 integration test | Blocked until F2 scripts are built. Can't be completed until F2 is ready. The other 5 success criteria can be validated independently. | Not presented separately — this is part of the end-to-end demo itself. If the demo works, SC-F3-006 passes. |
| 7 | Template versioning | Templates don't have version numbers. There's only one version — the current one. Versioning adds complexity for no demo day benefit. | "In production, you'd want template versioning so you can track changes over time. For the prototype, we keep it simple — one version, team sign-off on changes." |
| 8 | Large enum truncation | Edge case for schemas with 100+ enum values in a single field. UCI dataset has max 12. | "We've documented this as a known limitation. In production, you'd add truncation with a 'and N more values' summary." |
| 9 | 500+ field schema handling | No pagination or chunking. Demo is 25 fields. | "The pipeline handles any field count, but for very large schemas, you'd want pagination in the output. That's a post-handoff enhancement." |

---

## Recommended Demo Script Outline

Based on Tier 1 features. F3 is invisible to the audience — its assets are consumed by F1 and F2 behind the scenes. The demo shows the results of F3's work, not F3 itself.

### Step 1: Show the Input (`sample_schema.json`)

"Here's a real database schema — 25 fields from a credit card dataset. Some fields have rich metadata, some have almost nothing. This is typical of what Synchrony's teams encounter."

Open `sample_schema.json` briefly. Point out one well-documented field (`LIMIT_BAL` — has type, constraints, comments) and one sparse field (`PAY_0` — just a name). This sets up the confidence scoring story.

### Step 2: Run the Pipeline (F1 + F2 consume F3's templates)

"We upload the schema and ask Claude to generate a data dictionary."

Upload `sample_schema.json` to Claude. The pipeline runs. F2's scripts read F3's templates behind the scenes. The audience doesn't see this — they just see Claude working.

**Contingency:** If Claude is slow or down, use the cached `llm_output.json` from baseline testing (F1 quickstart §8). The audience sees the same final output either way.

### Step 3: Show the Data Dictionary (built from `data_dictionary_template.md`)

"Here's what came out — a complete data dictionary with descriptions, confidence scores, and evidence for every field."

Open `data_dictionary.md`. Walk through:

- A **High confidence** field (e.g., `AGE`) — "Claude was confident here. The field name is clear, the type confirms it, and the evidence explains why."
- A **Low confidence** field (e.g., `PAY_0`) — "Claude flagged this one. The name is ambiguous, there's no comment, and the numeric enums don't help. It's marked [NEEDS CLARIFICATION] so a human knows to review it."
- The **Evidence & Citations** section — "Every description traces back to the specific metadata that produced it. An auditor can verify any entry."

### Step 4: Show the QA Report (built from `qa_report_template.md`)

"Before anyone reviews the data dictionary, they check the QA report."

Open `qa_report.md`. Walk through:

- **Coverage Statistics** — "100% of fields got descriptions. If the LLM had failed on some, you'd see the gap here."
- **Confidence Distribution** — "At a glance: X fields are High confidence, Y are Medium, Z are Low. The auditor knows exactly how much human review is needed."
- **Fields Requiring Clarification** — "These are the fields that need attention. The system is honest about what it doesn't know."

### Step 5: Show the Audit Trail

"Everything is traceable. The template defines the format. The gold standard defines what correct looks like. The QA report shows what actually happened."

Briefly show:

- The template — "This is the single source of truth for the output format."
- The gold standard — "This is our answer key — hand-verified against the dataset documentation."
- The QA report numbers matching the gold standard — "The numbers add up. The system is internally consistent."

This is the Constitution Principle 4 (Audit-Ready by Default) moment.

---

## Risk Flags

### ⚠️ Risk 1: Gold Standard Creation — Effort Risk

**`example_data_dictionary.md`** — This is the highest-effort Tier 1 item in F3.

**What it requires:** AI-assisted drafting of descriptions for all 25 fields, then team review of every field against Kaggle documentation, corrections, internal consistency checks, and sign-off.

**Why it's risky:** This is 2–4 hours of focused team work. If it's not scheduled, it gets pushed to the last minute. A rushed gold standard means wrong answer keys, which means the team can't tell if the pipeline is producing correct output during testing. The SKILL.md worked examples are derived from this file — if the gold standard is wrong, Claude's output quality degrades.

**Mitigation:** Schedule a dedicated gold standard creation session. Block the time. Don't let it be "we'll get to it." This is the single most important quality gate for the entire demo.

### ⚠️ Risk 2: F2 Template Dependency — Timing Risk

**Template delivery timing** — `assemble_output.py` and `generate_qa_report.py` cannot be fully tested until F3 delivers the templates.

**Why it's risky:** The demo's two deliverables (`data_dictionary.md` and `qa_report.md`) are produced by these scripts. If they've never been tested with the real templates, bugs surface during the demo.

**Mitigation:** F3 templates should be the first F3 assets created — before the gold standards, before the sample schema is finalized. F2 can start integration testing immediately. The gold standard can come later since it's used for validation, not pipeline execution.

**Recommended F3 build order:**

1. `data_dictionary_template.md` + `qa_report_template.md` (unblocks F2 integration testing)
2. `sample_schema.json` (unblocks end-to-end testing)
3. `example_data_dictionary.md` + `example_qa_report.md` (unblocks scoring and SKILL.md examples)
4. `glossary_template.json` (whenever — Tier 2)

### ⚠️ Risk 3: Cross-Feature Section Name Alignment — Verify During Integration

**Section name hardcoding** — F3's QA report template defines authoritative section names. F2's scripts must use these exact names.

**Background:** This was previously a mismatch (e.g., "Coverage Summary" vs. "Coverage Statistics") but has been resolved in all planning documents. F2 now uses F3's canonical names.

**Why it's still flagged:** The documents are aligned, but the code hasn't been written yet. When F2 implements `generate_qa_report.py`, the developer needs to use the section names from the template, not from memory.

**Mitigation:** During F2 code review, verify that `generate_qa_report.py` reads section names from the template (or at minimum, uses the exact strings defined in F3's contracts). Don't rely on the developer remembering the correct names.

---

## Summary

| Category | Count |
|---|---|
| **Tier 1 (Must Build)** | 10 items — 5 asset files + 5 validation/process items |
| **Tier 2 (Document Only)** | 9 items — all correctly deferred per spec and constitution |
| **Risk Flags** | 3 — gold standard effort, F2 template timing, section name verification |

**F3's critical path:** Templates first (unblocks F2) → Sample schema (unblocks testing) → Gold standards (unblocks scoring). The glossary placeholder can be created anytime — it's a 15-line JSON file nobody reads.
