# Skill 2 — RCSA Control Narrative Generation — Plan

**Feature Branch**: `skill-2-rcsa-control-narratives`
**Created**: 2026-04-06
**Status**: Draft
**Owner**: Molly Lowell, Whitney Robinson, Sheila Green
**Demo Day Deadline**: May 7, 2026

## Section 1: Summary

Skill 2 is an agent skill that generates audit-ready RCSA (Risk Control Self-Assessment) control narratives from repository evidence. It combines deterministic Python scripts (tools) with LLM reasoning (SKILL.md) to map code artifacts to compliance controls and produce citation-backed documentation.

**What Skill 2 does:** Accepts three JSON inputs — an artifact index, a test catalog, and a control library. Scripts validate inputs, build a normalized artifact registry, and structurally map evidence to four compliance controls (Access Control, Change Management, Data Quality, Incident Handling). The LLM then receives the mapped evidence and generates 3–5 sentence auditor-friendly narratives per control with inline citations. Scripts validate all citations against the registry and assemble final outputs.

**What Skill 2 produces:** Two Markdown files:
- `rcsa_control_narratives.md` — Summary table + per-control narratives with inline citations (or explicit Gap flags)
- `validation_report.md` — Citation resolution stats: total, valid, invalid, coverage %

**What Skill 2 does NOT do:** Create or modify the input JSON files (those come from the repository). Design output templates (owned by assets). Make compliance decisions — it generates draft documentation for human review.

**Key design decisions:**
- Scripts handle all deterministic work (parsing, validation, mapping, citation checking) — the LLM only does reasoning and narrative generation
- The system follows a strict rule: if evidence is missing or insufficient, it MUST output a Gap and MUST NOT imply compliance (Constitution Principle 1)
- If the LLM fails, scripts still produce structured output with Gap placeholders (Constitution Principle 2)
- Architecture follows the same pattern as the Scout codebase exploration skill: standalone Python scripts outputting JSON, agent reads SKILL.md for workflow instructions

**Demo day target:** Run the skill against a sample repository's artifact index, test catalog, and control library. Demonstrate that the system produces accurate, citation-backed narratives for controls with evidence and explicit Gap flags for controls without evidence.

## Section 2: Technical Context

**Language/Version**: Python 3.11+ for scripts (tools). SKILL.md is a natural language Markdown instruction file. Output files are Markdown.

**Primary Dependencies**: Python standard library only (`json`, `os`, `sys`, `argparse`). Claude Sonnet as the LLM reasoning engine. No external packages required for scripts.

**Storage**: File-based. Inputs are local JSON files. Outputs are local Markdown files. No database. No cloud storage.

**Testing**: Baseline testing by running the full pipeline against sample inputs representing a small repository. Six success criteria measured:

- **SC-005** — Gap Detection Accuracy: ≥ 85% of true compliance gaps correctly flagged
- **SC-006** — Hallucination Rate: ≤ 5% of referenced artifacts are invented
- **SC-007** — Citation Resolution: ≥ 90% of citations resolve to real artifacts
- **SC-008** — Control Coverage: ≥ 3 of 4 controls addressed per run (system attempts all 4 every run; metric measures quality tolerance)
- **SC-009** — Consistency: ≥ 90% structural match across repeated runs with same inputs

Quality scored by manual review of narratives against source artifacts. Citation resolution and coverage checked via `validate_citations.py` script output.

**Target Platform**: Local execution via CLI. Claude Agent Skills runtime for SKILL.md. No cloud deployment for demo day.

**Project Type**: Agent skill (SKILL.md + Python tools + Markdown assets within agent skills architecture)

**Performance Goals**: Accuracy over speed (Constitution Principle 1). No latency SLA for demo day. The system must produce correct output for all four controls in a single run.

**Constraints**:
- Only structured metadata sent to LLM — no raw PII, API keys, credentials, full source code, or raw datasets (Constitution Never-Ever Rules)
- File-based I/O only
- No microservices, no UI, no cloud deployment (Constitution Principle 3)
- All citations must be traceable to real artifacts (Constitution Principle 4)

**Scale/Scope**: Demo day: small representative repository (sample JSON inputs provided in assets). The skill architecture is input-size agnostic — scripts handle any number of artifacts. Batching for very large repositories is a future enhancement, not a demo day requirement.

## Section 3: Constitution Check

*GATE: Must pass before proceeding. Re-check after implementation.*

**Status: PASS**

| Rule | Plan Compliance | Notes |
|------|-----------------|-------|
| **Accuracy Over Speed** | ✅ Compliant | Gap detection is mandatory. System must never imply compliance without proof. No latency SLA. |
| **Graceful Degradation** | ✅ Compliant | If Claude is offline or returns errors, scripts still produce structured output with Gap placeholders. No data is lost. (US-4 in spec) |
| **Simplicity First** | ✅ Compliant | Four standalone Python scripts, one SKILL.md, two template files. No database, no UI, no cloud. |
| **Audit-Ready by Default** | ✅ Compliant | Every narrative includes citations or explicit Gap flags. Validation report accompanies every run with resolution stats. |
| **Never send raw PII to LLM** | ✅ Compliant | LLM receives only structured mapped evidence (artifact names, paths, types). No actual data values sent. |
| **Never send API keys or credentials** | ✅ Compliant | No credentials in SKILL.md, scripts, or inputs. |
| **Never send full source code or raw datasets** | ✅ Compliant | Only structured metadata (file paths, function names, test results) is sent. |
| **Never commit secrets to the repo** | ✅ Compliant | No secrets in any skill files. If API keys are needed for Claude access, they use environment variables via `.env` (added to `.gitignore`). |
| **Never opt into LLM provider training** | ✅ Compliant | No opt-in. Anthropic does not train on API inputs by default. |

**No violations detected.** Skill 2's design is fully aligned with the constitution.

## Section 4: Project Structure

### Documentation
