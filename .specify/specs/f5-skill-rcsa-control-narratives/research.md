# F5 — SKILL.md: RCSA Control Narrative Generation — Research

**Feature Branch**: `f5-rcsa-skill`
**Created**: 2026-04-10
**Status**: Draft
**Owner**: Whitney Robinson (PM), Sheila Green, Molly Lowell

---

## 1. Problem Statement

Synchrony's risk/compliance teams assess compliance posture manually — reading repository artifacts, deciding which artifacts relate to which controls, writing narratives explaining how the evidence supports each control, and flagging gaps where evidence is missing. This process is slow, inconsistent, and hard to audit because the reasoning behind compliance claims is rarely documented with traceable citations.

F5 addresses the LLM reasoning and orchestration layer of this problem. Given a set of controls and pre-processed evidence from F6's deterministic scripts, the SKILL.md instructs the LLM to:

1. Call F6 scripts to validate inputs, build the artifact registry, and map evidence to controls
2. Reason over the mapped evidence to generate per-control narratives with inline citations
3. Assign confidence tiers based on a defined rubric
4. Flag gaps where zero evidence exists
5. Call F6 scripts to validate that every citation resolves to a real artifact
6. Assemble the final output documents

The core challenge is not "can an LLM write a narrative" — it's "can we build an instruction file that reliably orchestrates a multi-step workflow where the LLM knows when to call a script, when to reason on its own, and when to stop and flag uncertainty."

---

## 2. Orchestration Design

This is the central design challenge for F5. The SKILL.md must tell the LLM exactly when to delegate to scripts and when to think for itself.

### 2.1 The Division of Labor

| Work Type | Who Does It | Why |
|-----------|-------------|-----|
| Input validation | F6 script | Deterministic — either the input is valid or it isn't |
| Artifact registry building | F6 script | Deterministic — parsing files and classifying artifacts is mechanical |
| Evidence-to-control mapping | F6 script | Deterministic — matching artifact types to control evidence types is rule-based |
| Narrative generation | LLM | Requires reasoning — synthesizing multiple artifacts into coherent prose |
| Confidence scoring | LLM (rubric-guided) | Requires judgment — but constrained by a deterministic rubric |
| Gap detection | LLM | Trivial reasoning — if mapped_artifacts is empty, flag GAP |
| Citation validation | F6 script | Deterministic — checking citations against the registry is a lookup |
| Final assembly | LLM | Requires reasoning — building the summary table and YAML front matter from the narrative results |

**Design principle:** The LLM only does what only an LLM can do. Everything else goes to scripts. This minimizes hallucination surface area and maximizes testability.

### 2.2 The 6-Step Sequence

The SKILL.md defines this exact sequence:

1. **F6: Validate Inputs** — Confirm the control library, artifact registry, and mapped evidence are structurally valid. If validation fails, stop. Don't attempt to reason over bad data.
2. **F6: Build Artifact Registry** — Parse the repository and produce the structured artifact inventory with snippets.
3. **F6: Map Evidence** — Match artifacts to controls based on evidence type alignment. Produce the mapped evidence structure.
4. **LLM: Generate Narratives** — For each control, read the mapped artifacts and their snippets. Write a 3–5 sentence narrative with inline citations. Assign a confidence tier using the rubric. Flag GAP if no artifacts are mapped.
5. **F6: Validate Citations** — Check every inline citation against the artifact registry. Flag any citation that doesn't resolve.
6. **LLM: Assemble Output** — Build the summary table, YAML front matter, and final document structure. If Step 5 found unresolved citations, fix them before finalizing.

**Why this order:** Steps 1–3 must complete before Step 4 because the LLM needs structured input to reason over. Step 5 must follow Step 4 because it validates the LLM's output. Step 6 must follow Step 5 because the final output must reflect validated citations.

### 2.3 What the SKILL.md Actually Says at Each Step

The SKILL.md doesn't just list the steps — it tells the LLM how to behave at each one:

- **Before calling a script:** "Call [script name] with [these inputs]. Wait for the result before proceeding."
- **After receiving script output:** "Check the result. If [error condition], stop and report the error. If [success condition], proceed to the next step."
- **During reasoning steps:** "For each control in the mapped evidence, do the following: [specific instructions]."
- **At decision points:** "If the mapped_artifacts list is empty, flag this control as [GAP] and skip narrative generation. Do not write a compliance narrative for a control with no evidence."

This level of explicitness is necessary because the LLM will otherwise make assumptions about what to do when things go wrong.

---

## 3. Evidence Handling Strategy

How the LLM consumes, cites, and excludes evidence is where the anti-hallucination design lives.

### 3.1 What the LLM Sees

The LLM receives three structured inputs:

- **Control Library** — what controls to assess, including the control objective (what the control is supposed to achieve) and expected evidence types
- **Artifact Registry** — every artifact found in the repo, with file paths, types, and content snippets
- **Mapped Evidence** — which artifacts map to which controls, with a relevance indicator (direct or indirect)

The LLM reads the content snippets to understand what each artifact actually does. It does not read full files — F6 extracts the relevant snippets.

### 3.2 Citation Rules

| Rule | Description | Why |
|------|-------------|-----|
| Only cite what's in the input | Never reference an artifact not in the artifact registry | Prevents hallucinated citations |
| Use descriptive citations | Format: `[file_path — brief description]` | Readable by auditors who may not know the codebase |
| One citation per claim | Each factual claim in the narrative should trace to a specific artifact | Enables per-claim verification |
| Citations are validated post-generation | F6 checks every citation against the registry in Step 5 | Catches any citations the LLM invented |

### 3.3 Ambiguity Handling

This was a key design decision from the interview. When evidence is ambiguous — the artifact might relate to the control, but it's not clear — the SKILL.md instructs the LLM to:

1. **Exclude the artifact** from the narrative
2. **Flag the uncertainty** for human review
3. **Do not include it with caveats** — a hedged citation is still a citation, and it might be wrong

**Why exclude rather than include:** In compliance contexts, implying evidence exists when it's uncertain is worse than flagging a gap. An auditor who sees a citation assumes it's been verified. A false citation undermines trust in the entire document. Excluding ambiguous evidence and flagging it for human review is the conservative, audit-safe approach.

### 3.4 Multi-Control Artifacts

When a single artifact maps to multiple controls (e.g., a CI/CD config that's relevant to both Change Management and Access Control):

- Cite it in both narratives
- Note the overlap in each narrative (e.g., "This artifact also provides evidence for [other control]")
- Count it as evidence for both controls in confidence scoring

This was confirmed during the interview. The alternative — citing it in only one control — would create artificial gaps.

### 3.5 Snippet as Ground Truth

The `content_snippet` in the artifact registry is the LLM's ground truth. If the artifact's metadata (type, description) says one thing but the snippet shows something different, the LLM trusts the snippet. This prevents cases where F6's classification is slightly off but the actual code tells the real story.

---

## 4. Confidence & Gap Detection Model

### 4.1 Confidence Tiers

| Tier | Criteria | What It Signals |
|------|----------|-----------------|
| **HIGH** | ≥2 artifacts with direct relevance supporting the control objective | Strong evidence — multiple artifacts corroborate the control |
| **MEDIUM** | 1 artifact with direct relevance, OR ≥2 artifacts with indirect relevance | Moderate evidence — something exists but coverage is incomplete |
| **LOW** | 1 artifact with indirect relevance only | Weak evidence — barely supports the control; needs human review |

### 4.2 GAP Flag

GAP is not a confidence tier. It is a separate binary flag:

- **Triggered when:** `mapped_artifacts` is empty for a control (zero artifacts)
- **What it means:** No evidence was found in the repository for this control
- **What the LLM does:** Writes a GAP statement (what's missing), not a compliance narrative (what exists)
- **What the LLM does NOT do:** Suggest remediation steps, speculate about evidence that might exist elsewhere, or imply the control might be satisfied

**Why separate from confidence:** A control with LOW confidence has weak evidence — something was found, it's just not strong. A control with a GAP flag has nothing. These are fundamentally different situations that require different responses from the auditor. Collapsing them into a single scale would obscure the distinction.

### 4.3 Scoring Rules

1. Count artifacts with direct relevance and indirect relevance separately
2. Apply the tier criteria from the table above
3. If the snippet contradicts the artifact's metadata, lower confidence by one tier and note the mismatch
4. Never inflate confidence — if the evidence is weak, say so
5. Every control gets exactly one outcome: a confidence tier (HIGH/MEDIUM/LOW) or a GAP flag, never both

### 4.4 Edge Cases

| Situation | Outcome | Reasoning |
|-----------|---------|-----------|
| 3 direct artifacts | HIGH | ≥2 direct → HIGH |
| 2 artifacts: 1 direct, 1 indirect | HIGH | ≥2 total with at least 1 direct |
| 2 indirect artifacts | MEDIUM | ≥2 but only indirect support |
| 1 direct artifact | MEDIUM | 1 direct → MEDIUM |
| 1 indirect artifact | LOW | 1 indirect → LOW |
| 0 artifacts | GAP | No evidence → GAP flag |
| Metadata says "access control" but snippet shows logging code | Drop one tier, note mismatch | Snippet is ground truth |
| Same artifact in 3 controls | Counts for all 3 | Multi-control artifacts are valid evidence for each |
| All 4 controls have 0 artifacts | 4 GAP flags | Don't invent evidence to look useful |

---

## 5. Alternatives Considered

### 5.1 LLM-Driven Evidence Mapping (Rejected)

**What it is:** Having the LLM perform the evidence-to-control mapping itself instead of receiving pre-mapped evidence from F6.

**Why rejected:** Evidence mapping is deterministic — matching artifact types to control evidence types is rule-based. Giving this to the LLM introduces hallucination risk (the LLM might map artifacts based on surface-level keyword matching rather than actual type alignment). Constitution Principle 1 (Accuracy Over Speed) and the project spec's division of labor both specify that deterministic logic belongs in scripts.

### 5.2 Remediation Suggestions in Gap Flags (Rejected)

**What it is:** When the LLM flags a GAP, it also suggests what evidence would close the gap.

**Why rejected:** Remediation requires domain knowledge about Synchrony's specific compliance environment. The LLM doesn't have this context. Suggesting remediation could imply the system understands the organization's compliance posture, which it doesn't. Scoped out for the May 7th demo — prove detection works before adding guidance.

**Revisit condition:** Post-demo, the control library input could include "expected evidence types" that the SKILL.md references when flagging gaps.

### 5.3 Multi-File SKILL.md (Rejected)

**What it is:** Splitting the rubric, worked examples, and output templates into separate files alongside the SKILL.md.

**Why rejected:** The content volume is manageable in a single file (6-step sequence, 3-tier rubric, 3 worked examples). Splitting adds runtime file-loading dependencies. F4 will formalize templates later — during F5 development, keeping everything inline reduces coordination overhead. Constitution Principle 3 (Simplicity First).

**Revisit condition:** If the skill grows post-demo (more controls, more complex rubric, additional reference data).

### 5.4 Separate Gap Report (Rejected)

**What it is:** A third output document dedicated to gap analysis.

**Why rejected:** Gap flags are most useful inline alongside the narratives — an auditor reading the control narratives sees immediately which controls have evidence and which don't. The summary table already provides the at-a-glance gap overview. A third document adds complexity without clear value.

### 5.5 Include Ambiguous Evidence with Caveats (Rejected)

**What it is:** When evidence is ambiguous, include it in the narrative with hedging language rather than excluding it.

**Why rejected:** A hedged citation is still a citation. An auditor who sees it may assume it's been verified. In compliance contexts, implying evidence exists when it's uncertain is worse than flagging a gap. Excluding and flagging for human review is the conservative, audit-safe approach. Constitution Principle 1 (Accuracy Over Speed).

---

## 6. Existing Tools Landscape

Several GRC (Governance, Risk, Compliance) platforms exist for compliance management:

| Tool | What It Does | Why It Doesn't Solve F5's Problem |
|------|-------------|-----------------------------------|
| AuditBoard | Enterprise RCSA lifecycle management | Doesn't generate narratives from repository artifacts |
| ServiceNow GRC | Integrated risk/compliance workflows | Enterprise platform — heavy infrastructure, not file-based |
| Hyperproof | Evidence collection and control mapping | Manages evidence but doesn't generate auditor-facing narratives |
| Drata | Continuous compliance monitoring (SOC 2/ISO) | Automated evidence collection but no narrative generation from code |

**Why we're building our own:** No existing tool generates citation-backed, confidence-scored narratives from repository artifacts without infrastructure. F5 proves the LLM-based approach works as a file-based prototype that Synchrony can extend post-handoff.

**What we learned from the landscape:** Summary tables are standard (F5 includes one). Gap flagging is table stakes (F5's GAP flags align with industry expectations). Human review is always required (F5's confidence tiers and GAP flags formalize this).

---

## 7. Testing Strategy

### 7.1 Standalone Testing (F5 Only)

Run the SKILL.md against hand-crafted sample evidence in Claude Desktop. Tests:

- Narrative quality and citation accuracy
- Confidence rubric adherence
- GAP flag triggering (zero false negatives)
- Ambiguous evidence exclusion
- Summary table accuracy

**Test data:** Sample mapped evidence for all four demo controls:
- Access Control — 3 direct artifacts (expect HIGH)
- Change Management — 1 direct artifact (expect MEDIUM or LOW)
- Data Quality — 0 artifacts (expect GAP)
- Incident Handling — 0 artifacts (expect GAP)

### 7.2 End-to-End Testing (Full Pipeline)

Run the complete F6 → F5 → F6 pipeline. Tests:

- F6's evidence mapping produces correct structured input
- F5's output integrates with F6's citation validation
- Final documents match expected format
- Citation resolution rate is 100%

### 7.3 Red Teaming

Feed intentionally bad inputs to verify graceful degradation:

- All controls with zero evidence (should produce 4 GAP flags)
- Controls with only ambiguous evidence (should exclude and flag)
- Malformed input structure (should stop and report error)
- Artifact with metadata that contradicts its snippet (should trust snippet, lower confidence)

### 7.4 Ground Truth Evaluation

Manually score F5's output against success criteria. Establish baseline performance and set thresholds before demo day.

---

## 8. Design Decisions Log

Complete record of decisions made during the research interview.

| # | Decision | Choice | Rationale |
|---|----------|--------|-----------|
| 1 | Orchestration model | SKILL.md calls F6 scripts + reasons on its own | LLM only does what only an LLM can do; everything else is scripts |
| 2 | Gap handling | Flag only — no remediation | Remediation requires domain knowledge the LLM doesn't have |
| 3 | Confidence tiers | HIGH / MEDIUM / LOW | Three tiers with a deterministic rubric for repeatability |
| 4 | GAP flag | Separate from confidence — binary flag | GAP = zero evidence (fundamentally different from LOW = weak evidence) |
| 5 | Framework approach | Framework-agnostic | Controls provided as input; not tied to any specific compliance framework |
| 6 | Demo controls | Access Control, Change Management, Data Quality, Incident Handling | Four generic controls that demonstrate the pattern |
| 7 | Output documents | Two: narratives + validation report | Narratives for auditors, validation for QA |
| 8 | Output format | Markdown with YAML front matter | Human-readable, version-controllable |
| 9 | Evidence input | File paths + content snippets (structured YAML) | Enough context for narratives without sending full files |
| 10 | Weak evidence | Write narrative, assign LOW confidence | Provides value while being transparent about weakness |
| 11 | Multi-control artifacts | Cite in both, note overlap | Conservative but complete |
| 12 | Ambiguous mappings | Exclude, flag for human review | Anti-hallucination; accuracy over coverage |
| 13 | File structure | Single SKILL.md | Content volume manageable; Simplicity First |
| 14 | Build order | F5 → F4 → F6 | High-risk LLM dependency first |
| 15 | Citation format | Descriptive in narratives, technical in validation report | Narratives for auditors, validation for QA |

---

## 9. Open Research Questions

| # | Question | Status | Impact |
|---|----------|--------|--------|
| 1 | Optimal snippet length for narrative quality | TBD — test during baseline | Too short = vague narratives; too long = context window pressure |
| 2 | Scaling beyond 4 controls per run | TBD — test if needed | May require batching strategy for production |
| 3 | Output stability across Claude model versions | TBD — test if model updates occur | May require SKILL.md adjustments |
| 4 | Synchrony's actual control count and codebase size | Unknown — deferred to handoff | Determines production scaling requirements |
| 5 | Confidence rubric calibration from real-world data | Deferred — future enhancement | Would make tiers more meaningful for Synchrony's context |
| 6 | 90-second per-call ceiling under multi-pass reasoning | TBD — monitor during testing | Risk flagged in Constitution Check |

---

*This document captures the research and design rationale for F5. It is a living document — update it as new patterns are tested or alternatives are evaluated.*
