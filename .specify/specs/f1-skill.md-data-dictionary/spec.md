# F1 — SKILL.md: Data Dictionary Generation

## Feature Specification

---

## 1. Feature Identity

| Field | Value |
|---|---|
| **Feature Name** | F1 — SKILL.md: Data Dictionary Generation |
| **Feature Branch** | `f1-skill.md-data-dictionary` |
| **Created** | 2026-04-02 |
| **Status** | Draft |
| **Owner** | Whitney Robinson (PM), Sheila Green, Molly Lowell |
| **Demo Day Deadline** | May 7, 2026 |
| **Input** | User provides a schema file (JSON, YAML, or DDL). F2 parses it and passes structured field metadata to the LLM. |

---

## 2. User Scenarios

> **Note:** All LLM-generated output is a **draft intended for human review**. The [NEEDS CLARIFICATION] flags and confidence scores are designed to direct the reviewer's attention to items most likely needing correction.

---

### US-1: Generate Descriptions for Clear Fields (P1)

**As a** documentation team member,
**I want** the LLM to generate plain-language descriptions for schema fields that have strong identifying signals,
**So that** I have a usable first draft of the data dictionary without manually writing every description.

**Priority:** P1 — Required for demo day.

**What this tests:** The LLM's ability to produce accurate, concise, cited descriptions when the input signals are clear and unambiguous.

**Acceptance Scenarios:**

| # | Given | When | Then |
|---|---|---|---|
| 1.1 | A field with a specific, unambiguous name (e.g., `AGE`) and a type that narrows meaning (e.g., `INTEGER`) | The LLM processes this field | Description is ≤25 words, plain language, accurate. Confidence = High. `evidence_refs` lists the signals used with brief reasoning. `clarification_flag` = false. |
| 1.2 | A field with a descriptive name and existing schema comments that agree with the name (e.g., `LIMIT_BAL` with comment "credit limit") | The LLM processes this field | Description incorporates both the name and the comment. Confidence = High (≥2 agreeing strong signals). `evidence_refs` cites both signals. |
| 1.3 | Multiple related fields in the same table (e.g., `BILL_AMT1` through `BILL_AMT6`) | The LLM processes all fields | Descriptions are consistent across the series — same phrasing pattern, same confidence level. Cross-field context is evident. |

---

### US-2: Handle Ambiguous / Opaque Fields (P1)

**As a** documentation team member,
**I want** the LLM to correctly identify fields it cannot confidently describe and flag them for my review,
**So that** I don't waste time reviewing fields the LLM got right and can focus on the ones that need human judgment.

**Priority:** P1 — Required for demo day.

**What this tests:** The LLM's ability to apply the confidence rubric honestly, flag uncertainty, and degrade gracefully when signals are weak or conflicting.

**Acceptance Scenarios:**

| # | Given | When | Then |
|---|---|---|---|
| 2.1 | A field with a vague or overloaded name (e.g., `PAY_0`) and no schema comments | The LLM processes this field | Description describes what the field *appears* to be based on available signals. Confidence = Medium (1 strong signal or signals with conflict). `evidence_refs` explains the uncertainty. |
| 2.2 | A field with cryptic numeric enums (e.g., `SEX` with values `1, 2`) | The LLM processes this field | Description notes the field contains coded values. `evidence_refs` notes that enum values are numeric-only (weak signal). Confidence adjusted accordingly. |
| 2.3 | A field with zero strong signals (e.g., hypothetical `col_x7`, type `VARCHAR`, no comments, no constraints, no enums) | The LLM processes this field | Confidence = Low. `clarification_flag` = true. Description states what little can be inferred and explicitly notes insufficient evidence. |
| 2.4 | A field where schema comments contradict the field name | The LLM processes this field | Conflict penalty applied — confidence drops one level. `evidence_refs` notes the specific conflict between signals. |
| 2.5 | A field with missing metadata (e.g., F2 provides no `type`) | The LLM processes this field | LLM notes the missing input in `evidence_refs`, lowers confidence by one level, and generates the best description possible from remaining signals. |

---

### US-3: Glossary-Assisted Description (P3)

**As a** documentation team member,
**I want** the LLM to use a provided glossary to map field names to standardized business labels,
**So that** descriptions align with organizational terminology.

**Priority:** P3 — Not required for demo day. Placeholder for future iteration.

**Minimal spec:** If a glossary is provided as an additional input alongside field metadata, the LLM maps field names to glossary labels and uses the standardized terms in its descriptions to improve accuracy and consistency. Detailed design deferred.

---

### Edge Cases (F1-Specific)

| Edge Case | Expected LLM Behavior |
|---|---|
| Field name is a single character or purely numeric (e.g., `X`, `7`) | Low confidence. Flag [NEEDS CLARIFICATION]. Describe only what type/constraints reveal. |
| All fields in a table are ambiguous | All fields get Medium or Low confidence. LLM does not artificially inflate scores to appear useful. |
| Schema comments trivially restate the field name (e.g., comment "Account ID" for field `acct_id`) | Comment does NOT count as an independent signal. Confidence scored as if comment were absent. |
| Field name contains dots or special characters (e.g., `default.payment.next.month`) | LLM treats the full string as the field name and interprets it as best it can. |

---

## 3. Worked Examples

The SKILL.md must include **3 worked examples** from the UCI Credit Card dataset to anchor LLM consistency:

- One **High confidence** field (e.g., `AGE`)
- One **Medium confidence** field (e.g., `PAY_0`)
- One **Low confidence** field (e.g., a hypothetical opaque field like `col_x7`)

Each example must show the full input → output cycle using the input and output contracts defined in this spec. Exact examples to be finalized during SKILL.md implementation.

---

## 4. Requirements

### 4.1 Functional Requirements

#### FR-007: Description Generation

- The LLM must generate a plain-language description for each field in the input.
- Each description must be **≤25 words**.
- Descriptions must be understandable by a **non-technical reader** — no jargon, no abbreviations unless they appear in the field name itself.
- Descriptions must be based **only** on the evidence provided in the input metadata.

#### FR-008: Citation / Evidence References

- Every description must include `evidence_refs` listing which input signals the LLM used.
- `evidence_refs` must include **brief reasoning** — not just what signals were used, but why they led to the given confidence score.
- Example: *"Field name is unambiguous ('AGE'), INTEGER type confirms numeric age value. No conflicting signals → High confidence."*
- The LLM must reference the `source_path` provided by F2 when available.

#### FR-009: Confidence Scoring

The LLM must assign a confidence score of **High**, **Medium**, or **Low** to every field.

**Thresholds:**

| Confidence | Criteria |
|---|---|
| **High** | ≥2 strong signals that agree (no conflicts) |
| **Medium** | 1 strong signal, OR 2+ signals with a conflict |
| **Low** | 0 strong signals → auto-flag [NEEDS CLARIFICATION] |

**Signal Strength Definitions:**

| Signal | Strong | Weak |
|---|---|---|
| **Field name** | Specific and unambiguous (e.g., `account_open_date`) | Generic/overloaded (e.g., `status`, `type`) or opaque (e.g., `col_x7`) |
| **Schema comments** | Present and descriptive | Absent or trivially restates the field name |
| **Data type** | Narrows meaning significantly (e.g., `DATE`, `BOOLEAN`) | Broad/ambiguous (e.g., `VARCHAR`, `INTEGER`) |
| **Constraints** | Semantically informative (e.g., `FOREIGN KEY references customers(id)`) | Only structural (e.g., `NOT NULL` alone) |
| **Enums** | Self-descriptive string values (e.g., `["ACTIVE", "CLOSED"]`) | Numeric-only or cryptic codes (e.g., `[1, 2, 3]`) |

**Additional Rules:**

- **Conflict penalty:** If any signal contradicts another, drop confidence by one level and note the conflict in `evidence_refs`.
- **Low = always flag:** Low confidence always triggers `clarification_flag` = true.
- **Comments that restate the name don't count:** If `schema_comments` just says "Account ID" for a field named `acct_id`, that's not an independent signal.

#### FR-011: Glossary Mapping (P3 — Deferred)

- If a glossary is provided as an additional input, the LLM maps field names to glossary labels and uses standardized terms in descriptions.
- Detailed design deferred to future iteration. Not required for demo day.

---

### 4.2 F1-Specific Requirements

#### FR-F1-001: Missing Input Handling

- If any expected input field is missing or null (e.g., F2 provides no `type` for a field), the LLM must:
  - Note the missing input in `evidence_refs`
  - Lower confidence by one level
  - Generate the best description possible from remaining signals

#### FR-F1-002: Cross-Field Consistency

- The LLM must consider all fields in the table when generating descriptions to ensure consistency across related fields (e.g., `PAY_0` through `PAY_6` should follow the same description pattern).

#### FR-F1-003: Field Name Echo

- The LLM must echo back `field_name` in every output object for matching by F2.

#### FR-F1-004: Generation Constraints

- The LLM must **only** describe what can be inferred from the provided metadata.
- The LLM must **NOT**:
  - Invent business logic that doesn't exist in the schema
  - Assume domain-specific meaning without evidence
  - State anything as fact that isn't supported by the input signals
- When uncertain, the LLM must describe what the field *appears* to be and flag [NEEDS CLARIFICATION].

---

### 4.3 Input Contract (F2 → F1)

The LLM receives **only reasoning inputs** — raw clues extracted by F2:

| Field | Description | Source |
|---|---|---|
| `field_name` | Column name in the database | F2 extracts from schema |
| `table_name` | Which table the field belongs to | F2 extracts from schema |
| `type` | Data type (DATE, INTEGER, VARCHAR, etc.) | F2 extracts from schema |
| `nullable` | Whether the field can be empty (true/false) | F2 extracts from schema |
| `constraints` | Database rules (PRIMARY KEY, FOREIGN KEY, UNIQUE) | F2 extracts from schema |
| `enums` | Fixed list of allowed values, if any | F2 extracts from schema |
| `source_path` | File path + location where field was found | F2 constructs |
| `schema_comments` | Human-written notes/descriptions from the schema (may be null) | F2 extracts from schema |

---

### 4.4 Output Contract (F1 → F2)

The LLM produces **only what it generated** — F2 handles assembly:

| Field | Description |
|---|---|
| `field_name` | Echo back for matching |
| `description` | Plain-language explanation (≤25 words) |
| `confidence` | High / Medium / Low |
| `evidence_refs` | What evidence the LLM used, with brief reasoning for the confidence score |
| `clarification_flag` | true/false — whether field needs human review |

**Pending decisions (deferred until F2 is built):**

- Output format (JSON array recommended — to be confirmed with F2 team)
- Validation rule: LLM must return exactly one output object per input field, in the same order received (to be confirmed with F2 team)

---

### 4.5 Key Entities

- **Field Metadata Object** — What the LLM receives (see Input Contract 4.3)
- **LLM Output Object** — What the LLM produces (see Output Contract 4.4)
- **Confidence Signal** — One of the 5 evidence types the LLM evaluates: field name, schema comments, data type, constraints, enums. Each signal is classified as Strong or Weak per the rubric in FR-009.

---

## 5. Pipeline Flow

```
F2 (extract) → structured metadata → F1 (LLM reasons) → LLM output → F2 (assemble) → F3 (template) → final data_dictionary.md
```

**F1's scope:** Receives structured metadata, applies reasoning, returns output. Everything before and after is F2/F3.

---

## 6. Boundaries — What F1 Does NOT Do

| Responsibility | Owned By |
|---|---|
| Parsing schema files | F2 |
| Extracting field metadata | F2 |
| Validating inputs before sending to LLM | F2 |
| Timestamping | F2 |
| Assembling final output files | F2 |
| Generating QA reports | F2 |
| Output templates | F3 |

---

## 7. Constitution Constraints Active on F1

- **Accuracy Over Speed** — quality of descriptions matters more than generation speed
- **Audit-Ready by Default** — every description must be traceable to input signals via `evidence_refs`
- **Never send raw PII to LLM** — only metadata (field names, types, constraints — not actual data values)
- **Graceful Degradation** — if LLM is offline, F2 scripts still extract raw metadata
- **Simplicity First** — file-based, no UI

---

## 8. Processing Approach

- **All at once:** F2 sends the entire array of field metadata to the LLM in a single pass.
- The LLM sees all fields together, enabling cross-field consistency (FR-F1-002).
- **F2 may batch** if schemas exceed context window limits (500+ fields). Batching logic is F2's responsibility — the SKILL.md instructions do not change.

---

## 9. Success Criteria

**Approach:** Measurement categories defined now. Specific thresholds TBD after baseline testing on the UCI Credit Card dataset.

| SC ID | What We Measure | Threshold | How We Measure |
|---|---|---|---|
| SC-F1-001 | **Completeness** — LLM produces all 5 output fields for every input field | TBD | Automated check |
| SC-F1-002 | **Description Quality** — Descriptions are understandable, accurate, ≤25 words | TBD | Team scores manually |
| SC-F1-003 | **Confidence Calibration** — High/Medium/Low scores match actual signal strength | TBD | Team reviews sample |
| SC-F1-004 | **Citation Coverage** — Every description includes evidence_refs with reasoning | TBD | Automated count |
| SC-F1-005 | **Clarification Flag Accuracy** — All Low confidence fields flagged [NEEDS CLARIFICATION] | TBD | Automated check |
| SC-F1-006 | **Consistency** — Same input produces structurally similar output across runs | TBD | Run skill 3x, compare |

**Baseline Testing Plan:**

1. Build the SKILL.md
2. Run it on the UCI Credit Card dataset (25 fields)
3. Team manually scores the output across all 6 categories
4. Set thresholds based on actual performance
5. Document final thresholds before demo day

---

## 10. Test Data

- **Dataset:** UCI Default of Credit Card Clients (Kaggle)
- **File:** `UCI_Credit_Card.csv`
- **Fields:** 25 columns
- **Why this dataset:** Mix of clear fields (`AGE`, `LIMIT_BAL`), ambiguous fields (`PAY_0`), cryptic numeric enums (`SEX`: 1/2), numbered series (`BILL_AMT1`–`BILL_AMT6`), and an oddly formatted target variable (`default.payment.next.month`)
- **Note:** For demo day, this CSV will need to be converted into a schema file (JSON, YAML, or DDL) — that's a prep step for F2
- **Limitation:** This dataset is a proof-of-concept test case. Production schemas (e.g., Synchrony) may be more complex with nested structures, foreign keys across tables, and proprietary naming conventions.

---

## 11. Approved Assumptions

| # | Assumption | Mitigation |
|---|---|---|
| 1 | LLM receives well-structured JSON from F2 | FR-F1-001: Missing input handling rules — lower confidence, note in evidence_refs |
| 2 | ≤25 words is always sufficient for a description | Hard ceiling per FR-007. Extra context goes in evidence_refs. |
| 3 | Confidence rubric produces consistent results | Worked examples from UCI dataset serve as anchors in SKILL.md |
| 4 | LLM doesn't need structured neighboring_fields input | LLM sees all fields at once. FR-F1-002 instructs cross-field consistency. |
| 5 | UCI dataset is representative enough for demo day | Acceptable for proof of concept. Limitation noted in Section 10. |

---

## 12. Open Items

| # | Item | Status |
|---|---|---|
| 1 | LLM output format (JSON array recommended) | ⏳ PENDING — confirm with F2 team |
| 2 | Count validation (one output per input field) | ⏳ PENDING — confirm with F2 team |
| 3 | Success criteria thresholds | ⏳ TBD — after baseline testing |
| 4 | Worked example exact content | ⏳ TBD — during SKILL.md implementation |
