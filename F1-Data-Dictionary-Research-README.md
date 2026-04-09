# F1 — SKILL.md: Data Dictionary Generation — Research

**Feature Branch**: `f1-skill.md-data-dictionary`
**Created**: 2026-04-09
**Status**: Draft
**Owner**: Whitney Robinson (PM), Sheila Green, Molly Lowell

---

## 1. Problem Statement

Synchrony's documentation teams currently write data dictionaries manually — reading schema files, interpreting field names, writing descriptions, and verifying accuracy. This process takes 60–120 minutes per repository and is error-prone, inconsistent, and difficult to audit.

F1 addresses the LLM reasoning layer of this problem: given structured field metadata extracted by F2, can an LLM generate accurate, cited, confidence-scored descriptions that serve as a usable first draft for human review?

---

## 2. Prompt Engineering Patterns

### 2.1 Few-Shot Prompting (Used)

F1 embeds 3 worked examples directly in the SKILL.md — one for each confidence level (High, Medium, Low). These examples show the full input → output cycle so Claude can pattern-match against concrete demonstrations rather than interpreting abstract rules.

**Why this pattern:** Few-shot prompting is the most reliable technique for structured extraction tasks. Claude mirrors the format, specificity, and quality level it sees in the examples. Without them, output format and quality vary significantly across runs.

**Application in F1:**
- High confidence example (e.g., `AGE`) — shows rich evidence, specific description, confident scoring
- Medium confidence example (e.g., `PAY_0`) — shows partial evidence, hedged description, honest uncertainty
- Low confidence example (e.g., `col_x7`) — shows minimal evidence, flagged for review, explicit "insufficient evidence" language

**Key insight:** The specificity of the examples directly controls the specificity of the output. If examples show vague `evidence_refs` like "based on field name," Claude produces vague evidence. If examples show concrete citations like "schema_comment: 'Repayment status in September 2005'," Claude produces concrete citations.

### 2.2 Rubric-Based Scoring (Used)

Rather than letting Claude assign confidence scores based on intuition, F1 defines a deterministic rubric:

| Confidence | Criteria |
|---|---|
| **High** | ≥2 strong signals that agree (no conflicts) |
| **Medium** | 1 strong signal, OR 2+ signals with a conflict |
| **Low** | 0 strong signals → auto-flag [NEEDS CLARIFICATION] |

Each of the 5 signal types (field name, schema comments, data type, constraints, enums) is classified as Strong or Weak based on specific definitions in the SKILL.md.

**Why this pattern:** Rubric-based scoring makes confidence scores meaningful and repeatable. Without a rubric, Claude's scores are arbitrary — "High" might mean different things on different runs. With a rubric, the score is traceable to specific evidence, which is required for audit readiness (Constitution Principle 4).

### 2.3 Constraint-Based Generation (Used)

F1 applies explicit constraints to the LLM's output:
- Descriptions ≤25 words (FR-007)
- Only describe what can be inferred from provided metadata (FR-F1-004)
- Never invent business logic, assume domain-specific meaning without evidence, or state unsupported facts
- When uncertain, describe what the field *appears* to be and flag [NEEDS CLARIFICATION]

**Why this pattern:** LLMs tend toward confident, verbose output by default. Explicit constraints force Claude to stay within bounds. The 25-word limit prevents rambling. The "only cite what's in the input" rule prevents hallucination. The uncertainty language prevents false confidence.

### 2.4 Cross-Field Context (Used)

F2 sends the entire array of field metadata to the LLM in a single pass, so Claude sees all fields in a table simultaneously. This enables consistency across related fields (e.g., `PAY_0` through `PAY_6` should follow the same description pattern).

**Why this pattern:** Processing fields in isolation produces inconsistent descriptions for field families. Sending all fields together lets Claude recognize patterns (numbered series, shared prefixes) and apply consistent language.

**Limitation:** Claude processes each field's output independently — it doesn't revise earlier outputs based on later fields. Field family consistency still requires human review during triage.

### 2.5 Red Teaming (Planned — Verification)

After the SKILL.md is built, the team will feed intentionally bad inputs to verify the skill degrades gracefully:
- Fields with zero metadata (should produce Low confidence, not hallucinated descriptions)
- Fields with conflicting signals (should trigger conflict penalty, not ignore the conflict)
- Malformed input (should produce error descriptions, not crash)

**Why this pattern:** Red teaming validates that the rubric and constraints actually work under adversarial conditions. It's the difference between "the SKILL.md says it handles ambiguity" and "we proved it handles ambiguity."

---

## 3. Alternative Approaches Considered

### 3.1 Multi-File Approach (Rejected)

**What it is:** Separate files for the rubric, worked examples, and instructions — instead of embedding everything in one SKILL.md.

**Why considered:** Came up during Project Structure drafting. The initial proposal included a `worked-examples/` subfolder and a separate `reference/output-template.json` file alongside the SKILL.md.

**Why rejected:**
- The SKILL.md has 3 worked examples (one field each) and a confidence rubric with 3 levels. That's not enough content to justify splitting into separate files.
- Splitting means Claude has to find and load multiple files at runtime — an extra dependency that could break.
- The feature spec says the SKILL.md "must include 3 worked examples" — the word "include" reads as "put them in the file."

**External perspective:** MindStudio's best practice guide recommends that SKILL.md should only contain process steps, with context and examples in reference files. Anthropic's docs recommend progressive disclosure — keep SKILL.md lean and load supporting files only when needed. However, these recommendations target **complex skills with extensive reference data**. F1's content volume (3 short examples, a simple rubric) doesn't warrant the overhead of multi-file management.

**Revisit condition:** If the skill grows post-demo-day (more examples, more rubric levels, additional reference data), splitting into separate files would make sense.

### 3.2 Multi-Pass Processing (Rejected)

**What it is:** Having Claude process fields in multiple passes — first generate all descriptions, then score confidence separately, then add citations.

**Why rejected:**
- The 5 output fields are tightly coupled. Confidence depends on the evidence found while writing the description. Separating them into passes means Claude re-reads the same metadata twice.
- Two passes = two sets of instructions, two invocations, more coordination. One pass = one prompt, one response.
- Constitution Principle 3 (Simplicity First) favors the single-pass approach.

**Note:** This alternative was not formally discussed by the team. The single-pass approach was adopted early because the output contract naturally groups all 5 fields per input field.

### 3.3 Template Fill-in-the-Blank (Not Considered)

**What it is:** Giving Claude a template with blanks to fill rather than free-form instructions with examples.

**Why not considered:** The output is already highly structured (5-field JSON object). The worked examples effectively serve as templates by showing the exact output shape. A separate fill-in-the-blank template would be redundant.

---

## 4. Existing Data Dictionary Automation Tools

### 4.1 Landscape

Several tools exist for automated or semi-automated data dictionary generation:

| Tool | Approach | Strengths | Limitations for Our Use Case |
|---|---|---|---|
| **dbt docs** | Generates documentation from dbt model YAML files | Tight integration with dbt workflows, version-controlled | Requires dbt as the transformation layer; doesn't generate descriptions from raw schemas |
| **Alation** | Enterprise data catalog with AI-assisted descriptions | Rich metadata management, collaboration features | Enterprise SaaS — requires deployment, licensing, and integration with data sources |
| **Atlan** | Modern data catalog with automated profiling | Active metadata, lineage tracking | Enterprise SaaS — same deployment/licensing constraints as Alation |
| **Great Expectations** | Data quality and profiling framework | Statistical profiling, validation rules | Focused on data quality, not description generation; requires access to actual data |
| **AWS Glue Data Catalog** | Cloud-native schema registry | Automatic schema detection from S3/databases | AWS-specific; catalogs structure but doesn't generate human-readable descriptions |

### 4.2 Why We're Building Our Own

None of the existing tools solve F1's specific problem:

1. **We need descriptions from metadata alone.** Most tools either require access to the actual data (for profiling) or require humans to write descriptions manually. F1 generates descriptions from schema metadata — field names, types, constraints, comments — without touching the underlying data. This aligns with the Constitution's Never-Ever Rules (no raw data sent to the LLM).

2. **We need confidence scoring and gap flagging.** No existing tool produces confidence-scored descriptions with explicit [NEEDS CLARIFICATION] flags. This is the core value proposition for audit readiness (Constitution Principle 4).

3. **We need citation traceability.** Every description must trace back to the specific input signals that produced it via `evidence_refs`. Existing tools don't provide this level of per-field provenance.

4. **We need it to run on Claude with no infrastructure.** Constitution Principle 3 (Simplicity First) and the tech stack rules specify file-based, no cloud deployment, no enterprise licensing. The existing tools all require infrastructure we're not building for a 6-week prototype.

5. **The skillset is designed for Synchrony's handoff.** The goal is a proof of concept that Synchrony can extend within their environment. An enterprise catalog tool would be Synchrony's choice post-handoff — our job is to prove the LLM-based approach works.

### 4.3 What We Learned From the Landscape

- **Profiling stats improve descriptions.** Tools like Great Expectations show that statistical profiling (min/max values, null percentages, value distributions) dramatically improves documentation quality. This is a potential future enhancement for F1 — if F2 can extract profiling stats, the SKILL.md could use them as additional signals.
- **Templates matter.** Every tool uses structured templates for output. F1 follows this pattern with the 5-field output contract and F3's Markdown templates.
- **Human review is always required.** Even enterprise tools with AI features position their output as drafts for human review. F1's confidence scoring and clarification flags formalize this — the output is explicitly a draft, not a final document.

---

## 5. F1-Level Testing vs. End-to-End Testing

### 5.1 F1-Level Testing (Standalone)

F1 can be tested independently by running the SKILL.md against hand-crafted input JSON in Claude Desktop or claude.ai. This tests:
- Whether the SKILL.md produces valid 5-field output
- Whether confidence scoring follows the rubric
- Whether evidence citations are specific and accurate
- Whether clarification flags trigger correctly

**Test data:** 3-field sample from UCI Credit Card dataset (AGE, PAY_0, X1) — one per confidence level.

**Who runs it:** The F1 developer, during SKILL.md iteration.

### 5.2 End-to-End Testing (Full Pipeline)

End-to-end testing runs the complete pipeline: F2 parses a schema → F2 sends metadata to F1 → F1 returns output → F2 assembles the data dictionary → F3 formats the template. This tests:
- Whether F2's parsing produces correct metadata
- Whether F1's output integrates cleanly with F2's assembly logic
- Whether the final `data_dictionary.md` matches the expected format

**Test data:** Full UCI Credit Card dataset (25 fields).

**Who runs it:** The full team, after F2 and F3 are built.

### 5.3 Ground Truth Evaluation

The team will manually score F1's output against the 6 success criteria (SC-F1-001 through SC-F1-006) using the UCI Credit Card dataset. This establishes baseline performance and sets thresholds for demo day.

**Process:**
1. Build the SKILL.md
2. Run it on the UCI Credit Card dataset (25 fields)
3. Team manually scores output across all 6 categories
4. Set thresholds based on actual performance
5. Document final thresholds before demo day

---

## 6. Key Design Decisions Summary

| Decision | Choice | Rationale |
|---|---|---|
| File structure | Single SKILL.md with everything embedded | Content volume too small to justify splitting; reduces runtime dependencies |
| Processing approach | Single pass, all fields at once | Output fields are tightly coupled; Simplicity First |
| Confidence scoring | Rubric-based, not LLM intuition | Repeatable, auditable, traceable to specific signals |
| Worked examples | 3 examples (High/Medium/Low) embedded in SKILL.md | Few-shot prompting anchors output quality and consistency |
| Verification approach | Red teaming with adversarial inputs | Proves graceful degradation under bad conditions |
| Build vs. buy | Build custom on Claude | No existing tool provides confidence-scored, cited descriptions from metadata alone without infrastructure |

---

## 7. Open Research Questions

| # | Question | Status | Impact |
|---|---|---|---|
| 1 | Can profiling stats (min/max, null %, distributions) improve confidence scoring? | Deferred — future enhancement | Would add new signal types to the rubric |
| 2 | Does batch size affect output quality? (25 fields vs. 10 fields per pass) | TBD — test during baseline | May require adjusting F2's batching thresholds |
| 3 | How stable is output across Claude model versions? | TBD — test if model updates occur before demo day | May require SKILL.md adjustments |

---

*This document captures the research and design rationale for F1. It is a living document — update it as new patterns are tested or alternatives are evaluated.*
