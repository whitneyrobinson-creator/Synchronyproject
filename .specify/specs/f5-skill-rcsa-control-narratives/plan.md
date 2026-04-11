# F5 — SKILL.md: RCSA Control Narrative Generation — Plan

**Feature Branch**: `f5-rcsa-skill`
**Created**: 2026-04-07
**Updated**: 2026-04-11
**Status**: Draft
**Phase**: 2

---

## Section 1: Summary

F5 is the SKILL.md file — the natural language instruction set that tells the LLM agent how to generate citation-backed RCSA control narratives from pre-processed mapped evidence. It is the orchestrator of the RCSA workflow: it calls F6 scripts as tools, reasons over the structured data they return, and produces two audit-ready output documents.

**What F5 does:**
- Orchestrates the 6-step RCSA workflow (validate → build registry → map evidence → generate narratives → validate citations → assemble output)
- Generates 3–5 sentence auditor-friendly narratives per control with inline citations
- Flags explicit [GAP] statements when evidence is missing or insufficient
- Assigns confidence tiers (HIGH / MEDIUM / LOW) based on evidence strength
- Produces a summary table for at-a-glance compliance posture
- Follows defined output structure for consistent formatting

**What F5 does NOT do:**
- Parse raw files (F6 handles this)
- Validate input format (F6 handles this)
- Validate citations post-generation (F6 handles this)
- Define reusable templates (F4 handles this)
- Handle graceful degradation when the LLM fails (F6 handles this)

**Key decisions from research:**
- Orchestration model: SKILL.md is the conductor — calls scripts and reasons in between
- Gap handling: Flag only — no remediation suggestions
- Confidence tiers: HIGH / MEDIUM / LOW (separate from GAP flags)
- Framework-agnostic: Controls provided as input, not hardcoded
- Output: Markdown with YAML front matter
- Edge cases: Weak evidence → LOW confidence narrative; ambiguous mappings → exclude and flag

---

## Section 2: Technical Context

**Language/Version**: N/A — SKILL.md is a natural language instruction file, not executable code

**Primary Dependencies**:
- F6 (Scripts — RCSA) — deterministic tools the SKILL.md orchestrates
- F4 (Assets — RCSA) — templates formalized after F5 establishes output structure
- LLM agent runtime (platform-specific, not owned by F5)

**Storage**: N/A — F5 produces Markdown files; persistence handled by agent platform and F6 scripts

**Testing**: Manual evaluation + automated validation
- **Priority 1**: Gap detection accuracy — zero false negatives on missing evidence
- LLM output quality assessed via rubric (narrative completeness, citation accuracy, confidence tier correctness)
- Citation validation handled by F6 scripts (`validation_report.md`)
- Edge case testing: zero-evidence controls, single weak artifact, ambiguous mappings, multi-control artifacts

**Target Platform**: LLM agent runtime (framework-agnostic)

**Project Type**: Agent skill definition (natural language instruction file)

**Performance Goals**: NEEDS CLARIFICATION — latency/throughput targets for a full RCSA assessment run

**Constraints**:
- LLM context window must accommodate all evidence snippets per run; if evidence exceeds limits, chunking/prioritization is an F6 responsibility but F5 must handle partial input gracefully
- Anti-hallucination: exclude ambiguous evidence, flag for human review
- Gap flags must have zero false negatives

**Scale/Scope**:
- Demo: 4 controls, small codebase
- Production: NEEDS CLARIFICATION — target control count and codebase size

---

## Section 3: Constitution Check

**Principle 1 — Accuracy Over Speed**: F5 enforces this directly. The SKILL.md instructs the LLM to never imply compliance without proof, prefer GAP flags over uncertain claims, and cite only artifacts present in the registry. This is the hardest constraint in the system.

**Principle 2 — Graceful Degradation**: Not F5's responsibility. If the LLM fails, F6 scripts handle fallback. However, F5 must produce parseable output so F6 can validate it even if the narratives are imperfect.

**Principle 3 — Simplicity First**: F5 is a single Markdown file. No code, no dependencies, no configuration. The orchestration is expressed as natural language steps, not a state machine.

**Principle 4 — Audit-Ready by Default**: F5 produces two documents designed for auditor review: `rcsa_control_narratives.md` (the deliverable) and `validation_report.md` (the QA check). Every claim has a citation. Every gap is flagged.

**Never-Ever Rules**: F5 instructs the LLM to work only with provided evidence. No external knowledge, no assumptions about the repository, no raw PII, no credentials.

---

## Section 4: Project Structure

**Documentation (this feature)**

```text
specs/005-skill-rcsa/
├── plan.md              # This file
├── research.md          # Phase 0 output (interview decisions)
├── data-model.md        # Phase 1 output (input/output schemas)
├── quickstart.md        # Phase 1 output (how to run/test the skill)
├── contracts.md         # Phase 1 output (F5↔F4, F5↔F6 interfaces)
└── tasks.md             # Phase 2 output (implementation tasks)
```

**Source Code (repository root)**

```text
.cursor/skills/rcsa/
├── SKILL.md                          # F5 — The agent skill file (primary deliverable)
├── scripts/                          # F6
│   ├── validate_input.py
│   ├── build_registry.py
│   ├── map_evidence.py
│   ├── orchestrate_llm.py
│   ├── validate_citations.py
│   └── write_output.py
└── assets/                           # F4
    ├── templates/
    ├── sample-data/
    └── citation_format.md
```

**Structure Decision**: Single skill directory under `.cursor/skills/rcsa/`. F5 (SKILL.md), F6 (scripts/), and F4 (assets/) are co-located because they form a single agent skill. The `examples/` directory for F5 testing lives in `assets/sample-data/`.

---

## Section 5: Non-Functional Requirements

**From the Constitution:**

| NFR | Source | How F5 Addresses It |
|-----|--------|-------------------|
| Never imply compliance without proof | Principle 1 | SKILL.md explicitly instructs: "If mapped_artifacts is empty, output a [GAP] flag. Do not generate a compliance narrative." |
| Citations must reference real artifacts | Principle 1 | SKILL.md instructs: "Only cite artifacts present in the artifact registry. Use exact file paths." F6 validates post-generation. |
| Output must be audit-ready | Principle 4 | SKILL.md defines structured output with YAML front matter, summary table, per-control sections, and validation report. |
| No external knowledge | Never-Ever Rules | SKILL.md instructs: "Use only the evidence provided in the input. Do not use external knowledge or assumptions about the repository." |
| System must degrade gracefully | Principle 2 | F5 produces parseable Markdown so F6 can extract data even if narratives are imperfect. Graceful degradation logic is F6's responsibility. |

**From the Feature Spec:**

| NFR | Spec Reference | How F5 Addresses It |
|-----|---------------|-------------------|
| ≥ 85% gap detection accuracy | SC-001 | SKILL.md prioritizes GAP flags over uncertain claims. Confidence rubric is explicit. |
| ≤ 5% hallucinated artifacts | SC-002 | SKILL.md constrains citations to registry. F6 validates. |
| All 4 controls addressed every run | SC-003 | SKILL.md processes every control in the control library, regardless of evidence. |
| ≥ 90% structural consistency | SC-004 | SKILL.md defines exact output structure with YAML front matter and section templates. |

---

## Section 6: Performance Gates

| Gate | Metric | Target | How Measured |
|------|--------|--------|-------------|
| Gap Detection | % of true gaps correctly flagged | ≥ 85% | Test with known-gap inputs, count correct flags |
| Citation Accuracy | % of citations referencing real artifacts | ≥ 95% | F6 `validate_citations` script output |
| Control Coverage | Controls addressed per run | 4/4 (100%) | Count controls in output vs. input |
| Structural Consistency | % match across repeated runs | ≥ 90% | Run same input 5x, compare output structure |
| Narrative Quality | Rubric score vs. human-written ground truth | ≥ 80% | Manual evaluation using project rubric |

---

*This plan is the authoritative reference for F5 implementation. All design decisions are documented in research.md. Data schemas are in data-model.md. Interface contracts are in contracts.md. Testing procedures are in quickstart.md.*
