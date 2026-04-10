# Input Contract: F2 → F1 (SKILL.md)

**What this is:** The promise F2 makes to F1. When F2 sends data to the SKILL.md, it will always look like this.

**Full details:** See `data-model.md`, Section 2.

---

## What F2 Sends

A JSON object with the table name, source file, and a list of fields.

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

---

## Fields

| Field | Required? | What It Is |
|---|---|---|
| `table_name` | **Yes** | Name of the table |
| `source_file` | **Yes** | Name of the original data source file (e.g., `"sample_schema.json"`) |
| `field_name` | **Yes** | Column name, exactly as it appears in the schema |
| `type` | No | Data type (INTEGER, VARCHAR, DATE, etc.) |
| `nullable` | No | Can this field be empty? true/false |
| `constraints` | No | Database rules (PRIMARY KEY, FOREIGN KEY, CHECK, etc.). Empty list if none. |
| `enums` | No | Allowed values, if any. Empty list if none. |
| `source_path` | No | Where F2 found this field in the schema file |
| `schema_comments` | No | Human-written notes from the schema. null if none. |

---

## F2's Promises

1. **`table_name`, `source_file`, and `field_name` are always present.** Everything else can be missing or null.
2. **`field_name` is sent exactly as it appears in the schema.** No cleaning, no reformatting.
3. **`source_file` is a non-empty string.** It identifies the original data source file and is used to build `source_path` for each field.
4. **The `fields` list has at least one field.** F2 never sends an empty list.
5. **The JSON is valid.** F2 validates the structure before sending.
6. **No raw data is included.** Only metadata — never actual row values. (Constitution: Never-Ever Rules)

---

## What Happens When Input Is Incomplete

If optional fields are missing, the SKILL.md still works. It just has less to go on:

| Missing Field | Effect on Output |
|---|---|
| `type` missing | Confidence drops one level. Claude notes it in evidence_refs. |
| `schema_comments` missing | One fewer signal to evaluate. Confidence may drop. |
| `constraints` missing | One fewer signal. No effect if other signals are strong. |
| `enums` missing | One fewer signal. No effect if other signals are strong. |
| `source_path` missing | No effect on description or confidence. F2 just can't link back to the source. |
| Multiple fields missing | Confidence drops for each. Floor is `"Low"`. |

The SKILL.md never fails on missing optional fields. It degrades gracefully.
