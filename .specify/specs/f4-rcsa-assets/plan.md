# Implementation Plan: F4 — RCSA Assets (Templates, Sample Data, and Control Library)

**Branch**: `f4-rcsa-assets` | **Date**: 2026-04-10 | **Spec**: `/specs/f4-rcsa-assets/spec.md`

**Input**: Feature specification from `/specs/f4-rcsa-assets/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/plan-template.md` for the execution workflow.

---

## Summary

F4 delivers the static assets package for the RCSA Control Narrative Generation skill, targeting the **May 7th demo**. It produces six files: two Markdown output templates (`rcsa_control_narratives_template.md`, `validation_report_template.md`), three sample JSON input files (`sample_artifact_index.json`, `sample_test_catalog.json`, `sample_control_library.json`), and a citation format specification (`citation_format.md`). These assets formalize the output structures that F5 (SKILL.md) defined inline and the input contracts that F6 (Scripts) validates against — establishing the single source of truth for output formatting and control definitions (not pipeline logic, which remains in F6). F4 introduces no new behavior; it codifies what F5 proved into versioned, reusable artifacts. All files reside in `.cursor/skills/rcsa/assets/`, use UTF-8 encoding, and must pass F6's input validation with zero errors. F4 is independently testable — templates and sample data can be verified without running the full pipeline — but final validation against F6 requires that F6's input validation schema is stable.

---

## Technical Context

**Language/Version**: Python 3.11 (validation scripts only — no runtime code in F4)

**Primary Dependencies**: None. F4 is a static asset package with zero runtime dependencies.

**Storage**: File-based. Markdown and JSON files in `.cursor/skills/rcsa/assets/`.

**Testing**: Two-tier validation:
- **Tier 1 (during F4 dev)**: JSON parse checks, schema field verification, UTF-8 encoding checks, directory inventory validation. Runnable standalone.
- **Tier 2 (after F6)**: F6's input validator and output assembler consume F4 assets end-to-end. Requires F6 to be built.

**Target Platform**: Local development environment (Cursor IDE)

**Project Type**: Static asset package (data files + templates)

**Performance Goals**: N/A — no runtime execution

**Constraints**: All files UTF-8 encoded. No PII, secrets, or real Synchrony identifiers. File paths must be relative. Demo-scale data only (10–20 artifacts, 5–10 tests, 4 controls).

**Scale/Scope**: 6 files, 4 controls, 11 evidence types, demo-scale sample data

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Evidence |
|---|---|---|
| **Principle 1: Accuracy Over Speed** | ✅ Pass | F4 formalizes citation format specification and control library with explicit evidence types — supporting traceability and auditability. Templates include gap flag format to ensure the system never implies compliance without proof. |
| **Principle 2: Graceful Degradation** | ✅ Pass | F4 assets are static files independent of LLM availability. In degraded mode, F6 scripts can still read the control library, validate inputs, and populate templates with raw extracted data — the templates function with or without LLM-generated narratives. |
| **Principle 3: Simplicity First** | ✅ Pass | Static asset package — Markdown and JSON files only. Zero runtime dependencies, no services, no database, no build step. |
| **Principle 4: Audit-Ready by Default** | ✅ Pass | Templates enforce audit-ready output structure (summary tables, citation format, gap flags, validation statistics). Control library defines explicit evidence types for deterministic mapping. |
| **Never-Ever Rules** | ✅ Pass | Sample data will contain no PII, API keys, credentials, full source code, or raw datasets. All file paths are relative. No real Synchrony identifiers. Sample data will be explicitly reviewed for accidental secret inclusion before finalization. |
| **Security: Data Sent to LLM** | ✅ Pass | F4 assets are consumed locally by F6 scripts. The control library content referenced by F5 during LLM invocation describes generic compliance categories (not Synchrony-specific internal controls) — non-sensitive by design. |
| **Out of Scope Boundaries** | ✅ Pass | No microservices, cloud deployment, frontend UI, standalone API, real-time processing, or agent harness libraries. No repository scanning, multi-use-case documentation, or deployment features. |
| **Tech Stack Compliance** | ✅ Pass | Python (validation only), Markdown, JSON. No additional languages or frameworks. |
| **Handoff Boundary** | ✅ Pass | F4 assets are part of the deliverable handed to Synchrony. Sample data is clearly labeled as demo-scale test data — not production artifacts. Synchrony is responsible for replacing sample data with their own inputs post-handoff. |
| **Governance: Definition of Done** | ✅ Applicable | F4 is done when: (1) all team members have reviewed, (2) assets validated (tier 1 during F4 dev, tier 2 after F6), (3) acceptance scenarios pass, (4) all team members sign off. |

**Constitution Check Result**: **PASS — No violations. No complexity justification required.**

---

## Project Structure

### Documentation (this feature)

```text
specs/f4-rcsa-assets/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
.cursor/skills/rcsa/assets/
├── templates/
│   ├── rcsa_control_narratives_template.md
│   └── validation_report_template.md
├── sample-data/
│   ├── sample_artifact_index.json
│   ├── sample_test_catalog.json
│   └── sample_control_library.json
└── citation_format.md
```

**Structure Decision**: F4 uses a flat asset directory under `.cursor/skills/rcsa/assets/` with two subdirectories (`templates/` and `sample-data/`) plus one root-level spec file. No `src/` or `tests/` directories — F4 contains no executable code. Validation is performed by F6 scripts or standalone tier 1 checks.

---

## Complexity Tracking

**No violations found. This section is intentionally empty.**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|---|---|---|
| — | — | — |

---

## Acceptance Scenarios

| ID | Scenario | Given | When | Then |
|---|---|---|---|---|
| **AS-01** | All JSON files parse without error | All three sample JSON files exist in `skills/rcsa/assets/sample-data/` | Each file is loaded with `json.load()` | Parse succeeds with 0 errors for all three files |
| **AS-02** | Control library contains all four controls | `sample_control_library.json` is loaded | Controls array is inspected | Exactly 4 controls present: AC, CM, DQ, IH — each with `id`, `name`, `objective`, and ≥2 `evidence_types` |
| **AS-03** | Artifact index provides demo-scale evidence | `sample_artifact_index.json` is loaded | Artifacts array is inspected | Contains 10–20 artifacts with `id`, `file_path`, `file_type`, `description` fields; all file paths are relative |
| **AS-04** | Test catalog provides demo-scale tests | `sample_test_catalog.json` is loaded | Tests array is inspected | Contains 5–10 tests with `id`, `file_path`, `file_type`, `description`, `controls_relevant` fields; all file paths are relative |
| **AS-05** | Evidence coverage matches design exactly | Both sample data files are cross-referenced against control library | Evidence types are mapped per control | AC: 3/3 matched, CM: 3/3 matched, DQ: 1/2 matched (gap: `data_transform_test`), IH: 1/3 matched (gaps: `incident_response_plan`, `postmortem_record`) |
| **AS-06** | Narratives template is structurally valid | `rcsa_control_narratives_template.md` is inspected | Document structure is checked | Contains: YAML front matter with all defined fields, summary table with columns for Control/Confidence/Evidence Found/Gaps, four per-control sections each with gap callout area and citation placeholders |
| **AS-07** | Validation report template is structurally valid | `validation_report_template.md` is inspected | Document structure is checked | Contains: YAML front matter with citation statistics fields, Citation Index table, Evidence Coverage table, Flagged for Human Review section |
| **AS-08** | Citation format spec is complete | `citation_format.md` is inspected | Document content is checked | Defines bracket-index syntax `[XX-N]`, resolution rules mapping citation tags to file paths, and example citations for at least 2 controls |
| **AS-09** | No sensitive data in any file | All files in `assets/` are reviewed | Content is scanned for PII, secrets, absolute paths, real Synchrony identifiers | Zero instances found |
| **AS-10** | All files are UTF-8 encoded | All files in `assets/` are checked | Encoding is verified | All files are valid UTF-8 (FR-012) |
| **AS-11** | Asset directory contains exactly the expected files | `skills/rcsa/assets/` directory listing is checked | File inventory is compared to spec | Exactly 6 files in the defined structure — no extra files, no missing files |
| **AS-12** | Template placeholders use consistent syntax | Both template files are inspected | Placeholder patterns are checked | All placeholders use a single consistent syntax (e.g., `{{field_name}}`) that F6 can programmatically find-and-replace |

---

## Risks & Mitigations

| ID | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| **R-01** | **Sample data doesn't exercise F6's edge cases** — F6 may need evidence patterns not anticipated in F4 | Medium | High | Design sample data with explicit coverage matrix (AS-05). **Mandatory re-validation gate**: after F6's plan is approved, cross-reference F6's input expectations against F4's sample data. Add supplemental sample files if gaps are found. This is a blocking gate before F6 implementation begins. |
| **R-02** | **Template placeholder syntax misaligns with F6's output assembler** | Medium | Medium | AS-12 enforces consistent placeholder syntax. `citation_format.md` is the single source of truth. F6's plan must reference F4's format spec — not define its own. |
| **R-03** | **Control library evidence types too generic for deterministic matching** | Low | High | Evidence types designed with concrete artifact categories. Validate during F6 planning that each type maps to a distinguishable file pattern. Revise F4 if needed. |
| **R-04** | **Sample data accidentally contains sensitive information** | Low | High | AS-09 requires explicit review. Pre-commit checklist: scan all JSON for patterns matching API keys, emails, SSNs, IP addresses. Peer review required before merge. |
| **R-05** | **Citation format insufficient for complex evidence** | Low | Medium | Current format supports `file_path` + `lines` per citation. Multi-file evidence uses multiple citation tags (e.g., `[AC-1a]`, `[AC-1b]`). Document as extension rule in `citation_format.md`. |
| **R-06** | **F5 references asset paths that change after lock** | Medium | Medium | F4's directory structure locked in Section 4. Any post-approval changes require updating F5's references. Cross-feature dependency note added to F5's plan. |
| **R-07** | **YAML front matter fields misalign with F5's LLM instructions** — F5 tells the LLM to produce fields that don't match the template | Medium | Medium | After F5 is built, cross-reference F5's output instructions against F4's template YAML fields. Reconcile before F6 implementation. Include in the R-01 re-validation gate. |
| **R-08** | **Demo-scale assumptions misunderstood by Synchrony** — client expects production-scale sample data | Low | Medium | Document demo-scale assumption explicitly in sample data files (top-level `"_meta"` field in each JSON: `"scale": "demo"`, `"note": "Replace with production data post-handoff"`). |

---

## Dependencies

### Upstream Dependencies (F4 depends on)

| Dependency | Source | Status | Impact if Missing |
|---|---|---|---|
| Four control names and demo scope definition | Constitution | ✅ Available | Cannot define control library |
| File-based storage decision | Constitution + Project Spec | ✅ Available | Cannot determine asset format |
| `skills/rcsa/` directory structure convention | Project Spec | ✅ Available | Cannot determine file paths |

**Note**: F5 (SKILL.md) is **not** an upstream dependency for F4. F4 defines templates and formats; F5 references them. The dependency direction is F5 → F4. However, a post-build coordination checkpoint exists (R-07) to verify alignment between F5's LLM instructions and F4's template fields.

### Downstream Dependencies (other features depend on F4)

| Consumer | What They Need from F4 | Contract |
|---|---|---|
| **F5 (SKILL.md)** | Asset file paths | Directory structure locked (Project Structure section) |
| **F5 (SKILL.md)** | YAML front matter fields + template structure | Narratives and validation report templates locked |
| **F5 (SKILL.md)** | Citation format syntax | `citation_format.md` — bracket-index `[XX-N]` locked |
| **F6 (Scripts)** | Control library JSON schema | `sample_control_library.json` schema locked |
| **F6 (Scripts)** | Artifact index + test catalog JSON schemas | Both schemas locked |
| **F6 (Scripts)** | Template placeholder syntax | Consistent format locked via AS-12 |
| **F6 (Scripts)** | Sample data for integration testing | Three JSON files with designed coverage distribution locked |

### External Dependencies

None. Zero runtime dependencies, no third-party packages, no external services. This property must be preserved — use Python stdlib only for any validation during development.

### Cross-Feature Re-validation Gates

1. **After F5 build** → verify F5's LLM output instructions match F4's template fields (R-07)
2. **After F6 plan approval** → verify F6's input expectations match F4's sample data (R-01). **Blocking gate** before F6 implementation begins.

---

## Phases & Milestones

### Phase 1: Build F4 Assets

*Build order: M1.1 is the foundation — all subsequent milestones reference the control library. M1.2 and M1.3 depend on M1.1. M1.4 depends on M1.1 + M1.2 + M1.3. Remaining milestones can proceed in parallel after M1.1.*

| Milestone | Deliverable | Acceptance Check | Size | Depends On |
|---|---|---|---|---|
| M1.1 — Control Library | `sample_control_library.json` — 4 controls, 11 evidence types | AS-01, AS-02 | M | — |
| M1.2 — Sample Artifact Index | `sample_artifact_index.json` — 10–20 artifacts, designed coverage | AS-01, AS-03 | M | M1.1 |
| M1.3 — Sample Test Catalog | `sample_test_catalog.json` — 5–10 tests with `controls_relevant` | AS-01, AS-04 | S | M1.1 |
| M1.4 — Evidence Coverage Verification | Cross-reference all three JSON files against coverage design | AS-05 | S | M1.1, M1.2, M1.3 |
| M1.5 — Citation Format Spec | `citation_format.md` — bracket-index syntax, resolution rules, examples | AS-08 | S | M1.1 |
| M1.6 — Narratives Template | `rcsa_control_narratives_template.md` — YAML front matter, summary table, per-control sections | AS-06, AS-12 | M | M1.1, M1.5 |
| M1.7 — Validation Report Template | `validation_report_template.md` — citation index, evidence coverage, flagged-for-review | AS-07, AS-12 | M | M1.1, M1.5 |
| M1.8 — Security & Encoding Review | Scan all files for sensitive data, verify UTF-8, verify directory inventory | AS-09, AS-10, AS-11 | S | All above |

**Phase 1 Exit Criteria**: All 12 acceptance scenarios pass. Peer review complete. F4 is shippable as a standalone asset package.

### Phase 2: Cross-Feature Re-validation (event-driven — no idle waiting)

| Gate | Trigger | Action | Outcome |
|---|---|---|---|
| G2.1 — F5 Alignment Check | F5 build complete | Compare F5's LLM output instructions against F4's template fields and citation format. Reconcile mismatches. | Confirmed aligned — or F4 patched |
| G2.2 — F6 Compatibility Check | F6 plan approved | Compare F6's input expectations against F4's JSON schemas and sample data. Verify edge case coverage. **Blocking gate** — F6 implementation cannot start until this passes. | Confirmed compatible — or F4 supplemented |

**Phase 2 Exit Criteria**: Both gates pass. Any F4 patches re-validated against all Phase 1 acceptance scenarios.

---

## Design Decisions Log

*Decisions made during the planning interview, locked for implementation reference.*

### Control Library — 4 Controls, 11 Evidence Types

| Control ID | Control Name | Objective | Evidence Types |
|---|---|---|---|
| **AC** | Access Control | Ensure system access is restricted to authorized users through authentication, authorization, and session management | `auth_config`, `rbac_policy`, `session_management` |
| **CM** | Change Management | Ensure all changes to production systems follow approved processes with rollback capability | `change_approval_record`, `deployment_config`, `rollback_procedure` |
| **DQ** | Data Quality | Ensure data inputs are validated and transformations preserve integrity | `input_validation_rule`, `data_transform_test` |
| **IH** | Incident Handling | Ensure security incidents are detected, responded to, and reviewed | `monitoring_config`, `incident_response_plan`, `postmortem_record` |

### Sample Data Coverage Design

| Control | Sample Evidence Provided | Intentional Gaps |
|---|---|---|
| **AC** | All 3 types present | None — full coverage |
| **CM** | All 3 types present | None — full coverage |
| **DQ** | `input_validation_rule` present | `data_transform_test` missing — gap flagged |
| **IH** | `monitoring_config` present | `incident_response_plan` and `postmortem_record` missing — gaps flagged |

### Citation Format

- Bracket-index syntax: `[XX-N]` where `XX` = control ID, `N` = sequential number
- Each citation resolves to a `file_path` + optional `lines` range
- Multi-file evidence uses sub-indices: `[AC-1a]`, `[AC-1b]`

### Narratives Template Structure

1. **YAML front matter**: title, generated timestamp, skill version, total controls, overall confidence
2. **Summary table**: Control | Confidence Tier | Evidence Found | Gaps Identified
3. **Per-control sections**: heading, narrative paragraph with inline `[XX-N]` citations, gap callout block (if applicable), evidence list

### Validation Report Template Structure

1. **YAML front matter**: title, generated timestamp, total/valid/invalid citations, resolution rate
2. **Citation Index table**: Citation | File Path | Lines | Status (✅ Valid / ❌ Invalid)
3. **Evidence Coverage table**: Control | Expected Types | Matched | Unmatched | Coverage %
4. **Flagged for Human Review**: bullet list of specific gaps requiring follow-up

### Sample Data JSON Schemas

**`sample_control_library.json`**:
```json
{
  "_meta": { "scale": "demo", "note": "Replace with production data post-handoff" },
  "controls": [
    {
      "id": "AC",
      "name": "Access Control",
      "objective": "...",
      "evidence_types": [
        { "id": "auth_config", "label": "Authentication Configuration", "description": "..." }
      ]
    }
  ]
}
```

**`sample_artifact_index.json`**:
```json
{
  "_meta": { "scale": "demo", "note": "Replace with production data post-handoff" },
  "artifacts": [
    {
      "id": "ART-001",
      "file_path": "auth/oauth_config.yaml",
      "file_type": "yaml",
      "description": "...",
      "lines": "1-42"
    }
  ]
}
```

**`sample_test_catalog.json`**:
```json
{
  "_meta": { "scale": "demo", "note": "Replace with production data post-handoff" },
  "tests": [
    {
      "id": "TST-001",
      "file_path": "tests/auth/test_rbac_permissions.py",
      "file_type": "python",
      "description": "...",
      "controls_relevant": ["AC"]
    }
  ]
}
```

**Design decision**: `controls_relevant` is included in the test catalog as a self-documenting hint. F6 may validate or override this mapping at runtime. The field makes sample data independently understandable without running the pipeline.
