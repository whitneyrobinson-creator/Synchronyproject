# Data Dictionary — credit_card_clients

**Source File:** sample_schema.json  
**Generated:** 2026-04-15T23:19:49Z

---

## Field Summary

| field_name | type | nullable | constraints | enums | description | confidence | last_verified |
|---|---|---|---|---|---|---|---|
| ID | INTEGER | false | PRIMARY KEY | - | Unique identifier for each client record. | High | 2026-04-15T23:19:49Z |
| LIMIT_BAL | DECIMAL | false | CHECK (LIMIT_BAL > 0) | - | Credit limit amount assigned to the client account. | High | 2026-04-15T23:19:49Z |
| SEX | INTEGER | false | - | 1; 2 | [NEEDS CLARIFICATION] Coded value representing the client's sex category. | Low | 2026-04-15T23:19:49Z |
| EDUCATION | INTEGER | true | - | 0; 1; 2; 3; 4; 5; 6 | [NEEDS CLARIFICATION] Coded value representing the client's education category. | Low | 2026-04-15T23:19:49Z |
| MARRIAGE | INTEGER | true | - | 0; 1; 2; 3 | [NEEDS CLARIFICATION] Coded value representing the client's marital status. | Low | 2026-04-15T23:19:49Z |
| AGE | INTEGER | false | CHECK (AGE >= 18) | - | Age of the client, measured in years. | High | 2026-04-15T23:19:49Z |
| PAY_0 | UNKNOWN | true | - | -2; -1; 0; 1; 2; 3; 4; 5; 6; 7; 8; 9 | [NEEDS CLARIFICATION] Repayment status represented as a coded numeric value. | Low | 2026-04-15T23:19:49Z |
| PAY_2 | INTEGER | true | - | - | [NEEDS CLARIFICATION] Repayment status represented as a coded numeric value. | Low | 2026-04-15T23:19:49Z |
| PAY_3 | INTEGER | true | - | -2; -1; 0; 1; 2; 3; 4; 5; 6; 7; 8; 9 | Repayment status represented as a coded numeric value. | Medium | 2026-04-15T23:19:49Z |
| PAY_4 | INTEGER | true | - | -2; -1; 0; 1; 2; 3; 4; 5; 6; 7; 8; 9 | Repayment status represented as a coded numeric value. | Medium | 2026-04-15T23:19:49Z |
| PAY_5 | INTEGER | true | - | -2; -1; 0; 1; 2; 3; 4; 5; 6; 7; 8; 9 | Repayment status represented as a coded numeric value. | Medium | 2026-04-15T23:19:49Z |
| PAY_6 | INTEGER | true | - | -2; -1; 0; 1; 2; 3; 4; 5; 6; 7; 8; 9 | Repayment status represented as a coded numeric value. | Medium | 2026-04-15T23:19:49Z |
| BILL_AMT1 | DECIMAL | true | - | - | Bill statement amount for the referenced billing period. | High | 2026-04-15T23:19:49Z |
| BILL_AMT2 | DECIMAL | true | - | - | Bill statement amount for the referenced billing period. | High | 2026-04-15T23:19:49Z |
| BILL_AMT3 | DECIMAL | true | - | - | Bill statement amount for the referenced billing period. | High | 2026-04-15T23:19:49Z |
| BILL_AMT4 | DECIMAL | true | - | - | Bill statement amount for the referenced billing period. | High | 2026-04-15T23:19:49Z |
| BILL_AMT5 | DECIMAL | true | - | - | Bill statement amount for the referenced billing period. | High | 2026-04-15T23:19:49Z |
| BILL_AMT6 | DECIMAL | true | - | - | Bill statement amount for the referenced billing period. | High | 2026-04-15T23:19:49Z |
| PAY_AMT1 | DECIMAL | true | - | - | Payment amount made during the referenced billing period. | High | 2026-04-15T23:19:49Z |
| PAY_AMT2 | DECIMAL | true | - | - | Payment amount made during the referenced billing period. | High | 2026-04-15T23:19:49Z |
| PAY_AMT3 | DECIMAL | true | - | - | Payment amount made during the referenced billing period. | High | 2026-04-15T23:19:49Z |
| PAY_AMT4 | DECIMAL | true | - | - | Payment amount made during the referenced billing period. | High | 2026-04-15T23:19:49Z |
| PAY_AMT5 | DECIMAL | true | - | - | Payment amount made during the referenced billing period. | High | 2026-04-15T23:19:49Z |
| PAY_AMT6 | DECIMAL | true | - | - | Payment amount made during the referenced billing period. | High | 2026-04-15T23:19:49Z |
| default.payment.next.month | INTEGER | false | - | 0; 1 | Indicator of whether payment default occurs next month. | High | 2026-04-15T23:19:49Z |

## Evidence & Citations

### ID
1. field_name: 'ID' is specific and semantically informative.
2. schema_comments: provides additive descriptive context.
3. type: INTEGER is broad or code-oriented; limited semantic specificity.
4. constraints: semantically informative rule(s) support interpretation.
5. source_path: sample_schema.json -> table: credit_card_clients -> field: ID.

### LIMIT_BAL
1. field_name: 'LIMIT_BAL' is specific and semantically informative.
2. schema_comments: provides additive descriptive context.
3. type: DECIMAL is broad or code-oriented; limited semantic specificity.
4. constraints: semantically informative rule(s) support interpretation.
5. source_path: sample_schema.json -> table: credit_card_clients -> field: LIMIT_BAL.

### SEX
1. field_name: 'SEX' provides limited semantic detail.
2. schema_comments: present but mostly restates coded naming context.
3. type: INTEGER is broad or code-oriented; limited semantic specificity.
4. enums: numeric codes are not self-descriptive.
5. source_path: sample_schema.json -> table: credit_card_clients -> field: SEX.

### EDUCATION
1. field_name: 'EDUCATION' provides limited semantic detail.
2. schema_comments: present but mostly restates coded naming context.
3. type: INTEGER is broad or code-oriented; limited semantic specificity.
4. enums: numeric codes are not self-descriptive.
5. source_path: sample_schema.json -> table: credit_card_clients -> field: EDUCATION.

### MARRIAGE
1. field_name: 'MARRIAGE' provides limited semantic detail.
2. schema_comments: missing.
3. type: INTEGER is broad or code-oriented; limited semantic specificity.
4. enums: numeric codes are not self-descriptive.
5. source_path: sample_schema.json -> table: credit_card_clients -> field: MARRIAGE.

### AGE
1. field_name: 'AGE' is specific and semantically informative.
2. schema_comments: provides additive descriptive context.
3. type: INTEGER is broad or code-oriented; limited semantic specificity.
4. constraints: semantically informative rule(s) support interpretation.
5. source_path: sample_schema.json -> table: credit_card_clients -> field: AGE.

### PAY_0
1. field_name: 'PAY_0' suggests repayment status series, but index semantics are ambiguous.
2. schema_comments: missing.
3. type: UNKNOWN is broad or code-oriented; limited semantic specificity.
4. enums: numeric codes are not self-descriptive.
5. source_path: sample_schema.json -> table: credit_card_clients -> field: PAY_0.

### PAY_2
1. field_name: 'PAY_2' suggests repayment status series, but index semantics are ambiguous.
2. schema_comments: missing.
3. type: INTEGER is broad or code-oriented; limited semantic specificity.
4. source_path: sample_schema.json -> table: credit_card_clients -> field: PAY_2.

### PAY_3
1. field_name: 'PAY_3' suggests repayment status series, but index semantics are ambiguous.
2. schema_comments: provides additive descriptive context.
3. type: INTEGER is broad or code-oriented; limited semantic specificity.
4. enums: numeric codes are not self-descriptive.
5. source_path: sample_schema.json -> table: credit_card_clients -> field: PAY_3.

### PAY_4
1. field_name: 'PAY_4' suggests repayment status series, but index semantics are ambiguous.
2. schema_comments: provides additive descriptive context.
3. type: INTEGER is broad or code-oriented; limited semantic specificity.
4. enums: numeric codes are not self-descriptive.
5. source_path: sample_schema.json -> table: credit_card_clients -> field: PAY_4.

### PAY_5
1. field_name: 'PAY_5' suggests repayment status series, but index semantics are ambiguous.
2. schema_comments: provides additive descriptive context.
3. type: INTEGER is broad or code-oriented; limited semantic specificity.
4. enums: numeric codes are not self-descriptive.
5. source_path: sample_schema.json -> table: credit_card_clients -> field: PAY_5.

### PAY_6
1. field_name: 'PAY_6' suggests repayment status series, but index semantics are ambiguous.
2. schema_comments: provides additive descriptive context.
3. type: INTEGER is broad or code-oriented; limited semantic specificity.
4. enums: numeric codes are not self-descriptive.
5. source_path: sample_schema.json -> table: credit_card_clients -> field: PAY_6.

### BILL_AMT1
1. field_name: 'BILL_AMT1' is specific and semantically informative.
2. schema_comments: provides additive descriptive context.
3. type: DECIMAL is broad or code-oriented; limited semantic specificity.
4. source_path: sample_schema.json -> table: credit_card_clients -> field: BILL_AMT1.

### BILL_AMT2
1. field_name: 'BILL_AMT2' is specific and semantically informative.
2. schema_comments: provides additive descriptive context.
3. type: DECIMAL is broad or code-oriented; limited semantic specificity.
4. source_path: sample_schema.json -> table: credit_card_clients -> field: BILL_AMT2.

### BILL_AMT3
1. field_name: 'BILL_AMT3' is specific and semantically informative.
2. schema_comments: provides additive descriptive context.
3. type: DECIMAL is broad or code-oriented; limited semantic specificity.
4. source_path: sample_schema.json -> table: credit_card_clients -> field: BILL_AMT3.

### BILL_AMT4
1. field_name: 'BILL_AMT4' is specific and semantically informative.
2. schema_comments: provides additive descriptive context.
3. type: DECIMAL is broad or code-oriented; limited semantic specificity.
4. source_path: sample_schema.json -> table: credit_card_clients -> field: BILL_AMT4.

### BILL_AMT5
1. field_name: 'BILL_AMT5' is specific and semantically informative.
2. schema_comments: provides additive descriptive context.
3. type: DECIMAL is broad or code-oriented; limited semantic specificity.
4. source_path: sample_schema.json -> table: credit_card_clients -> field: BILL_AMT5.

### BILL_AMT6
1. field_name: 'BILL_AMT6' is specific and semantically informative.
2. schema_comments: provides additive descriptive context.
3. type: DECIMAL is broad or code-oriented; limited semantic specificity.
4. source_path: sample_schema.json -> table: credit_card_clients -> field: BILL_AMT6.

### PAY_AMT1
1. field_name: 'PAY_AMT1' is specific and semantically informative.
2. schema_comments: provides additive descriptive context.
3. type: DECIMAL is broad or code-oriented; limited semantic specificity.
4. source_path: sample_schema.json -> table: credit_card_clients -> field: PAY_AMT1.

### PAY_AMT2
1. field_name: 'PAY_AMT2' is specific and semantically informative.
2. schema_comments: provides additive descriptive context.
3. type: DECIMAL is broad or code-oriented; limited semantic specificity.
4. source_path: sample_schema.json -> table: credit_card_clients -> field: PAY_AMT2.

### PAY_AMT3
1. field_name: 'PAY_AMT3' is specific and semantically informative.
2. schema_comments: provides additive descriptive context.
3. type: DECIMAL is broad or code-oriented; limited semantic specificity.
4. source_path: sample_schema.json -> table: credit_card_clients -> field: PAY_AMT3.

### PAY_AMT4
1. field_name: 'PAY_AMT4' is specific and semantically informative.
2. schema_comments: provides additive descriptive context.
3. type: DECIMAL is broad or code-oriented; limited semantic specificity.
4. source_path: sample_schema.json -> table: credit_card_clients -> field: PAY_AMT4.

### PAY_AMT5
1. field_name: 'PAY_AMT5' is specific and semantically informative.
2. schema_comments: provides additive descriptive context.
3. type: DECIMAL is broad or code-oriented; limited semantic specificity.
4. source_path: sample_schema.json -> table: credit_card_clients -> field: PAY_AMT5.

### PAY_AMT6
1. field_name: 'PAY_AMT6' is specific and semantically informative.
2. schema_comments: provides additive descriptive context.
3. type: DECIMAL is broad or code-oriented; limited semantic specificity.
4. source_path: sample_schema.json -> table: credit_card_clients -> field: PAY_AMT6.

### default.payment.next.month
1. field_name: 'default.payment.next.month' is specific and semantically informative.
2. schema_comments: provides additive descriptive context.
3. type: INTEGER is broad or code-oriented; limited semantic specificity.
4. enums: numeric codes are not self-descriptive.
5. source_path: sample_schema.json -> table: credit_card_clients -> field: default.payment.next.month.

## Confidence Legend

- **High**: Multiple strong signals agree on field meaning.
- **Medium**: Some signals are present, but ambiguity remains.
- **Low**: Limited or conflicting signals; human clarification is required.

## Generation Notes

- Descriptions are AI-generated drafts for human review.
- Fields marked `[NEEDS CLARIFICATION]` require human attention before production use.
