# F1 — SKILL.md: Data Dictionary Generation — Data Model

**Feature Branch**: `f1-skill.md-data-dictionary`
**Created**: 2026-04-09
**Status* Draft
**Owner**: Whitney Robinson (PM), Sheila Green, Molly Lowell

---

## 1. What This Document Covers

This defines the exact shape of data going into and out of the SKILL.md:

- **What F2 sends to the SKILL.md** (input)
- **What the SKILL.md sends back** (output)
- **How confidence scoring works** (the rubric)
- **How F2 checks the output** (validation rules)

---

## 2. What F2 Sends to the SKILL.md

F2 sends a JSON object with the table name and a list of fields.

### Structure

```json
{
  "table_name": "credit_card_clients",
  "source_file": "sample_schema.json",
  "fields": [
    {
      "field_name": "AGE",
      "type": "INTEGER",
      "nullable": false,
      "constraints": [],
      "enums": [],
      "source_path": "schemas/credit.json#/fields/AGE",
      "schema_comments": null
    }
  ]
}
```

### Field Descriptions

| Field | What It Is | Required? |
|---|---|---|
| `table_name` | The name of the table these fields belong to. Claude needs this to understand context and keep related fields consistent. | **Yes** |
| `source_file` | The name of the original data source file (e.g., `"sample_schema.json"`). Used to build `source_path` for each field. | **Yes** |
| `field_name` | The column name exactly as it appears in the schema. Case-sensitive — `PAY_0` stays `PAY_0`. | **Yes** |
| `type` | The data type — `INTEGER`, `VARCHAR`, `DATE`, `BOOLEAN`, etc. | No |
| `nullable` | Can this field be empty? `true` or `false`. | No |
| `constraints` | Database rules like `PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `CHECK`. Empty list `[]` if none. | No |
| `enums` | A fixed list of allowed values, if any. Can be strings or numbers. Empty list `[]` if none. | No |
| `source_path` | Where F2 found this field in the schema file. | No |
| `schema_comments` | Any human-written notes from the schema. `null` if there aren't any. | No |

**The rule:** `table_name`, `source_file`, and `field_name` are the only required fields. Everything else can be missing or null. When something is missing, Claude notes it and lowers confidence — it doesn't break.

**Validation rule:** `field_name` must be a non-empty string. If `field_name` is an empty string, whitespace-only, or null, F2 must reject the field before sending it to the SKILL.md. This is a pre-LLM validation check owned by F2 (`validate_input.py`).

**Extra fields:** If the input contains fields beyond those listed above, the SKILL.md must ignore them. `evidence_refs` must only cite the contracted input signals (`field_name`, `type`, `nullable`, `constraints`, `enums`, `source_path`, `schema_comments`). Claude must not reference any additional fields in its output, even if they are present in the input.

### Full Example (3 fields)

```json
{
  "table_name": "credit_card_clients",
  "source_file": "sample_schema.json",
  "fields": [
    {
      "field_name": "AGE",
      "type": "INTEGER",
      "nullable": false,
      "constraints": [],
      "enums": [],
      "source_path": "schemas/credit.json#/fields/AGE",
      "schema_comments": null
    },
    {
      "field_name": "PAY_0",
      "type": "INTEGER",
      "nullable": true,
      "constraints": [],
      "enums": [-2, -1, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9],
      "source_path": "schemas/credit.json#/fields/PAY_0",
      "schema_comments": "Repayment status in September 2005"
    },
    {
      "field_name": "col_x7",
      "type": "VARCHAR",
      "nullable": true,
      "constraints": [],
      "enums": [],
      "source_path": "schemas/credit.json#/fields/col_x7",
      "schema_comments": null
    }
  ]
}
```

---

## 3. What the SKILL.md Sends Back

The SKILL.md returns a JSON list — one object per field, in the same order it received them. Every object has exactly 5 fields. Always 5. No exceptions.

### The 5 Output Fields

| Field | What It Is | Allowed Values |
|---|---|---|
| `field_name` | The field name, echoed back exactly as received. No renaming, no reformatting. | Must match the input exactly |
| `description` | A plain-language explanation of the field. No jargon. ≤25 words. | Any clear, simple text |
| `confidence` | How confident Claude is, based on the rubric (see Section 4). | `"High"`, `"Medium"`, or `"Low"` |
| `evidence_refs` | The specific clues Claude used and why they led to that confidence score. | A list with at least one entry |
| `clarification_flag` | Does this field need a human to look at it? | `true` or `false` |

**Word count rule:** Words are counted by splitting on whitespace. Hyphenated terms count as one word. Contractions count as one word. Technical terms with underscores (e.g., `source_path`) count as one word. F2 uses the same counting method when checking VR-05.

### Example: High Confidence

```json
{
  "field_name": "AGE",
  "description": "The age of the credit card holder, in years.",
  "confidence": "High",
  "evidence_refs": [
    "field_name: 'AGE' is specific and unambiguous — universally understood to mean a person's age",
    "type: INTEGER confirms numeric value consistent with age in years",
    "nullable: false — age is required, consistent with a core demographic field"
  ],
  "clarification_flag": false
}
```

### Example: Medium Confidence

```json
{
  "field_name": "PAY_0",
  "description": "Repayment status for the most recent billing period, represented as a coded integer value.",
  "confidence": "Medium",
  "evidence_refs": [
    "field_name: 'PAY_0' suggests payment-related, but '_0' suffix is ambiguous",
    "schema_comment: 'Repayment status in September 2005' confirms payment context but references a specific month",
    "enums: numeric codes [-2 through 9] — not self-descriptive, meaning of each code is unclear",
    "Conflict: field name implies generic index ('_0') while comment specifies a calendar month. Confidence limited to Medium."
  ],
  "clarification_flag": false
}
```

### Example: Low Confidence

```json
{
  "field_name": "col_x7",
  "description": "Purpose unknown. Field name is non-descriptive and no metadata provides meaningful context.",
  "confidence": "Low",
  "evidence_refs": [
    "field_name: 'col_x7' is opaque — no semantic meaning can be inferred",
    "type: VARCHAR is broad and does not narrow the field's purpose",
    "schema_comments: absent",
    "constraints: none",
    "enums: none",
    "Insufficient evidence to generate a meaningful description."
  ],
  "clarification_flag": true
}
```

### When Something Goes Wrong

If Claude can't process a field (bad input, missing required fields), it still returns all 5 fields. It doesn't skip the field or return an error code.

```json
{
  "field_name": "UNKNOWN",
  "description": "ERROR: Missing required field identifier.",
  "confidence": "Low",
  "evidence_refs": [
    "field_name: not provided — cannot identify this field"
  ],
  "clarification_flag": true
}
```

This way F2 always gets the same shape back. No special error handling needed.

---

## 4. How Confidence Scoring Works

### The Three Levels

| Confidence | What It Means | When It Happens |
|---|---|---|
| `"High"` | Claude found 2 or more strong clues that agree with each other | Clear field name + helpful comment, or clear name + informative type, etc. |
| `"Medium"` | Claude found 1 strong clue, OR found multiple clues that conflict | Decent field name but nothing else to confirm it, or name says one thing and comment says another |
| `"Low"` | Claude found 0 strong clues | Opaque field name, no comments, generic type, no constraints, no enums |

### What Counts as a Strong vs. Weak Clue

| Clue | Strong | Weak |
|---|---|---|
| **Field name** | Specific and clear (e.g., `account_open_date`, `AGE`) | Vague (e.g., `status`, `type`) or meaningless (e.g., `col_x7`, `X`) |
| **Schema comments** | Descriptive and adds new information | Missing, or just restates the field name (e.g., comment "Account ID" for field `acct_id`) |
| **Data type** | Narrows the meaning (e.g., `DATE`, `BOOLEAN`) | Could be anything (e.g., `VARCHAR`, `INTEGER`) |
| **Constraints** | Tells you something meaningful (e.g., `FOREIGN KEY references customers(id)`) | Just structural (e.g., `NOT NULL` by itself) |
| **Enums** | Readable values (e.g., `["ACTIVE", "CLOSED"]`) | Just numbers or codes (e.g., `[1, 2, 3]`) |

### Scoring Rules

1. **Count the strong clues.** 0 = `"Low"`. 1 = `"Medium"`. 2+ agreeing = `"High"`.

2. **Conflict penalty.** If any clue contradicts another, drop confidence by one level. A field with 2 strong clues that conflict = `"Medium"` (not `"High"`). Note the conflict in `evidence_refs`.

3. **Restated comments don't count.** If the schema comment just repeats the field name in different words, it's not a separate clue. Score as if the comment weren't there.

4. **Low always gets flagged.** If confidence is `"Low"`, `clarification_flag` must be `true`. Always.

5. **Missing input lowers confidence.** If F2 didn't provide a field (e.g., no `type`), note it in `evidence_refs` and drop confidence by one level.

### Edge Cases

| Situation | What Happens |
|---|---|
| 2 strong clues that agree, no conflicts | `"High"` |
| 2 strong clues that conflict | `"Medium"` (dropped from `"High"`) |
| 1 strong clue, no conflicts | `"Medium"` |
| 1 strong clue + 1 conflict | `"Low"` (dropped from `"Medium"`) |
| 0 strong clues | `"Low"`, flag = `true` |
| 3+ strong clues, 1 conflict | `"Medium"` (dropped from `"High"`) |
| Field name is a single character (e.g., `X`) | Field name = Weak. Probably `"Low"` unless other clues are strong. |
| Every field in the table is ambiguous | Every field gets `"Medium"` or `"Low"`. Claude doesn't inflate scores to look useful. |
| Comment just restates the field name | Doesn't count. Score as if comment were missing. |
| Field name has dots (e.g., `default.payment.next.month`) | Claude treats the full string as the name and interprets it as best it can. |
| `type` is missing from input | Note it, drop confidence one level, describe from remaining clues. |
| `type` AND `schema_comments` both missing | Drop confidence for each. Floor is `"Low"`. |
| `field_name` is empty string or whitespace-only | F2 rejects the field before it reaches the SKILL.md. Not sent to Claude. |
| Field name contains special characters (e.g., `@`, `#`, newlines) | Claude echoes the `field_name` exactly as received. If special characters would break JSON output formatting, F2's VR-01 catches the malformed JSON and retries. |
| Field name is extremely long (100+ characters) | Claude echoes it back and writes a description as normal. No length limit is enforced. If combined input approaches context window limits, F2's batching logic applies. |
| Enums list is very large (100+ values) | Claude reads all values but may summarize rather than list each one in `evidence_refs`. Large enum lists consume context window space — F2 should account for this in token estimation. |
| Duplicate field names in the input array | Claude processes each occurrence independently and returns one output object per input object. Descriptions may differ if surrounding context differs. F2's VR-13 flags duplicates in the output. The SKILL.md does not deduplicate — that is F2's responsibility. |

---

## 5. How F2 Checks the Output

When F2 gets the JSON back from Claude, it runs these checks before accepting it.

### Must-Pass Checks (Reject and Retry if Failed)

| # | What F2 Checks | Why |
|---|---|---|
| VR-01 | Is the output valid JSON? No extra text, no markdown formatting around it. | If Claude added commentary outside the JSON, the whole response is unusable. |
| VR-02 | Does the output have the same number of fields as the input? | Claude sometimes skips fields in the middle of long lists. |
| VR-03 | Does every object have all 5 fields? | F2 needs a predictable structure. |
| VR-06 | Is `confidence` one of exactly `"High"`, `"Medium"`, or `"Low"`? | Anything else breaks downstream logic. |
| VR-07 | Is `evidence_refs` a list with at least one entry? | Empty evidence means no audit trail. |
| VR-08 | Is `clarification_flag` a true/false value? | Anything else breaks downstream logic. |

**If any of these fail:** F2 rejects the entire batch and retries. F2 owns retry logic — maximum 2 retries (3 total attempts). If it still fails after 3 attempts, F2 stops and alerts the team.

### Flag-and-Continue Checks (Log It, Keep Going)

| # | What F2 Checks | Why |
|---|---|---|
| VR-04 | Does each `field_name` in the output match the input exactly? | Claude sometimes "corrects" casing or formatting. |
| VR-05 | Is each `description` ≤25 words? | Occasionally Claude goes slightly over. |
| VR-09 | If confidence is `"Low"`, is `clarification_flag` true? | The rubric requires this. If Claude missed it, F2 can auto-correct. |
| VR-12 | Are the output fields in the same order as the input? | Nice to have. F2 can reorder by matching `field_name` if needed. |
| VR-13 | Are there any duplicate `field_name` values? | Shouldn't happen, but worth catching. |
| VR-14 | Does any `evidence_ref` cite a signal that was null or absent in the input? (e.g., references "schema_comments" when `schema_comments` was null) | Catches hallucinated evidence automatically instead of relying solely on human spot-checks. |

**If any of these fail:** F2 logs the issue and flags it for manual review, but doesn't reject the batch.

### Spot-Check by Humans (During Triage)

| # | What the Reviewer Checks | Why |
|---|---|---|
| VR-10 | Do the `evidence_refs` actually reference clues from the input? | Makes sure Claude cited real signals, not made-up ones. |
| VR-11 | Did Claude cite a signal that wasn't in the input? (e.g., referencing a comment when there was none) | Catches hallucinated evidence. |

VR-14 provides automated first-pass detection of hallucinated evidence. VR-10 and VR-11 remain as human verification for subtler issues (e.g., Claude citing a signal correctly but misinterpreting its meaning).

---

## 6. How It All Fits Together

```
F2 builds the input JSON (Section 2)
        │
        ▼
SKILL.md receives it, processes each field:
  → Evaluates the 5 clue types (strong or weak?)
  → Applies the confidence rubric (Section 4)
  → Writes a description (≤25 words)
  → Lists the evidence in evidence_refs
  → Sets the clarification flag
        │
        ▼
SKILL.md returns the output JSON (Section 3)
        │
        ▼
F2 validates the output (Section 5)
        │
        ▼
F2 passes validated output to F3 for formatting
```

---

*This document is the reference for F1's data structures and validation rules. Update it if the input/output shapes or the rubric change.*
