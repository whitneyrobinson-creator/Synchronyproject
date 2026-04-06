Skill 2 — RCSA Control Narrative Generation — Plan
Feature Branch: skill-2-rcsa-control-narratives Created: 2026-04-06 Status: Draft Owner: Molly Lowell, Whitney Robinson (PM), Sheila Green Demo Day Deadline: May 7, 2026

Section 1: Summary
Skill 2 is an agent skill that generates audit-ready RCSA (Risk Control Self-Assessment) control narratives from repository evidence. It combines deterministic Python scripts (tools) with LLM reasoning (SKILL.md) to map code artifacts to compliance controls and produce citation-backed documentation.

What Skill 2 does: Accepts three JSON inputs — an artifact index, a test catalog, and a control library. Scripts validate inputs, build a normalized artifact registry, and structurally map evidence to four compliance controls (Access Control, Change Management, Data Quality, Incident Handling). The LLM then receives the mapped evidence and generates 3–5 sentence auditor-friendly narratives per control with inline citations. Scripts validate all citations against the registry and assemble final outputs.

What Skill 2 produces: Two Markdown files:

rcsa_control_narratives.md — Summary table + per-control narratives with inline citations (or explicit Gap flags)
validation_report.md — Citation resolution stats: total, valid, invalid, coverage %
What Skill 2 does NOT do: Create or modify the input JSON files (those come from the repository). Design output templates (owned by assets). Make compliance decisions — it generates draft documentation for human review.

Key design decisions:

Scripts handle all deterministic work (parsing, validation, mapping, citation checking) — the LLM only does reasoning and narrative generation
The system follows a strict rule: if evidence is missing or insufficient, it MUST output a Gap and MUST NOT imply compliance (Constitution Principle 1)
If the LLM fails, scripts still produce structured output with Gap placeholders (Constitution Principle 2)
Architecture follows the same pattern as the Scout codebase exploration skill: standalone Python scripts outputting JSON, agent reads SKILL.md for workflow instructions
Demo day target: Run the skill against a sample repository's artifact index, test catalog, and control library. Demonstrate that the system produces accurate, citation-backed narratives for controls with evidence and explicit Gap flags for controls without evidence.

Section 2: Technical Context
Language/Version: Python 3.11+ for scripts (tools). SKILL.md is a natural language Markdown instruction file. Output files are Markdown.

Primary Dependencies: Python standard library only (json, os, sys, argparse). Claude Sonnet as the LLM reasoning engine. No external packages required for scripts.

Storage: File-based. Inputs are local JSON files. Outputs are local Markdown files. No database. No cloud storage.

Testing: Baseline testing by running the full pipeline against sample inputs representing a small repository. Five success criteria measured:

SC-005 — Gap Detection Accuracy: ≥ 85% of true compliance gaps correctly flagged
SC-006 — Hallucination Rate: ≤ 5% of referenced artifacts are invented
SC-007 — Citation Resolution: ≥ 90% of citations resolve to real artifacts
SC-008 — Control Coverage: ≥ 3 of 4 controls addressed per run (system attempts all 4 every run; metric measures quality tolerance)
SC-009 — Consistency: ≥ 90% structural match across repeated runs with same inputs
Quality scored by manual review of narratives against source artifacts. Citation resolution and coverage checked via validate_citations.py script output.

Target Platform: Local execution via CLI. Claude Agent Skills runtime for SKILL.md. No cloud deployment for demo day.

Project Type: Agent skill (SKILL.md + Python tools + Markdown assets within agent skills architecture)

Performance Goals: Accuracy over speed (Constitution Principle 1). No latency SLA for demo day. The system must produce correct output for all four controls in a single run.

Constraints:

Only structured metadata sent to LLM — no raw PII, API keys, credentials, full source code, or raw datasets (Constitution Never-Ever Rules)
File-based I/O only
No microservices, no UI, no cloud deployment (Constitution Principle 3)
All citations must be traceable to real artifacts (Constitution Principle 4)
Scale/Scope: Demo day: small representative repository (sample JSON inputs provided in assets). The skill architecture is input-size agnostic — scripts handle any number of artifacts. Batching for very large repositories is a future enhancement, not a demo day requirement.

Section 3: Constitution Check
GATE: Must pass before proceeding. Re-check after implementation.

Status: PASS

Rule	Plan Compliance	Notes
Accuracy Over Speed	✅ Compliant	Gap detection is mandatory. System must never imply compliance without proof. No latency SLA.
Graceful Degradation	✅ Compliant	If Claude is offline or returns errors, scripts still produce structured output with Gap placeholders. No data is lost. (US-4 in spec)
Simplicity First	✅ Compliant	Four standalone Python scripts, one SKILL.md, two template files. No database, no UI, no cloud.
Audit-Ready by Default	✅ Compliant	Every narrative includes citations or explicit Gap flags. Validation report accompanies every run with resolution stats.
Never send raw PII to LLM	✅ Compliant	LLM receives only structured mapped evidence (artifact names, paths, types). No actual data values sent.
Never send API keys or credentials	✅ Compliant	No credentials in SKILL.md, scripts, or inputs.
Never send full source code or raw datasets	✅ Compliant	Only structured metadata (file paths, function names, test results) is sent.
Never commit secrets to the repo	✅ Compliant	No secrets in any skill files. If API keys are needed for Claude access, they use environment variables via .env (added to .gitignore).
Never opt into LLM provider training	✅ Compliant	No opt-in. Anthropic does not train on API inputs by default.
No violations detected. Skill 2's design is fully aligned with the constitution.

Section 4: Project Structure
Documentation
.specify/specs/skill-2-rcsa-control-narratives/
├── spec.md              ← Feature-level spec (done ✅)
├── plan.md              ← This plan
└── tasks.md             ← Week 4 deliverable
Source Code
.cursor/skills/rcsa/
├── SKILL.md                          ← Agent instructions (prompt)
├── README.md                         ← Skill documentation
├── assets/
│   ├── rcsa_narrative_template.md    ← Output template for narratives
│   └── validation_report_template.md ← Output template for validation
└── tools/
    ├── validate_inputs.py            ← Validates 3 JSON input files (FR-002, FR-003, FR-004, FR-005)
    ├── build_artifact_registry.py    ← Builds normalized artifact registry (FR-015)
    ├── map_evidence_to_controls.py   ← Maps evidence to 4 controls (FR-016)
    └── validate_citations.py         ← Validates citations against registry (FR-019, FR-020, FR-022)
Structure Decision: Single-skill layout following the Scout skill pattern. Tools are standalone Python scripts (stdlib only) that output structured JSON to stdout. The agent reads SKILL.md, runs tools in sequence, and uses templates from assets to produce final Markdown outputs.

What Skill 2 owns:

SKILL.md — LLM instruction file for narrative generation and gap detection
tools/ — Four Python scripts for the deterministic pipeline
assets/ — Output templates for narratives and validation report
README.md — Skill documentation
What Skill 2 uses but doesn't own:

constitution.md — Referenced for rules, not modified
spec.md — Referenced during development, not modified
Input JSON files — Provided by the user at runtime
What Skill 2 doesn't touch:

Skill 1 (Data Dictionary) files — completely independent
Project-level documents (Draft Proposal, README.md)
Section 5: Complexity Tracking
#	Complexity Added	Justification	Source
1	Separate structural mapping (scripts) + reasoning-based mapping (LLM)	Evidence mapping requires both deterministic rules (matching file types to controls) and LLM judgment (evaluating evidence sufficiency). Combining both in one place would violate Simplicity First.	Constitution Principle 3, FR-016
2	Citation validation as a separate post-processing step	LLMs hallucinate citations. A deterministic validation pass catches fabricated references before they reach the final output. Without it, audit-readiness is compromised.	Constitution Principle 1, Constitution Principle 4, FR-019
3	Graceful degradation with Gap placeholders	Adds complexity to every script (must handle LLM-absent case), but Constitution Principle 2 requires it. A compliance tool that crashes on LLM failure is worse than one that reports "I couldn't analyze this."	Constitution Principle 2, US-4
All three are justified by the constitution or the feature spec. None are gold-plating.

Section 6: Implementation Priority
Phase	What	Risk Level	Rationale
Phase 1	validate_inputs.py + build_artifact_registry.py	🟢 Low	De-risk the data pipeline first. Everything downstream depends on clean, validated inputs and a normalized registry.
Phase 2	map_evidence_to_controls.py + gap detection logic	🟡 Medium	Core mapping logic. Must correctly associate artifacts with controls using control library rules.
Phase 3	SKILL.md + LLM narrative generation	🔴 High	Riskiest part — LLM output quality is unpredictable. Needs iteration on prompt design to get consistent, citation-backed narratives. De-risk early by testing with sample inputs from Phase 1/2.
Phase 4	validate_citations.py + output assembly using templates	🟡 Medium	Depends on Phase 3 output. Citation validation logic is straightforward but critical for audit-readiness.
Phase 5	Graceful degradation + end-to-end testing	🟢 Low	Wire up the LLM-failure path. Run full pipeline against sample data. Verify all success criteria.
