# F1 — SKILL.md: Quickstart Guide

**What this is:** A step-by-step guide for running, testing, and improving the SKILL.md. No coding experience required.

---

## 1. What Does the SKILL.md Do?

The SKILL.md is an instruction file. You paste it into Claude along with some field metadata, and Claude writes data dictionary entries for you.

For each field you give it, Claude returns:
- A **description** of what the field means (25 words or less)
- A **confidence score** (High, Medium, or Low)
- **Evidence** explaining why it scored that way
- A **flag** telling you if a human needs to review it

That's it. You give it field info, it gives you descriptions with honesty about how sure it is.

---

## 2. How to Run It

### Step 1: Open Claude

Go to [claude.ai](https://claude.ai) or open Claude Desktop. Start a new conversation.

### Step 2: Paste the SKILL.md

Open the file at `skills/data-dictionary/SKILL.md`. Copy the entire thing. Paste it into the Claude conversation.

### Step 3: Paste Your Test Input

Copy the test input below and paste it right after the SKILL.md:

```json
{
  "table_name": "credit_card_clients",
  "fields": [
    {
      "field_name": "AGE",
      "type": "INTEGER",
      "nullable": false,
      "constraints": [],
      "enums": [],
      "schema_comments": "Age of the credit card holder in years"
    },
    {
      "field_name": "PAY_0",
      "type": "INTEGER",
      "nullable": true,
      "constraints": [],
      "enums": [],
      "schema_comments": ""
    },
    {
      "field_name": "X1",
      "type": "VARCHAR",
      "nullable": true,
      "constraints": [],
      "enums": [],
      "schema_comments": ""
    }
  ]
}
```

### Step 4: Hit Enter

Claude will process all 3 fields and return a JSON array with one object per field.

### Step 5: Check the Output

You should see something like this (exact wording may vary):

- **AGE** — High confidence, clear description, no flag
- **PAY_0** — Medium or Low confidence, hedged description, possibly flagged
- **X1** — Low confidence, vague description, flagged for review

If that's roughly what you see, the SKILL.md is working.

---

## 3. Why These 3 Test Fields?

Each one tests a different confidence level:

| Field | What It Tests | Expected Confidence |
|---|---|---|
| `AGE` | Clear name + helpful comment = lots of evidence | High |
| `PAY_0` | Ambiguous name + no comment = some evidence from name/type only | Medium or Low |
| `X1` | Meaningless name + no comment = almost no evidence | Low |

The SKILL.md has 3 worked examples inside it — one for each confidence level. These 3 test fields let you compare Claude's live output against those examples to make sure they match.

---

## 4. How to Read the Output

When you get the results back, read them in this order:

### First: Look for Flags

Scan for any field where `clarification_flag` is `true`. These are the fields Claude is saying "I'm not confident — a human should check this." Start your review here.

### Second: Check Confidence Levels

- **High** — Claude is confident. Probably fine, but worth a quick glance.
- **Medium** — Claude found some clues but isn't sure. Read the description and decide if it makes sense.
- **Low** — Claude is guessing. Definitely review this one.

### Third: Read the Evidence

Every field has an `evidence_refs` section that explains *why* Claude scored it that way. This is how you check Claude's work. If the evidence doesn't make sense, the description probably needs to be rewritten by a human.

### Fourth: Check Related Fields

If you see fields that look like a family (like `PAY_0`, `PAY_2`, `PAY_3`, `PAY_4`, `PAY_5`, `PAY_6`), make sure their descriptions are consistent. They should follow the same pattern since they're related. If one says "repayment status" and another says "payment amount," something is off.

---

## 5. Using Your Own Data

You don't have to use the test input above. You can test with any field metadata you want.

### What You Need (minimum)

The only thing you absolutely need is a table name and at least one field with a name:

```json
{
  "table_name": "my_table",
  "fields": [
    {
      "field_name": "my_field"
    }
  ]
}
```

That's the bare minimum. Claude will still produce all 5 output fields — but confidence will be Low because there's almost nothing to work with. That's correct behavior, not a bug.

### What Makes the Output Better

The more info you provide, the better the output gets:

| What You Provide | What Happens |
|---|---|
| Just `field_name` | Low confidence, generic description, flagged for review |
| `field_name` + `type` + `nullable` | Medium confidence, better description |
| `field_name` + `type` + `constraints` + `schema_comments` | High confidence, specific description with evidence |

Think of it like a scale — more metadata in, better descriptions out. It never breaks. It just gets less confident when it has less to work with.

---

## 6. When Something Looks Wrong

If the output doesn't look right, here's what to check — in this order:

### Step 0: Check Your Input

Before blaming the SKILL.md, make sure the input is correct:
- Is it valid JSON? (No missing commas, no trailing commas, brackets match)
- Does it have `table_name` at the top level?
- Does each field have at least `field_name`?

If the input is broken, the output will be broken. Fix the input first.

### Step 1: Fix the Worked Examples

This is the most common fix. Claude mirrors what it sees in the examples. If the output descriptions are too vague, make the example descriptions more specific. If confidence scores seem off, adjust the example scores.

The 3 worked examples in the SKILL.md are the single biggest lever you have. Change them first.

### Step 2: Adjust the Rubric

If confidence scores are consistently wrong (everything is High when it should be Medium, or everything is Low), the signal strength definitions might need tweaking. Look at the rubric section of the SKILL.md and adjust what counts as a "strong" vs. "weak" signal.

### Step 3: Rewrite the Instructions

This is the last resort. If the examples and rubric are fine but the output is still wrong, the written instructions in the SKILL.md might be unclear. Rewrite the specific section that's causing problems.

**Important:** Always try Steps 0–2 before Step 3. Most problems are input issues or example issues, not instruction issues.

---

## 7. Common Problems and Fixes

| Problem | Likely Cause | Fix |
|---|---|---|
| Output isn't valid JSON | Input wasn't valid JSON | Check your input for syntax errors — missing commas, extra commas, mismatched brackets |
| All fields show High confidence | Rubric is too loose | Tighten the signal strength definitions — make it harder to qualify as "strong" |
| All fields show Low confidence | Rubric is too strict, OR input has very little metadata | If input is sparse, that's correct behavior. If input is rich, loosen the rubric |
| Descriptions are too vague | Worked examples are too vague | Make the High confidence example description more specific and detailed |
| Descriptions are too long | 25-word limit isn't being enforced | Check that the constraint is clearly stated in the SKILL.md instructions |
| Related fields have inconsistent descriptions | Claude doesn't always catch field families | Review related fields manually — this is a known limitation |
| `evidence_refs` are generic ("based on field name") | Worked example evidence is generic | Make the example `evidence_refs` more specific — cite exact metadata values |
| Claude returns fewer objects than fields sent | Claude may have skipped fields | Re-run the same input. If it happens again, reduce the number of fields per batch |
| Claude adds extra fields to the output object | Instructions aren't strict enough about the 5-field contract | Add a line to the SKILL.md: "Return exactly these 5 fields per object. No additional fields." |
| Output looks completely different from examples | SKILL.md may not have been pasted correctly | Make sure you pasted the entire SKILL.md, not just part of it |
| Claude asks clarifying questions instead of producing output | Instructions need to say "do not ask questions" | Add to the SKILL.md: "Do not ask clarifying questions. Process every field with whatever metadata is available." |

---

## 8. Quick Reference

| Item | Location |
|---|---|
| SKILL.md (the instruction file) | `skills/data-dictionary/SKILL.md` |
| Input format details | `specs/f1-skill.md-data-dictionary/data-model.md` Section 2 |
| Output format details | `specs/f1-skill.md-data-dictionary/data-model.md` Section 3 |
| Confidence rubric details | `specs/f1-skill.md-data-dictionary/data-model.md` Section 4 |
| Input contract | `specs/f1-skill.md-data-dictionary/contracts/input-contract.md` |
| Output contract | `specs/f1-skill.md-data-dictionary/contracts/output-contract.md` |
| Research and design rationale | `specs/f1-skill.md-data-dictionary/research.md` |
| Full project plan | `specs/f1-skill.md-data-dictionary/plan.md` |

---

*This guide is for anyone on the team who needs to run, test, or improve the SKILL.md. If something isn't covered here, check data-model.md for the technical details or research.md for the design rationale.*
