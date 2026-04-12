---
description: "Task list for F5 — SKILL.md: RCSA Control Narrative Generation"
---

# Tasks: F5 — SKILL.md: RCSA Control Narrative Generation

**Input**: Design documents from `/specs/f5-skill-rcsa/` — plan.md, research.md, data-model.md, quickstart.md, contracts.md
**Prerequisites**: plan.md (required), spec-v0.3.md (required for user stories), research.md, data-model.md, contracts.md
**F4 Dependency**: F4 (Assets — RCSA) must be complete. F5 references F4's templates, sample data, control library, and citation format.
**F6 Dependency**: F6 (Scripts — RCSA) does NOT need to be complete. F5 defines orchestration instructions that call F6 scripts, but F5 can be written and manually tested before F6 is built.

**Tests**: Manual evaluation tasks are included in Phase 7. Since F5 is a natural language file (not executable code), testing means running the SKILL.md in Claude with sample data and checking outputs against the performance gates in plan.md.

**Organization**: Tasks are grouped by user story to enable independent implementation and review of each story.

**Definition of Done**: A task is done when the relevant SKILL.md section is written, aligns with contracts.md, and passes manual review against the mapped FR and SC.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different sections, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2)
- Exact file paths included in descriptions

## Path Conventions

This project uses the Claude Agent Skills architecture:

- **SKILL.md location**: `skills/rcsa/SKILL.md`
- **Assets (F4)**: `skills/rcsa/assets/`
- **Scripts (F6)**: `skills/rcsa/scripts/`

## User Story Map

| Story | Title | Priority | Phases |
|---|---|---|---|
| US1 | Generate Citation-Backed Control Narratives | P1 | Phase 3 |
| US2 | Detect and Flag Compliance Gaps | P1 | Phase 4 |
| US3 | Produce Summary Table | P1 | Phase 5 |
| US4 | Follow Output Templates | P2 | Phase 6 |

## FR / SC Traceability

| FR | Description | Tasks |
|---|---|---|
| FR-001 | Inline citations in narratives | T008, T009 |
| FR-002 | GAP flagging | T009, T011 |
| FR-003 | Never imply compliance without proof | T007, T009, T010 |
| FR-004 | Summary table showing evidence vs. gaps | T012 |
| FR-005 | All four controls in every run | T006, T012 |
| FR-006 | Only provided evidence — no external knowledge | T006, T009, T010 |
| FR-007 | Citations reference real artifacts | T005, T008, T013 |
| FR-008 | Prefer GAP over uncertain claims | T007, T009, T010, T011 |
| FR-009 | Follow output templates | T014, T015, T021 |
| FR-010 | Orchestration sequence | T004, T013, T016, T022 |

| SC | Description | Tasks |
|---|---|---|
| SC-001 | ≥85% gap detection accuracy | T007, T010, T011 |
| SC-002 | ≤5% hallucinated artifacts | T005, T008, T010, T013 |
| SC-003 | All 4 controls addressed | T004, T006, T009, T011, T012 |
| SC-004 | ≥90% structural consistency | T014, T019, T020 |
| SC-005 | All required sections present | T012, T014, T015, T016, T021, T022 |

---

## Phase 1: Setup

**Purpose**: Create the SKILL.md file and establish the frontmatter block.

- [ ] T001 [US1] Create `skills/rcsa/SKILL.md` and write the **YAML frontmatter** block. Must include:
  - `name`: RCSA Control Narrative Generation
  - `description`: Brief description of what the skill does — generates citation-backed RCSA (Risk and Control Self-Assessment) control narratives from pre-processed mapped evidence
  - `trigger_phrases`: List of natural language phrases that activate this skill (e.g., "generate RCSA narratives", "run RCSA control analysis", "create control narratives")
  - Frontmatter must be valid YAML between `---` delimiters
  - **FR mapped**: FR-010 (first element of the orchestration file)
  - **SC mapped**: SC-005 (frontmatter is a required section)
  - **Done when**: `skills/rcsa/SKILL.md` exists with valid YAML frontmatter containing name, description, and trigger phrases.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Establish the 6-step orchestration skeleton and universal error handling rules that all subsequent phases build on.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T002 [US1] Write the **6-step orchestration skeleton** in SKILL.md. This is a high-level outline that names each step, states what it does, and identifies whether it is a script call `[SCRIPT]` or LLM reasoning `[LLM]`:
  - Step 1: Validate Inputs `[SCRIPT]` — call `validate_inputs` to check that all required files are present and parseable
  - Step 2: Build Artifact Registry `[SCRIPT]` — call `build_registry` to scan repository files and produce a structured artifact list
  - Step 3: Map Evidence to Controls `[SCRIPT]` — call `map_evidence` to associate artifacts with controls
  - Step 4: Generate Narratives `[LLM]` — reason over mapped evidence to produce per-control narratives with citations and confidence tiers
  - Step 5: Validate Citations `[SCRIPT]` — call `validate_citations` to verify all citations reference real artifacts
  - Step 6: Assemble Final Output `[LLM]` — combine all results into two output Markdown files
  - Each step must state its input (what it receives from the previous step) and output (what it passes to the next step)
  - **FR mapped**: FR-010 (defines the orchestration sequence)
  - **SC mapped**: SC-005 (skeleton establishes the required structure)
  - **Done when**: SKILL.md contains a numbered 6-step outline with step names, types, and input/output descriptions.

- [ ] T003 [US1][US2][US3][US4] Write the **Universal Error Handling Rules** section in SKILL.md. This section applies to all 6 steps and must include:
  - If any `[SCRIPT]` step returns an error, **stop immediately** and report the exact error message to the user. Do not proceed to the next step. Do not attempt to work around the error. (Source: constitution Principle 2 — never silent errors, explicit actionable messages)
  - Never fall back to LLM reasoning when a script fails. The LLM must not attempt to replicate what the script would have done. (Source: constitution — no LLM fallback)
  - If a `[LLM]` step encounters data it cannot interpret, flag the specific issue and ask the user for guidance. Do not guess.
  - All error messages must include: which step failed, what the error was, and what the user should do next
  - **FR mapped**: FR-010 (error handling is part of orchestration)
  - **SC mapped**: SC-003 (errors must not silently drop controls from the output)
  - **Done when**: SKILL.md contains a clearly labeled "Error Handling" section with the stop-on-error rule, no-LLM-fallback rule, LLM uncertainty rule, and error message format.

**Checkpoint**: SKILL.md has frontmatter, a 6-step skeleton, and universal error handling. User story implementation can now begin.

---

## Phase 3: User Story 1 — Generate Citation-Backed Control Narratives (Priority: P1) 🎯 MVP

**Goal**: Write the detailed SKILL.md instructions for Steps 1–4 of the orchestration, plus the confidence rubric and citation format rules. After this phase, the SKILL.md can instruct the LLM to take structured input and produce draft narratives with citations and confidence tiers.

**Independent Test**: Provide sample mapped evidence for all four controls (Access Control, Change Management, Data Quality, Incident Handling). Verify the LLM produces a 3–5 sentence narrative per control with inline citations referencing real artifacts, and assigns the correct confidence tier per the rubric.

### Implementation for User Story 1

- [ ] T004 [US1] Write Step 1 instructions in SKILL.md — **Validate Inputs** `[SCRIPT]`. Instructions must tell the LLM to:
  - Call `validate_inputs` with `control_library.yaml` and the raw repository file listing
  - Wait for the result before proceeding
  - If `status: "pass"`, proceed to Step 2
  - If `status: "fail"`, stop and report the exact errors from the `errors` list to the user
  - Must align with contracts.md Section 1.1 (Input Validation Result schema)
  - **FR mapped**: FR-010 (partial — first step of the orchestration sequence)
  - **SC mapped**: SC-003 (validation ensures all 4 controls are present in the input)
  - **Done when**: SKILL.md Step 1 section is written, references the correct script name and input/output format per contracts.md, and includes the pass/fail branching logic.

- [ ] T005 [US1] Write Step 2 instructions in SKILL.md — **Build Artifact Registry** `[SCRIPT]`. Instructions must tell the LLM to:
  - Call `build_registry` with raw repository files and `control_library.yaml`
  - Wait for the result
  - Receive `artifact_registry.yaml` — a list of artifacts with `id`, `file_path`, `artifact_type`, `snippet`, and `snippet_line_range`
  - If the registry is empty (`artifacts: []`), note that all controls will be flagged as GAP in Step 4 — but still proceed
  - Must align with contracts.md Section 1.1 (Artifact Registry schema) and data-model.md Section 1.2
  - **FR mapped**: FR-007 (partial — the registry provides the artifact identifiers the LLM will cite)
  - **SC mapped**: SC-002 (the registry is the source of truth for valid citations)
  - **Done when**: SKILL.md Step 2 section is written, references the correct script name and output schema per contracts.md, and includes the empty-registry handling.

- [ ] T006 [US1] Write Step 3 instructions in SKILL.md — **Map Evidence to Controls** `[SCRIPT]`. Instructions must tell the LLM to:
  - Call `map_evidence` with `artifact_registry.yaml` and `control_library.yaml`
  - Wait for the result
  - Receive `mapped_evidence.yaml` — each control with its `mapped_artifacts` list (may be empty) and `relevance` indicators (`"direct"` or `"indirect"`)
  - Understand that every control in the control library will appear in the mappings, even if `mapped_artifacts` is empty
  - Must align with contracts.md Section 1.1 (Mapped Evidence schema) and data-model.md Section 1.3
  - **FR mapped**: FR-005 (all four controls appear in the mappings), FR-006 (evidence comes from the pipeline, not external knowledge)
  - **SC mapped**: SC-003 (every control is represented in the mapped evidence)
  - **Done when**: SKILL.md Step 3 section is written, references the correct script name and output schema per contracts.md, and explains the relevance field.

- [ ] T007 [P] [US1] Write the **Confidence Tier Rubric** section in SKILL.md. This is a standalone reference section the LLM consults during Step 4. Must include:
  - **HIGH**: ≥2 artifacts with `"direct"` relevance supporting the control objective
  - **MEDIUM**: 1 artifact with `"direct"` relevance, OR ≥2 artifacts with `"indirect"` relevance
  - **LOW**: 1 artifact with `"indirect"` relevance only
  - **GAP**: `mapped_artifacts` is empty — zero artifacts mapped (GAP is a separate flag, not a confidence tier)
  - Edge case table from research.md Section 4.4 (e.g., 2 artifacts where 1 direct + 1 indirect = HIGH; metadata contradicts snippet = drop one tier)
  - Rule: if the snippet contradicts the artifact's metadata, lower confidence by one tier and note the mismatch
  - Rule: never inflate confidence — if evidence is weak, say so
  - Must align with spec SC-001 (≥85% gap detection) and the confidence tier definitions in spec-v0.3.md
  - **FR mapped**: FR-003 (never imply compliance without proof), FR-008 (prefer GAP over uncertain claims)
  - **SC mapped**: SC-001 (gap detection accuracy depends on correct rubric application)
  - **Done when**: SKILL.md contains a clearly labeled "Confidence Tier Rubric" section with all four tiers, the edge case table, and both rules.

- [ ] T008 [P] [US1] Write the **Citation Format Rules** section in SKILL.md. This is a standalone reference section the LLM consults during Step 4. Must include:
  - Citation format: `[file_path — description]` (space-emdash-space)
  - `file_path` must match exactly as provided in the artifact registry — no modifications
  - `description` is a brief phrase describing what the artifact shows (for auditor readability)
  - Only cite artifacts present in the artifact registry — never invent citations
  - Each factual claim in the narrative should trace to a specific artifact
  - Multi-control artifacts: cite in every control where they appear, note the overlap
  - Must align with contracts.md Section 1.2 (F5 provides citations in this format so F6 can parse them)
  - **FR mapped**: FR-001 (inline citations), FR-007 (exact identifiers from registry)
  - **SC mapped**: SC-002 (≤5% hallucinated artifacts depends on strict citation rules)
  - **Done when**: SKILL.md contains a clearly labeled "Citation Format" section with the format definition, all rules, and the multi-control artifact instruction.

- [ ] T009 [US1] Write Step 4 instructions in SKILL.md — **Generate Narratives** `[LLM]`. This is the core reasoning step. Depends on T004–T008. Instructions must tell the LLM to:
  - For each control in `mapped_evidence.yaml`:
    - Read the control's `objective` from the control library
    - Read the `mapped_artifacts` list and look up each artifact's `snippet` in the artifact registry
    - If `mapped_artifacts` is empty → assign GAP flag, write a GAP statement (what evidence types are missing), skip narrative generation for this control
    - If `mapped_artifacts` is not empty → write a 3–5 sentence auditor-friendly narrative that synthesizes the evidence, include inline citations per the Citation Format Rules, assign a confidence tier per the Confidence Tier Rubric
    - Trust the snippet as ground truth — if snippet contradicts artifact metadata, trust the snippet
    - Exclude ambiguous evidence and flag it for human review (do not include with caveats)
  - Use only the evidence provided — no external knowledge or assumptions about the repository (constitution Never-Ever Rules)
  - Narrative tone: professional, auditor-friendly, factual — no hedging language, no speculation
  - **FR mapped**: FR-001 (narrative generation), FR-002 (GAP flagging), FR-003 (no compliance without proof), FR-006 (only provided evidence), FR-008 (prefer GAP over uncertain claims)
  - **SC mapped**: SC-001 (gap detection), SC-002 (citation accuracy), SC-003 (all 4 controls addressed)
  - **Done when**: SKILL.md Step 4 section is written with the per-control loop, GAP branching logic, narrative instructions, and references to the Confidence Tier Rubric and Citation Format Rules sections.

**Checkpoint**: SKILL.md can instruct the LLM through Steps 1–4. Given structured input, the LLM can produce draft narratives with citations, confidence tiers, and GAP flags. Steps 5–6 (validation and assembly) are not yet written.

---

## Phase 4: User Story 2 — Detect and Flag Compliance Gaps (Priority: P1) 🎯 MVP

**Goal**: Write the SKILL.md sections that harden GAP detection and prevent false compliance claims. After this phase, the LLM has explicit rules for when to exclude evidence, when to flag gaps, and what a GAP statement looks like.

**Independent Test**: Provide mapped evidence where one control has zero artifacts and another has one artifact with ambiguous metadata. Verify the LLM flags both correctly — one as GAP, one as flagged for human review — and does not generate a narrative implying compliance for either.

### Implementation for User Story 2

- [ ] T010 [US2] Write the **Anti-Hallucination & Evidence Exclusion Rules** section in SKILL.md. This is a standalone reference section the LLM consults during Step 4. Must include:
  - **Hard rule**: Never imply compliance without proof. If evidence is insufficient, flag as GAP or lower the confidence tier — never fill the gap with reasoning or external knowledge.
  - **Exclusion rule**: If an artifact's snippet is ambiguous (does not clearly support the control objective), exclude it from the narrative and flag it for human review. Do not include it with caveats.
  - **No external knowledge**: The LLM must use only the artifacts and snippets provided by the pipeline. No assumptions about what the repository "probably" contains. No general knowledge about compliance frameworks. (Source: constitution Never-Ever Rules)
  - **No remediation suggestions**: When flagging a GAP, state what is missing — never suggest how to fix it. (Source: research.md)
  - **Zero false negatives target**: It is better to flag a control as GAP incorrectly than to claim compliance incorrectly. When in doubt, flag. (Source: spec SC-001, FR-008)
  - **FR mapped**: FR-003 (never imply compliance without proof), FR-006 (only provided evidence), FR-008 (prefer GAP over uncertain claims)
  - **SC mapped**: SC-001 (≥85% gap detection — zero false negatives is the priority), SC-002 (≤5% hallucinated artifacts)
  - **Done when**: SKILL.md contains a clearly labeled "Anti-Hallucination & Evidence Exclusion Rules" section with all five rules listed above.

- [ ] T011 [US2] Write the **GAP Statement Template** section in SKILL.md. This defines the exact structure the LLM must use when a control is flagged as GAP. Must include:
  - **When to use**: `mapped_artifacts` is empty for a control, OR all mapped artifacts were excluded per the evidence exclusion rules in T010
  - **GAP statement structure**:
    - Control ID and name
    - Confidence: `GAP`
    - Statement: "No evidence was identified for [control objective]. The following evidence types were expected but not found: [list expected evidence types from control library]."
    - No narrative section — GAP controls do not get a narrative
    - No remediation suggestions
  - **Edge case**: If a control had artifacts mapped but all were excluded as ambiguous, the GAP statement must note: "Artifacts were identified but excluded due to ambiguity. Flagged for human review."
  - Must align with data-model.md output schema (GAP controls appear in the summary table with `confidence: GAP`)
  - **FR mapped**: FR-002 (GAP flagging), FR-008 (prefer GAP over uncertain claims)
  - **SC mapped**: SC-001 (gap detection accuracy), SC-003 (all 4 controls appear in output — including GAP controls)
  - **Done when**: SKILL.md contains a clearly labeled "GAP Statement Template" section with the trigger condition, statement structure, edge case handling, and the no-remediation rule.

**Checkpoint**: SKILL.md now has defensive rules (anti-hallucination, evidence exclusion) and a defined GAP output format. Combined with Phase 3, the LLM can generate narratives for evidenced controls and properly flag gaps for unevidenced controls.

---

## Phase 5: User Story 3 — Produce Summary Table (Priority: P1) 🎯 MVP

**Goal**: Write the SKILL.md instructions for generating the summary table that appears at the top of `rcsa_control_narratives.md`. After this phase, the LLM knows how to produce an at-a-glance compliance overview before the detailed narratives.

**Independent Test**: Run the skill with sample data (2 evidenced controls, 2 GAP controls). Verify the output begins with a summary table listing all four controls with correct confidence tiers and status indicators.

### Implementation for User Story 3

- [ ] T012 [US3] Write the **Summary Table Generation** instructions in SKILL.md. This tells the LLM how to build the summary table after completing Step 4 (narrative generation). Must include:
  - **Table placement**: The summary table appears immediately after the YAML front matter, before any per-control sections
  - **Table columns** (per data-model.md Section 2.1):
    - Control ID (e.g., `AC`)
    - Control Name (e.g., "Access Control")
    - Confidence (`HIGH`, `MEDIUM`, `LOW`, or `GAP`)
    - Artifacts (count of artifacts cited in the narrative)
    - Status (visual indicator: `✅ Assessed` for HIGH/MEDIUM, `⚠️ Weak Evidence` for LOW, `🔴 No Evidence` for GAP)
  - **Row ordering**: One row per control, in the same order they appear in the control library input
  - **Completeness rule**: Every control in the input must appear in the table — no controls may be omitted, even if flagged as GAP
  - **Consistency rule**: The confidence tier and artifact count in the table must exactly match the values in the corresponding per-control section below. No discrepancies.
  - **Edge case**: If all four controls are GAP, the table still appears with four rows — all showing `GAP` / `0` / `🔴 No Evidence`
  - Must align with data-model.md Section 2.1 (Markdown Body Structure) and spec US3 acceptance scenarios
  - **FR mapped**: FR-004 (summary table showing evidence vs. gaps), FR-005 (all four controls in every run)
  - **SC mapped**: SC-003 (all 4 controls addressed), SC-005 (output follows defined template structure)
  - **Done when**: SKILL.md contains a clearly labeled "Summary Table" instruction section with the column definitions, row ordering rule, completeness rule, consistency rule, and the all-GAP edge case.

**Checkpoint**: SKILL.md now instructs the LLM to produce a summary table. Combined with Phases 3–4, the LLM can generate narratives, flag gaps, and present an at-a-glance overview. Output assembly (Step 6) and citation validation (Step 5) are not yet written.

---

## Phase 6: User Story 4 — Follow Output Templates (Priority: P2)

**Goal**: Write the SKILL.md instructions for citation validation (Step 5), output structure definitions for both deliverables, and final assembly (Step 6). After this phase, the SKILL.md is content-complete.

**Independent Test**: Run the skill end-to-end with sample data. Compare both output files against the structures defined in data-model.md Section 2. Verify all required sections are present and correctly ordered.

### Implementation for User Story 4

- [ ] T013 [US4] Write Step 5 instructions in SKILL.md — **Validate Citations** `[SCRIPT]`. Instructions must tell the LLM to:
  - Call `validate_citations` with the draft narratives from Step 4 and `artifact_registry.yaml`
  - Wait for the result
  - Receive the citation validation output: `total_citations`, `resolved`, `unresolved`, `resolution_rate`, and per-citation details
  - If `unresolved` is 0 → proceed to Step 6
  - If `unresolved` > 0 → fix the broken citations before proceeding. Options: remove the citation, replace with a valid artifact from the registry, or flag the issue. Never leave an unresolved citation in the final output.
  - Must align with contracts.md Section 1.3 (Citation Validation Results schema) and Section 1.4 (error handling for unresolved citations)
  - **FR mapped**: FR-007 (citations must reference real artifacts), FR-010 (Step 5 of the orchestration sequence)
  - **SC mapped**: SC-002 (≤5% hallucinated artifacts — Step 5 is the enforcement mechanism)
  - **Done when**: SKILL.md Step 5 section is written, references the correct script name and output schema per contracts.md, and includes the unresolved citation handling logic.

- [ ] T014 [US4] Write the **Output Structure: `rcsa_control_narratives.md`** section in SKILL.md. This defines the exact structure of the primary deliverable. Must include:
  - **YAML front matter fields** (per data-model.md Section 2.1): `title`, `generated`, `controls_assessed`, `controls_passing`, `controls_low_confidence`, `controls_gap`, `framework`
  - **Section ordering**: YAML front matter → Summary Table → per-control sections (one `## [control_id] — [control_name]` section per control)
  - **Per-control section layout**: Confidence tier, Artifacts Cited count, Evidence Types Covered list, then either a 3–5 sentence narrative with citations (for evidenced controls) or a GAP statement (for unevidenced controls)
  - **Formatting rules**: Markdown headings, table syntax, citation format — all must be consistent across runs
  - Must align with data-model.md Section 2.1 and contracts.md Section 2.1
  - **FR mapped**: FR-009 (follow output templates), FR-001 (narrative structure)
  - **SC mapped**: SC-004 (≥90% structural consistency), SC-005 (all required sections present)
  - **Done when**: SKILL.md contains a clearly labeled output structure section for `rcsa_control_narratives.md` with all fields, section ordering, per-control layout, and formatting rules.

- [ ] T015 [US4] Write the **Output Structure: `validation_report.md`** section in SKILL.md. This defines the exact structure of the QA deliverable. Must include:
  - **Header fields**: Generated timestamp, SKILL.md version, controls assessed count
  - **Citation Resolution table**: columns for Citation text, Artifact ID, File Path, Resolved (yes/no), Match Type — populated from Step 5 results
  - **Confidence Scoring Audit table**: columns for Control ID, Confidence, Direct Artifacts count, Indirect Artifacts count, Rubric Match explanation — shows how each tier was calculated
  - **Excluded Evidence section** (optional — only included when artifacts were excluded): Artifact ID, Control ID, Reason Excluded
  - **Warnings section** (optional — only included when warnings exist): e.g., low evidence type coverage for a control
  - Must align with data-model.md Section 2.2
  - **FR mapped**: FR-009 (follow output templates)
  - **SC mapped**: SC-005 (all required sections present)
  - **Done when**: SKILL.md contains a clearly labeled output structure section for `validation_report.md` with all sections, table columns, and the conditional inclusion rules for optional sections.

- [ ] T016 [US4] Write Step 6 instructions in SKILL.md — **Assemble Final Output** `[LLM]`. Depends on T013–T015. Instructions must tell the LLM to:
  - Build `rcsa_control_narratives.md` following the structure defined in T014:
    - Populate YAML front matter by counting controls in each category
    - Insert the summary table (built per T012)
    - Insert per-control sections in control library order
    - Incorporate any citation fixes from Step 5
  - Build `validation_report.md` following the structure defined in T015:
    - Populate the citation resolution table from Step 5 results
    - Populate the confidence scoring audit table from Step 4 results
    - Include excluded evidence section only if exclusions occurred
    - Include warnings section only if warnings exist
  - Both files must be complete, standalone Markdown documents — a reader should understand them without seeing the intermediate data
  - **FR mapped**: FR-009 (follow output templates), FR-010 (final step of orchestration)
  - **SC mapped**: SC-004 (structural consistency), SC-005 (all required sections present)
  - **Done when**: SKILL.md Step 6 section is written with assembly instructions for both output files, references the output structure sections from T014 and T015, and includes the citation fix incorporation rule.

**Checkpoint**: SKILL.md is content-complete. All 6 orchestration steps have detailed instructions. All reference sections (confidence rubric, citation format, anti-hallucination rules, GAP template, output structures) are written. Ready for manual evaluation.

---

## Phase 7: Manual Evaluation

**Purpose**: Verify the completed SKILL.md produces correct outputs by running it in Claude with sample data and checking against the performance gates from plan.md Section 6.

**Prerequisites**: Phases 1–6 complete. F6 scripts available (or sample data hand-crafted to simulate F6 output).

- [ ] T017 [US1][US2][US3][US4] **Happy path test**. Run the SKILL.md in Claude with the sample data from quickstart.md (AC = 3 artifacts, CM = 1 artifact, DQ = 0 artifacts, IH = 0 artifacts). Verify:
  - AC narrative: 3–5 sentences, 3 citations, confidence = HIGH
  - CM narrative: 3–5 sentences, 1 citation, confidence = LOW
  - DQ: GAP flag, no narrative, lists missing evidence types
  - IH: GAP flag, no narrative, lists missing evidence types
  - Summary table: all 4 controls, correct confidence tiers and status indicators
  - YAML front matter: correct counts (`controls_passing: 1`, `controls_low_confidence: 1`, `controls_gap: 2`)
  - `validation_report.md`: all citations resolved, confidence scoring audit matches rubric
  - All citations use `[file_path — description]` format
  - Use the full verification checklist from quickstart.md Step 4
  - **Performance gates checked**: Gap Detection ≥85%, Citation Accuracy ≥95%, Control Coverage 4/4, Narrative Quality ≥80%
  - **Done when**: Both output files pass all checklist items. Any failures are documented with the specific item that failed and what the LLM produced instead.

- [ ] T018 [US2] **Edge case tests**. Run the SKILL.md with each of the following scenarios (one run per scenario):
  - **All GAP**: All 4 controls have zero artifacts. Expect 4 GAP flags, no narratives, summary table shows all `🔴 No Evidence`.
  - **Ambiguous evidence**: One control has artifacts where the snippet contradicts the artifact metadata. Expect confidence dropped by one tier and mismatch noted.
  - **Single weak artifact**: One control has exactly 1 indirect artifact. Expect LOW confidence.
  - **Multi-control artifact**: One artifact mapped to 2 controls. Expect it cited in both narratives with overlap noted.
  - Document results for each scenario: pass/fail per acceptance criterion, any unexpected LLM behavior.
  - **Performance gates checked**: Gap Detection (zero false negatives is the priority), Citation Accuracy
  - **Done when**: All four edge case scenarios are run and documented. Any failures are logged with the scenario, expected outcome, and actual outcome.

- [ ] T019 [US4] **Structural consistency test**. Run the SKILL.md with the same happy-path input 5 times. Compare output structure across all 5 runs:
  - YAML front matter fields: same fields present in all runs
  - Summary table: same columns and row count in all runs
  - Per-control section layout: same headings and sub-elements in all runs
  - Narrative wording will vary — that's expected and acceptable
  - Calculate structural match percentage across the 5 runs
  - **Performance gate checked**: Structural Consistency ≥90%
  - **Done when**: 5 runs completed, structural comparison documented, match percentage calculated. If <90%, document which structural elements varied.

**Checkpoint**: SKILL.md has been tested against happy path, edge cases, and consistency. Performance gate results are documented. Ready for polish.

---

## Phase 8: Polish

**Purpose**: Final cross-cutting review, internal consistency check, and documentation cleanup. After this phase, SKILL.md is complete and ready for handoff to F6 implementation.

- [ ] T020 [US1][US2][US3][US4] **Cross-reference audit**. Review the completed SKILL.md end-to-end and verify:
  - Every script name referenced in SKILL.md matches contracts.md Section 1.1 exactly (spelling, parameters, return schemas)
  - Every asset file path referenced in SKILL.md matches the F4 asset paths from contracts.md Section 2.1
  - Every citation format example in SKILL.md uses `[file_path — description]` (space-emdash-space) — no variations
  - Every confidence tier reference matches the rubric in T007 — no contradictions between the rubric section and Step 4 instructions
  - The output structure sections (T014, T015) match data-model.md Section 2 exactly — same fields, same ordering
  - **FR mapped**: All FRs (cross-cutting consistency check)
  - **SC mapped**: SC-004 (structural consistency depends on internal consistency of the instructions)
  - **Done when**: Audit checklist completed. Any discrepancies found are fixed. Zero mismatches between SKILL.md and contracts.md/data-model.md.

- [ ] T021 [US1][US2][US3][US4] **Plain language review**. Review the completed SKILL.md for readability per constitution Principle 3:
  - All acronyms defined on first use (RCSA = Risk and Control Self-Assessment, GAP, LLM = Large Language Model, etc.)
  - No unnecessary jargon — a non-developer stakeholder should be able to read the workflow steps and understand what happens at each stage
  - Instructions are unambiguous — the LLM should not need to interpret vague language
  - Section headings are descriptive and scannable
  - No orphaned references (e.g., "see above" without specifying which section)
  - **FR mapped**: FR-009 (documentation quality)
  - **SC mapped**: SC-005 (output follows defined template — starts with clear instructions)
  - **Done when**: SKILL.md passes a readability review. Any unclear passages are rewritten.

- [ ] T022 [US1][US2][US3][US4] **Quickstart.md validation**. Run through quickstart.md end-to-end using the completed SKILL.md and verify:
  - Every step in quickstart.md still accurately describes how to run the skill
  - File paths in quickstart.md match the actual SKILL.md location
  - The verification checklist in quickstart.md aligns with the outputs SKILL.md actually produces
  - If quickstart.md needs updates based on the final SKILL.md, document the needed changes (do not modify quickstart.md unilaterally — it is a Phase 1 locked document)
  - **FR mapped**: FR-010 (the quickstart is the user-facing orchestration guide)
  - **SC mapped**: SC-005 (quickstart and SKILL.md must be in sync)
  - **Done when**: Quickstart.md walkthrough completed. Either confirmed accurate or discrepancies documented for review.

**Checkpoint**: SKILL.md is internally consistent, readable, and aligned with all upstream documents. F5 is complete and ready for F6 implementation to begin.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Phase 1 completion — BLOCKS all user stories
- **User Stories (Phases 3–6)**: All depend on Phase 2 completion
  - Phase 3 (US1) can start after Phase 2
  - Phase 4 (US2) can start after Phase 2 (T010 and T011 are standalone reference sections)
  - Phase 5 (US3) can start after Phase 3 (summary table references Step 4 output)
  - Phase 6 (US4) can start after Phase 3 (Step 5 validates Step 4 output)
- **Manual Evaluation (Phase 7)**: Depends on Phases 1–6 being complete
- **Polish (Phase 8)**: Depends on Phase 7 being complete (findings may require revisions)

### Task Dependencies Within Phases

- **T007 [P] and T008 [P]**: Can run in parallel with T004–T006 (different sections, no dependencies)
- **T009**: Depends on T004–T008 (Step 4 references all prior steps and both reference sections)
- **T010 and T011**: Can run in parallel (different sections)
- **T016**: Depends on T013–T015 (assembly references validation and output structures)

### Parallel Opportunities

```
# Phase 3 parallel work:
T004, T005, T006 can run sequentially (Steps 1→2→3)
T007 and T008 can run in parallel with T004–T006 (standalone reference sections)
T009 must wait for T004–T008

# Phase 4 parallel work:
T010 and T011 can run in parallel

# Phase 7 parallel work:
T017, T018, T019 can run in parallel (independent test scenarios)

# Phase 8 parallel work:
T020, T021, T022 can run in parallel (independent review activities)
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL — blocks all stories)
3. Complete Phase 3: User Story 1
4. **STOP and VALIDATE**: Run a quick manual test with sample data
5. Proceed to remaining phases

### Incremental Delivery

1. Complete Setup + Foundational → Skeleton ready
2. Add User Story 1 (Phase 3) → Narratives work → Quick validation
3. Add User Story 2 (Phase 4) → GAP detection hardened
4. Add User Story 3 (Phase 5) → Summary table works
5. Add User Story 4 (Phase 6) → Output assembly complete → SKILL.md content-complete
6. Manual Evaluation (Phase 7) → Performance gates verified
7. Polish (Phase 8) → Ready for F6 handoff

### Single-Author Strategy (Recommended for F5)

F5 is a single Markdown file written by one person. The recommended approach:

1. Work through phases sequentially (Phase 1 → 8)
2. Use parallel opportunities within phases where marked [P]
3. Commit after each phase checkpoint
4. Stop at Phase 3 checkpoint for early validation before continuing

---

## Task Summary

| Phase | Tasks | Count |
|---|---|---|
| Phase 1: Setup | T001 | 1 |
| Phase 2: Foundational | T002–T003 | 2 |
| Phase 3: US1 — Narratives | T004–T009 | 6 |
| Phase 4: US2 — Gap Detection | T010–T011 | 2 |
| Phase 5: US3 — Summary Table | T012 | 1 |
| Phase 6: US4 — Output Templates | T013–T016 | 4 |
| Phase 7: Manual Evaluation | T017–T019 | 3 |
| Phase 8: Polish | T020–T022 | 3 |
| **Total** | | **22** |

---

## Notes

- [P] tasks = different SKILL.md sections, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Commit after each phase checkpoint
- Stop at any checkpoint to validate independently
- F5 is a natural language file — tasks describe prose sections to write, not code to implement
- The only "code" in F5's scope is the script invocation syntax that SKILL.md uses to call F6 tools
- If a conflict is found between documents during implementation, surface it — do not resolve unilaterally
- Document priority order: constitution → spec → contracts → plan → research
