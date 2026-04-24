# Data Dictionary — credit_card_clients

**Source File:** sample_schema.json  
**Generated:** 2026-04-23T02:56:30Z

---

## Field Summary

| field_name | type | nullable | constraints | enums | description | confidence | last_verified |
|---|---|---|---|---|---|---|---|
| ID | INTEGER | false | PRIMARY KEY | - | Unique identifier for each client record. | High | 2026-04-23T02:56:30Z |
| LIMIT_BAL | DECIMAL | false | CHECK (LIMIT_BAL > 0) | - | Credit limit assigned to the client, measured in NT dollars. | High | 2026-04-23T02:56:30Z |
| SEX | INTEGER | false | - | 1; 2 | Coded value representing the client's sex. | Medium | 2026-04-23T02:56:30Z |
| EDUCATION | INTEGER | true | - | 0; 1; 2; 3; 4; 5; 6 | Coded value for the client's education level. | Medium | 2026-04-23T02:56:30Z |
| MARRIAGE | INTEGER | true | - | 0; 1; 2; 3 | [NEEDS CLARIFICATION] Coded value representing the client's marital status. | Low | 2026-04-23T02:56:30Z |
| AGE | INTEGER | false | CHECK (AGE >= 18) | - | Age of the client in years. | High | 2026-04-23T02:56:30Z |
| PAY_0 | UNKNOWN | true | - | -2; -1; 0; 1; 2; 3; 4; 5; 6; 7; 8; 9 | [NEEDS CLARIFICATION] Repayment status code for the most recent billing period. | Low | 2026-04-23T02:56:30Z |
| PAY_2 | INTEGER | true | - | - | [NEEDS CLARIFICATION] Repayment status for a prior billing period, stored as a code. | Low | 2026-04-23T02:56:30Z |
| PAY_3 | INTEGER | true | - | -2; -1; 0; 1; 2; 3; 4; 5; 6; 7; 8; 9 | Repayment status in July 2005, represented as a coded integer. | Medium | 2026-04-23T02:56:30Z |
| PAY_4 | INTEGER | true | - | -2; -1; 0; 1; 2; 3; 4; 5; 6; 7; 8; 9 | Repayment status in June 2005, represented as a coded integer. | Medium | 2026-04-23T02:56:30Z |
| PAY_5 | INTEGER | true | - | -2; -1; 0; 1; 2; 3; 4; 5; 6; 7; 8; 9 | Repayment status in May 2005, represented as a coded integer. | Medium | 2026-04-23T02:56:30Z |
| PAY_6 | INTEGER | true | - | -2; -1; 0; 1; 2; 3; 4; 5; 6; 7; 8; 9 | Repayment status in April 2005, represented as a coded integer. | Medium | 2026-04-23T02:56:30Z |
| BILL_AMT1 | DECIMAL | true | - | - | Bill statement amount recorded for September 2005. | High | 2026-04-23T02:56:30Z |
| BILL_AMT2 | DECIMAL | true | - | - | Bill statement amount recorded for August 2005. | High | 2026-04-23T02:56:30Z |
| BILL_AMT3 | DECIMAL | true | - | - | Bill statement amount recorded for July 2005. | High | 2026-04-23T02:56:30Z |
| BILL_AMT4 | DECIMAL | true | - | - | Bill statement amount recorded for June 2005. | High | 2026-04-23T02:56:30Z |
| BILL_AMT5 | DECIMAL | true | - | - | Bill statement amount recorded for May 2005. | High | 2026-04-23T02:56:30Z |
| BILL_AMT6 | DECIMAL | true | - | - | Bill statement amount recorded for April 2005. | High | 2026-04-23T02:56:30Z |
| PAY_AMT1 | DECIMAL | true | - | - | Payment amount made in September 2005. | High | 2026-04-23T02:56:30Z |
| PAY_AMT2 | DECIMAL | true | - | - | Payment amount made in August 2005. | High | 2026-04-23T02:56:30Z |
| PAY_AMT3 | DECIMAL | true | - | - | Payment amount made in July 2005. | High | 2026-04-23T02:56:30Z |
| PAY_AMT4 | DECIMAL | true | - | - | Payment amount made in June 2005. | High | 2026-04-23T02:56:30Z |
| PAY_AMT5 | DECIMAL | true | - | - | Payment amount made in May 2005. | High | 2026-04-23T02:56:30Z |
| PAY_AMT6 | DECIMAL | true | - | - | Payment amount made in April 2005. | High | 2026-04-23T02:56:30Z |
| default.payment.next.month | INTEGER | false | - | 0; 1 | Indicator of whether the client defaults on payment next month. | High | 2026-04-23T02:56:30Z |

## Evidence & Citations

### ID
1. field_name: 'ID' is a standard identifier name.
2. constraints: PRIMARY KEY indicates row uniqueness.
3. schema_comments: explicitly states unique client identifier.
4. type: INTEGER supports identifier storage.

### LIMIT_BAL
1. field_name: 'LIMIT_BAL' clearly indicates balance limit context.
2. schema_comments: states credit limit in NT dollars.
3. constraints: CHECK (LIMIT_BAL > 0) supports monetary limit meaning.
4. type: DECIMAL is appropriate for currency values.

### SEX
1. field_name: 'SEX' suggests demographic sex category.
2. schema_comments: confirms this is a sex code.
3. enums: values [1, 2] indicate coded categories without labels.
4. type: INTEGER supports encoded category storage.

### EDUCATION
1. field_name: 'EDUCATION' is semantically specific.
2. schema_comments: confirms education level code.
3. enums: values [0-6] are numeric and unlabeled, reducing interpretability.
4. type: INTEGER supports coded category values.

### MARRIAGE
1. field_name: 'MARRIAGE' suggests marital-status context.
2. schema_comments: missing, so no textual confirmation.
3. enums: values [0, 1, 2, 3] are numeric and unlabeled.
4. type: INTEGER supports coded categories but not specific labels.

### AGE
1. field_name: 'AGE' is specific and unambiguous.
2. schema_comments: explicitly states age in years.
3. constraints: CHECK (AGE >= 18) confirms age semantics.
4. type: INTEGER supports whole-year age values.

### PAY_0
1. field_name: 'PAY_0' implies payment-status series but index meaning is ambiguous.
2. schema_comments: missing, so period context is not explicitly confirmed.
3. enums: values -2 through 9 indicate coded status levels without labels.
4. type: UNKNOWN provides no type-level support.

### PAY_2
1. field_name: 'PAY_2' suggests repayment-status sequence but period mapping is unclear.
2. schema_comments: missing.
3. enums: missing, so code meanings are not provided.
4. type: INTEGER supports coded status storage but not semantics.

### PAY_3
1. field_name: 'PAY_3' aligns with repayment-status series naming.
2. schema_comments: explicitly states repayment status in July 2005.
3. enums: values -2 through 9 are numeric and unlabeled codes.
4. type: INTEGER supports coded status representation.

### PAY_4
1. field_name: 'PAY_4' indicates repayment-status series.
2. schema_comments: explicitly states repayment status in June 2005.
3. enums: values -2 through 9 are numeric and unlabeled codes.
4. type: INTEGER supports coded status values.

### PAY_5
1. field_name: 'PAY_5' indicates repayment-status series.
2. schema_comments: explicitly states repayment status in May 2005.
3. enums: values -2 through 9 are numeric and unlabeled codes.
4. type: INTEGER supports coded status storage.

### PAY_6
1. field_name: 'PAY_6' indicates repayment-status series.
2. schema_comments: explicitly states repayment status in April 2005.
3. enums: values -2 through 9 are numeric and unlabeled codes.
4. type: INTEGER supports coded status representation.

### BILL_AMT1
1. field_name: 'BILL_AMT1' indicates bill amount series.
2. schema_comments: explicitly states bill statement amount in September 2005.
3. type: DECIMAL supports monetary amounts.

### BILL_AMT2
1. field_name: 'BILL_AMT2' indicates bill amount series.
2. schema_comments: explicitly states bill statement amount in August 2005.
3. type: DECIMAL supports monetary values.

### BILL_AMT3
1. field_name: 'BILL_AMT3' indicates bill amount series.
2. schema_comments: explicitly states bill statement amount in July 2005.
3. type: DECIMAL supports currency amounts.

### BILL_AMT4
1. field_name: 'BILL_AMT4' indicates bill amount series.
2. schema_comments: explicitly states bill statement amount in June 2005.
3. type: DECIMAL supports monetary amounts.

### BILL_AMT5
1. field_name: 'BILL_AMT5' indicates bill amount series.
2. schema_comments: explicitly states bill statement amount in May 2005.
3. type: DECIMAL supports currency amounts.

### BILL_AMT6
1. field_name: 'BILL_AMT6' indicates bill amount series.
2. schema_comments: explicitly states bill statement amount in April 2005.
3. type: DECIMAL supports monetary values.

### PAY_AMT1
1. field_name: 'PAY_AMT1' indicates payment amount series.
2. schema_comments: explicitly states amount paid in September 2005.
3. type: DECIMAL supports monetary amounts.

### PAY_AMT2
1. field_name: 'PAY_AMT2' indicates payment amount series.
2. schema_comments: explicitly states amount paid in August 2005.
3. type: DECIMAL supports currency values.

### PAY_AMT3
1. field_name: 'PAY_AMT3' indicates payment amount series.
2. schema_comments: explicitly states amount paid in July 2005.
3. type: DECIMAL supports monetary amounts.

### PAY_AMT4
1. field_name: 'PAY_AMT4' indicates payment amount series.
2. schema_comments: explicitly states amount paid in June 2005.
3. type: DECIMAL supports currency amounts.

### PAY_AMT5
1. field_name: 'PAY_AMT5' indicates payment amount series.
2. schema_comments: explicitly states amount paid in May 2005.
3. type: DECIMAL supports monetary values.

### PAY_AMT6
1. field_name: 'PAY_AMT6' indicates payment amount series.
2. schema_comments: explicitly states amount paid in April 2005.
3. type: DECIMAL supports monetary values.

### default.payment.next.month
1. field_name: dotted name explicitly indicates default payment next month.
2. schema_comments: confirms default payment indicator context.
3. enums: values [0, 1] support binary indicator semantics.
4. type: INTEGER supports binary encoded values.

## Confidence Legend

- **High**: Multiple strong signals agree on field meaning.
- **Medium**: Some signals are present, but ambiguity remains.
- **Low**: Limited or conflicting signals; human clarification is required.

## Generation Notes

- Descriptions are AI-generated drafts for human review.
- Fields marked `[NEEDS CLARIFICATION]` require human attention before production use.

- **Medium**: Some signals are present, but ambiguity remains.
- **Low**: Limited or conflicting signals; human clarification is required.

## Generation Notes

- Descriptions are AI-generated drafts for human review.
- Fields marked `[NEEDS CLARIFICATION]` require human attention before production use.
