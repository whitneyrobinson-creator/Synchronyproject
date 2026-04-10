# F5 — SKILL.md: RCSA Control Narrative Generation — Research

**Feature Branch**: `f5-rcsa-skill`
**Created**: 2026-04-10
**Status**: Draft
**Owner**: Whitney Robinson (PM), Sheila Green, Molly Lowell

---

## 1. Problem Statement

Synchrony's risk/compliance teams currently assess compliance posture manually — reading through repository artifacts, mapping evidence to controls, writing narratives, and flagging gaps. This process is time-consuming, inconsistent, and difficult to audit because the reasoning behind compliance claims is rarely documented with traceable citations.

F5 addresses the LLM reasoning layer of this problem: given pre-processed mapped evidence from the deterministic scripts pipeline (F6), can an LLM generate accurate, citation-backed, confidence-scored control narratives that serve as audit-ready documentation — while explicitly flagging gaps where evidence is missing or insufficient?

F5 is the orchestrator. It tells the LLM when to call F6 scripts (input validation, artifact registry building, evidence mapping, citation validation) and when to reason on its own (narrative generation, confidence scoring, gap detection). The SKILL.md is the single instruction file that defines this entire workflow.

---

## 2. Prompt Engineering Patterns

### 2.1 Few-Shot Prompting (Used)

F5 embeds worked examples directly in the SKILL.md — one for a control with strong evidence (HIGH confidence), one for a control with weak evidence (LOW confidence), and one for a control with no evidence (GAP flag). These examples show the full input → output cycle so the LLM can pattern-match against concrete demonstrations.

**Why this pattern:** Few-shot prompting is the most reliable technique for structured generation tasks. The LLM mirrors the format, citation style, and quality level it sees in the examples. Without them, narrative tone, citation format, and gap flag language vary significantly across runs.

**Application in F5:**

- HIGH confidence example — shows multiple strong artifacts, specific narrative with inline citations, confident assessment
- LOW confidence example — shows a single weak artifact, hedged narrative with caveat language, explicit note that stronger evidence is needed
- GAP example — shows zero artifacts, explicit `[GAP]` flag, no compliance narrative generated

**Key insight:** The citation format in the examples directly controls the citation format in the output. If examples show vague citations like "based on repository files," the LLM produces vague citations. If examples show specific citations like `[src/auth/login.py — Role-based access check function]`, the LLM produces specific citations.

### 2.2 Rubric-Based Confidence Scoring (Used)

Rather than letting the LLM assign confidence tiers based on intuition, F5 defines a deterministic rubric:

| Confidence | Criteria |
|------------|----------|
| **HIGH** | ≥2 artifacts with matching evidence types that directly support the control objective |
| **MEDIUM** | 1 artifact with a matching evidence type, OR ≥2 artifacts with indirect/partial support |
| **LOW** | 1 artifact with indirect/partial support only |
| **GAP** | 0 artifacts mapped to the control (separate flag, not a confidence tier) |

**Why this pattern:** Rubric-based scoring makes confidence tiers meaningful and repeatable. Without a rubric, "HIGH" might mean different things on different runs. With a rubric, the tier is traceable to specific evidence counts and types, which is required for audit readiness (Constitution Principle 4).

**Relationship to GAP flags:** GAP is not a confidence tier — it is a distinct flag that triggers when zero evidence exists. A control can have LOW confidence (weak evidence exists) without being a GAP (no evidence exists). This distinction matters because a GAP means "we found nothing" while LOW means "we found something, but it's weak."

### 2.3 Constraint-Based Generation (Used)

F5 applies explicit constraints to the LLM's output:

- Narratives must be 3–5 sentences per control (FR-001)
- Only cite artifacts present in the mapped evidence input (FR-006)
- Never imply compliance without proof (FR-003, Constitution Principle 1)
- When evidence is ambiguous, exclude it and flag for human review rather than include with caveats
- Use exact artifact identifiers from the artifact registry (FR-007)
- Prefer GAP flags over uncertain compliance claims (FR-008)

**Why this pattern:** LLMs tend toward confident, comprehensive output by default. In a compliance context, this is dangerous — implying compliance without evidence is worse than flagging a gap. Explicit constraints force the LLM to stay conservative. The "exclude ambiguous evidence" rule prevents hallucinated compliance claims. The "prefer GAP over uncertainty" rule ensures the system fails safe.

### 2.4 Orchestration Pattern (Used)

The SKILL.md defines a sequential workflow where the LLM alternates between calling F6 scripts (deterministic operations) and performing its own reasoning (narrative generation). The sequence is:

1. **Call script**: Validate inputs (F6)
2. **Call script**: Build artifact registry (F6)
3. **Call script**: Map evidence to controls (F6)
4. **LLM reasons**: Generate narratives with citations, assign confidence tiers, flag gaps
5. **Call script**: Validate citations against registry (F6)
6. **LLM reasons**: Produce summary table and assemble final output

**Why this pattern:** This separates deterministic work (parsing, validation, mapping) from reasoning work (narrative generation, confidence assessment). The LLM only does what only an LLM can do. Everything else is handled by reliable, testable scripts. This aligns with Constitution Principle 3 (Simplicity First) and the project spec's division of labor between SKILL.md and Scripts.

### 2.5 Conservative Citation Strategy (Used)

When an artifact maps to multiple controls, the SKILL.md instructs the LLM to cite it in both narratives but note the overlap. When the LLM is unsure about a mapping, it excludes the artifact and flags the uncertainty for human review.

**Why this pattern:** In audit contexts, over-claiming is worse than under-claiming. A conservative citation strategy means every citation in the output is defensible. The post-processing validation (F6) catches any citations that slip through without matching the registry, but the SKILL.md's first line of defense is to be conservative at generation time.

### 2.6 Red Teaming (Planned — Verification)

After the SKILL.md is built, the team will feed intentionally bad inputs to verify the skill degrades gracefully:

- Controls with zero evidence (should produce GAP flags, not hallucinated narratives)
- Controls with only ambiguous evidence (should exclude evidence and flag uncertainty)
- Malformed input structure (should flag the issue rather than guess at the data)
- All four controls with no evidence (should produce four GAP flags)

**Why this pattern:** Red teaming validates that the constraints and rubric actually work under adversarial conditions. It's the difference between "the SKILL.md says it handles missing evidence" and "we proved it handles missing evidence."

---

## 3. Alternative Approaches Considered

### 3.1 Multi-File Approach (Rejected)

**What it is:** Separate files for the rubric, worked examples, output templates, and instructions — instead of embedding everything in one SKILL.md.

**Why considered:** The SKILL.md for F5 is more complex than F1 — it includes an orchestration workflow, confidence rubric, worked examples, output structure definitions, and edge case handling rules. Splitting could improve readability.

**Why rejected:**

- The content volume is manageable in a single file. The orchestration sequence is 6 steps. The rubric is 4 levels. The worked examples are 3 short blocks.
- Splitting means the LLM has to find and load multiple files at runtime — an extra dependency that could break.
- F4 (Assets) will formalize templates into separate files later. During F5 development, keeping everything inline reduces coordination overhead.
- Constitution Principle 3 (Simplicity First) favors the single-file approach for a 6-week prototype.

**Revisit condition:** If the skill grows post-demo (more controls, more complex rubric, additional reference data), splitting into separate files would make sense.

### 3.2 LLM-Driven Evidence Mapping (Rejected)

**What it is:** Having the LLM perform the evidence-to-control mapping itself, rather than receiving pre-mapped evidence from F6 scripts.

**Why rejected:**

- Evidence mapping is a deterministic operation — matching artifact types to control requirements. Scripts do this reliably and repeatably.
- LLM-driven mapping introduces hallucination risk — the LLM might map artifacts to controls based on surface-level keyword matching rather than actual evidence type alignment.
- Constitution Principle 1 (Accuracy Over Speed) and the project spec's division of labor both specify that deterministic logic belongs in scripts.
- The SKILL.md's job is reasoning (narratives, confidence, gaps), not data processing (parsing, mapping, validating).

### 3.3 Remediation Suggestions in Gap Flags (Rejected)

**What it is:** When the LLM flags a GAP, it also suggests what evidence would be needed to close the gap.

**Why considered:** Remediation suggestions would make the output more actionable for compliance teams.

**Why rejected:**

- Remediation suggestions require domain knowledge about Synchrony's specific compliance requirements. The LLM doesn't have this context.
- Suggesting remediation could imply the system understands the organization's compliance posture, which it doesn't — it only sees the evidence provided.
- The feature spec scopes F5 to detection and documentation, not remediation. Adding remediation expands scope beyond what's needed for the May 7th demo.
- Constitution Principle 3 (Simplicity First) — prove core functionality works before adding complexity.

**Revisit condition:** Post-demo, if Synchrony wants remediation guidance, the control library input could be extended to include "expected evidence types" that the SKILL.md references when flagging gaps.

### 3.4 Separate Gap Report (Rejected)

**What it is:** Producing a third output document dedicated to gap analysis, separate from the control narratives.

**Why rejected:**

- The feature spec defines two output documents: `rcsa_control_narratives.md` and `validation_report.md`. Adding a third document increases complexity without clear value.
- Gap flags are most useful when they appear inline alongside the narratives — an auditor reading the control narratives sees immediately which controls have evidence and which don't.
- The summary table at the top of `rcsa_control_narratives.md` already provides the at-a-glance gap overview.

---

## 4. Existing RCSA / Compliance Automation Tools

### 4.1 Landscape

Several tools exist for automated or semi-automated compliance documentation:

| Tool | Approach | Strengths | Limitations for Our Use Case |
|------|----------|-----------|------------------------------|
| **AuditBoard** | Enterprise GRC platform with RCSA workflows | Full RCSA lifecycle management, risk scoring | Enterprise SaaS — requires deployment, licensing, integration with data sources |
| **ServiceNow GRC** | Integrated risk and compliance module | Workflow automation, audit trail | Enterprise platform — heavy infrastructure, not file-based |
| **LogicGate** | No-code GRC platform | Flexible risk assessment workflows | SaaS platform — doesn't generate narratives from repository artifacts |
| **Hyperproof** | Compliance operations platform | Evidence collection, control mapping | Focused on evidence management, not narrative generation from code artifacts |
| **Drata** | Continuous compliance monitoring | Automated evidence collection | SaaS, focused on SOC 2/ISO — doesn't generate auditor-friendly narratives |

### 4.2 Why We're Building Our Own

None of the existing tools solve F5's specific problem:

1. **We need narratives generated from repository artifacts.** Existing GRC tools manage compliance workflows but don't analyze code repositories to generate evidence-backed narratives. F5 reads mapped evidence (file paths, content snippets, artifact types) and produces auditor-friendly prose — no existing tool does this.
2. **We need explicit gap detection with zero false negatives.** Existing tools track compliance status but rely on human input to flag gaps. F5 automatically detects when evidence is missing or insufficient and flags it explicitly — the system must never imply compliance without proof.
3. **We need citation traceability to specific artifacts.** Every claim in a narrative must trace back to a specific file, function, or test in the repository via inline citations. Existing tools don't provide this level of per-claim provenance from code artifacts.
4. **We need it to run on Claude with no infrastructure.** Constitution Principle 3 (Simplicity First) and the tech stack rules specify file-based, no cloud deployment, no enterprise licensing. Existing GRC tools all require infrastructure we're not building for a 6-week prototype.
5. **The skillset is designed for Synchrony's handoff.** The goal is a proof of concept that Synchrony can extend within their environment. An enterprise GRC tool would be Synchrony's choice post-handoff — our job is to prove the LLM-based approach works.

### 4.3 What We Learned From the Landscape

- **Summary tables are standard.** Every GRC tool provides at-a-glance compliance dashboards. F5 follows this pattern with the summary table at the top of the output.
- **Gap flagging is table stakes.** Compliance tools that don't explicitly flag gaps are not audit-ready. F5's explicit GAP flags with zero false negative tolerance align with industry expectations.
- **Human review is always required.** Even enterprise GRC platforms position their output as inputs to human decision-making. F5's confidence tiers and GAP flags formalize this — the output is explicitly a draft for human review, not a final compliance determination.

---

## 5. F5-Level Testing vs. End-to-End Testing

### 5.1 F5-Level Testing (Standalone)

F5 can be tested independently by running the SKILL.md against hand-crafted sample evidence in Claude Desktop or claude.ai. This tests:

- Whether the SKILL.md produces valid narratives for each control
- Whether confidence scoring follows the rubric
- Whether GAP flags trigger correctly when evidence is missing
- Whether citations reference only artifacts present in the input
- Whether the summary table accurately reflects control statuses
- Whether ambiguous evidence is excluded and flagged

**Test data:** Sample mapped evidence for all four demo controls — Access Control (strong evidence), Change Management (weak evidence), Data Quality (ambiguous evidence), Incident Handling (no evidence).

**Who runs it:** The F5 developer, during SKILL.md iteration.

### 5.2 End-to-End Testing (Full Pipeline)

End-to-end testing runs the complete pipeline: F6 validates inputs → F6 builds artifact registry → F6 maps evidence → F5 generates narratives → F6 validates citations → F6 assembles final output. This tests:

- Whether F6's evidence mapping produces correct structured input for F5
- Whether F5's output integrates cleanly with F6's assembly and validation logic
- Whether the final `rcsa_control_narratives.md` and `validation_report.md` match expected format
- Whether citation validation catches any hallucinated references

**Test data:** Full sample artifact index, test catalog, and control library for the demo dataset.

**Who runs it:** The full team, after F6 is built.

### 5.3 Ground Truth Evaluation

The team will manually score F5's output against the success criteria (SC-001 through SC-005) using the demo dataset. This establishes baseline performance and sets thresholds for demo day.

**Process:**

1. Build the SKILL.md
2. Run it on sample mapped evidence for all four controls
3. Team manually scores output across all 5 success criteria
4. Set thresholds based on actual performance
5. Document final thresholds before demo day

---

## 6. Key Design Decisions Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Orchestration model | SKILL.md orchestrates F6 scripts + does its own reasoning | Separates deterministic work from LLM reasoning; LLM only does what only an LLM can do |
| File structure | Single SKILL.md with everything embedded | Content volume manageable; reduces runtime dependencies; Simplicity First |
| Output documents | Two: `rcsa_control_narratives.md` + `validation_report.md` | Matches feature spec; narratives for auditors, validation for QA |
| Output format | Markdown with YAML front matter | Human-readable, version-controllable, consistent with project-wide output format |
| Evidence input format | File paths + content snippets (structured JSON) | Gives LLM enough context to write narratives without sending raw source files |
| Confidence scoring | Rubric-based (HIGH/MEDIUM/LOW) | Repeatable, auditable, traceable to specific evidence counts and types |
| GAP flags | Distinct flag, separate from confidence tiers | GAP = zero evidence (binary); confidence = evidence quality (graduated) |
| Gap handling | Flag only — no remediation suggestions | Remediation requires domain knowledge the LLM doesn't have; out of scope for demo |
| Weak evidence (single artifact) | Write narrative, assign LOW confidence | Provides value while being transparent about evidence weakness |
| Multi-control artifacts | Cite in both narratives, note overlap | Conservative but complete; auditor sees all relevant evidence per control |
| Ambiguous mappings | Exclude, flag uncertainty for human review | Anti-hallucination; accuracy over coverage (Constitution Principle 1) |
| Framework approach | Framework-agnostic — controls provided as input | Not tied to a specific compliance framework; adaptable to any control set |
| Demo controls | Access Control, Change Management, Data Quality, Incident Handling | Four generic controls that demonstrate the pattern without requiring Synchrony-specific knowledge |
| Citation format | Descriptive in narratives, full technical detail in validation report | Narratives readable by auditors; validation report useful for QA |
| Build order | F5 first, then F4, then F6 | High-risk LLM dependency addressed first; output structure defined before templates formalized |

---

## 7. Open Research Questions

| # | Question | Status | Impact |
|---|----------|--------|--------|
| 1 | What is the optimal evidence snippet length for narrative quality? | TBD — test during baseline | Too short = vague narratives; too long = context window pressure |
| 2 | Does the number of controls per run affect output quality? | TBD — test if scaling beyond 4 controls | May require batching strategy for production use |
| 3 | How stable is output across Claude model versions? | TBD — test if model updates occur before demo day | May require SKILL.md adjustments |
| 4 | What is Synchrony's actual control count and codebase size? | Unknown — deferred to handoff | Determines production scaling requirements |
| 5 | Can the confidence rubric thresholds be calibrated from real-world data? | Deferred — future enhancement | Would make confidence tiers more meaningful for Synchrony's specific context |
| 6 | Does the 90-second per-call ceiling hold under multi-pass reasoning? | TBD — monitor during testing | Risk flagged in Constitution Check; may require optimizing prompt length |

---

*This document captures the research and design rationale for F5. It is a living document — update it as new patterns are tested or alternatives are evaluated.*
