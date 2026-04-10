# Output Contract: F1 (SKILL.md) → F2

**What this is:** The promise F1 makes to F2. When the SKILL.md returns data, it will always look like this.

**Full details:** See `data-model.md`, Section 3.

---

## What the SKILL.md Returns

A JSON list — one object per field, in the same order they were received. Every object has exactly 5 fields.

```json
[
  {
    "field_name": "AGE",
    "description": "The age of the credit card holder, in years.",
    "confidence": "High",
    "evidence_refs": [
      "field_name: 'AGE' is specific and unambiguous",
      "type: INTEGER confirms numeric value consistent with age"
    ],
    "clarification_flag": false
  }
]
```

---

## The 5 Output Fields

| Field | What It Is | Allowed Values |
|---|---|---|
| `field_name` | The field name, echoed back exactly as received. No renaming. | Must match the input exactly |
| `description` | A plain-language explanation of the field. ≤25 words. | Any clear, simple text |
| `confidence` | How confident Claude is in the description. | `"High"`, `"Medium"`, or `"Low"` only |
| `evidence_refs` | The specific clues Claude used and why. | A list with at least one entry |
| `clarification_flag` | Does this field need a human to review it? | `true` or `false` |

---

## F1's Promises

1. **Every input field gets an output object.** If F2 sends 20 fields, F1 returns 20 objects. No skipping.
2. **Every object has all 5 fields.** Always. No optional fields in the output.
3. **`field_name` is returned exactly as received.** No case changes, no reformatting, no "corrections."
4. **`confidence` is always one of three values.** `"High"`, `"Medium"`, or `"Low"`. Nothing else.
5. **Low confidence always gets flagged.** If `confidence` is `"Low"`, `clarification_flag` is always `true`.
6. **`evidence_refs` always cites real input signals.** Claude never invents metadata that wasn't in the input.
7. **Output order matches input order.** Fields come back in the same sequence they were sent.

---

## What Happens When Something Goes Wrong

If Claude can't process a field, it still returns all 5 fields. It doesn't skip the field or return a different format.

```json
{
  "field_name": "SOME_FIELD",
  "description": "Unable to process. Input metadata may be malformed.",
  "confidence": "Low",
  "evidence_refs": [
    "Processing error: could not interpret input metadata for this field"
  ],
  "clarification_flag": true
}
```

This way F2 always gets the same shape back. No special error handling needed — problem fields just show up as `"Low"` confidence with a flag.

---

## How F2 Validates the Output

F2 owns retry logic. F2 runs checks before accepting the output. Two categories:

### Reject and Retry (max 2 retries, 3 total attempts)

| Check | What Could Go Wrong |
|---|---|
| Is it valid JSON? | Claude added commentary or markdown outside the JSON |
| Same number of objects as input fields? | Claude skipped fields in the middle of a long list |
| Every object has all 5 fields? | Claude dropped a field from some objects |
| `confidence` is `"High"`, `"Medium"`, or `"Low"`? | Claude used a different word |
| `evidence_refs` is a non-empty list? | Claude left it empty |
| `clarification_flag` is true or false? | Claude used a string instead of a boolean |

### Flag and Continue (log it, keep going)

| Check | What Could Go Wrong |
|---|---|
| `field_name` matches input exactly? | Claude "corrected" the casing |
| `description` is ≤25 words? | Claude went slightly over on a complex field |
| `"Low"` confidence → flag is true? | Claude forgot to set the flag (F2 can auto-correct this) |
| Output is in the same order as input? | Claude reordered (F2 can match by field_name) |
| No duplicate field names? | Claude repeated a field |

### If Retries Fail

If the output fails the reject-and-retry checks 3 times in a row, F2 stops and alerts the team. Most likely cause: the batch is too large for Claude's context window. Fix: reduce the number of fields per request.

---

*This contract is the quick reference for what F1 returns. For full examples, edge cases, and the confidence rubric, see `data-model.md`.*
