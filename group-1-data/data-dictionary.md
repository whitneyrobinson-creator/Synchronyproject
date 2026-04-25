# Data Dictionary — obesity_dataset_engineered

**Source File:** ObesityDataSet_engineered.csv  
**Generated:** 2026-04-25T00:28:38Z

---

## Field Summary

| field_name | type | nullable | constraints | enums | description | confidence | last_verified |
|---|---|---|---|---|---|---|---|
| gender | VARCHAR | false | - | Female; Male | Self-reported gender category for the individual. | High | 2026-04-25T00:28:38Z |
| age | INTEGER | false | - | - | Age of the individual in years. | High | 2026-04-25T00:28:38Z |
| height | DECIMAL | false | - | - | Height measurement for the individual. | High | 2026-04-25T00:28:38Z |
| weight | DECIMAL | false | - | - | Weight measurement for the individual. | High | 2026-04-25T00:28:38Z |
| family_history_with_overweight | VARCHAR | false | - | no; yes | Indicates whether family history includes overweight conditions. | High | 2026-04-25T00:28:38Z |
| favc | VARCHAR | false | - | no; yes | Indicates whether the individual frequently consumes high-calorie food. | High | 2026-04-25T00:28:38Z |
| fcvc | INTEGER | false | - | 1.0; 2.0; 3.0 | Frequency score for vegetable consumption. | Medium | 2026-04-25T00:28:38Z |
| ncp | INTEGER | false | - | 1; 3; 4 | Number of main meals consumed per day. | Medium | 2026-04-25T00:28:38Z |
| caec | VARCHAR | false | - | Always; Frequently; Sometimes; no | Frequency category for eating food between main meals. | High | 2026-04-25T00:28:38Z |
| smoke | VARCHAR | false | - | no; yes | Indicates whether the individual smokes. | High | 2026-04-25T00:28:38Z |
| ch2o | INTEGER | false | - | 1.0; 2.0; 3.0 | Daily water consumption level score. | Medium | 2026-04-25T00:28:38Z |
| scc | VARCHAR | false | - | no; yes | Indicates whether the individual monitors calorie intake. | High | 2026-04-25T00:28:38Z |
| faf | INTEGER | false | - | 0.0; 1.0; 2.0; 3.0 | Physical activity frequency score. | Medium | 2026-04-25T00:28:38Z |
| tue | INTEGER | false | - | 0; 1; 2 | Daily technology-use time score. | Medium | 2026-04-25T00:28:38Z |
| calc | VARCHAR | false | - | Frequently; Sometimes; no | Frequency category for alcohol consumption. | High | 2026-04-25T00:28:38Z |
| mtrans | VARCHAR | false | - | Automobile; Bike; Motorbike; Public_Transportation; Walking | Primary transportation mode used by the individual. | High | 2026-04-25T00:28:38Z |
| nobeyesdad | VARCHAR | false | - | Insufficient_Weight; Normal_Weight; Obesity_Type_I; Obesity_Type_II; Obesity_Type_III; Overweight_Level_I; Overweight_Level_II | Obesity level classification label for the individual. | High | 2026-04-25T00:28:38Z |
| feat_mean_age_by_nobeyesdad | DECIMAL | false | - | - | Mean age value computed within each nobeyesdad group. | High | 2026-04-25T00:28:38Z |
| feat_mean_faf_by_nobeyesdad | DECIMAL | false | - | - | Mean faf value computed within each nobeyesdad group. | High | 2026-04-25T00:28:38Z |
| feat_mean_weight_by_gender | DECIMAL | false | - | - | Mean weight value computed within each gender group. | High | 2026-04-25T00:28:38Z |
| feat_bmi | DECIMAL | false | - | - | Engineered body mass index feature derived from height and weight. | High | 2026-04-25T00:28:38Z |
| feat_caloric_balance_proxy | INTEGER | false | - | -2.0; -1.0; 0.0; 1.0; 2.0; 3.0 | Engineered proxy score representing caloric intake versus expenditure balance. | Medium | 2026-04-25T00:28:38Z |
| feat_caec_one_hot_always | INTEGER | false | - | 0; 1 | One-hot indicator showing whether caec equals always. | Medium | 2026-04-25T00:28:38Z |
| feat_caec_one_hot_frequently | INTEGER | false | - | 0; 1 | One-hot indicator showing whether caec equals frequently. | Medium | 2026-04-25T00:28:38Z |
| feat_caec_one_hot_no | INTEGER | false | - | 0; 1 | One-hot indicator showing whether caec equals no. | Medium | 2026-04-25T00:28:38Z |
| feat_caec_one_hot_sometimes | INTEGER | false | - | 0; 1 | One-hot indicator showing whether caec equals sometimes. | Medium | 2026-04-25T00:28:38Z |
| feat_calc_one_hot_frequently | INTEGER | false | - | 0; 1 | One-hot indicator showing whether calc equals frequently. | Medium | 2026-04-25T00:28:38Z |
| feat_calc_one_hot_no | INTEGER | false | - | 0; 1 | One-hot indicator showing whether calc equals no. | Medium | 2026-04-25T00:28:38Z |
| feat_calc_one_hot_sometimes | INTEGER | false | - | 0; 1 | One-hot indicator showing whether calc equals sometimes. | Medium | 2026-04-25T00:28:38Z |
| feat_family_history_encoded | INTEGER | false | - | 0; 1 | Numeric encoded representation of family history categories. | Medium | 2026-04-25T00:28:38Z |
| feat_favc_encoded | INTEGER | false | - | 0; 1 | Numeric encoded representation of favc categories. | Medium | 2026-04-25T00:28:38Z |
| feat_gender_encoded | INTEGER | false | - | 0; 1 | Numeric encoded representation of gender categories. | Medium | 2026-04-25T00:28:38Z |
| feat_mtrans_one_hot_automobile | INTEGER | false | - | 0; 1 | One-hot indicator showing whether mtrans equals automobile. | Medium | 2026-04-25T00:28:38Z |
| feat_mtrans_one_hot_bike | INTEGER | false | - | 0; 1 | One-hot indicator showing whether mtrans equals bike. | Medium | 2026-04-25T00:28:38Z |
| feat_mtrans_one_hot_motorbike | INTEGER | false | - | 0; 1 | One-hot indicator showing whether mtrans equals motorbike. | Medium | 2026-04-25T00:28:38Z |
| feat_mtrans_one_hot_public_transportation | INTEGER | false | - | 0; 1 | One-hot indicator showing whether mtrans equals public transportation. | Medium | 2026-04-25T00:28:38Z |
| feat_mtrans_one_hot_walking | INTEGER | false | - | 0; 1 | One-hot indicator showing whether mtrans equals walking. | Medium | 2026-04-25T00:28:38Z |
| feat_scc_encoded | INTEGER | false | - | 0; 1 | Numeric encoded representation of scc categories. | Medium | 2026-04-25T00:28:38Z |
| feat_smoke_encoded | INTEGER | false | - | 0; 1 | Numeric encoded representation of smoke categories. | Medium | 2026-04-25T00:28:38Z |
| feat_age_scaled | DECIMAL | false | - | - | Scaled version of the age feature. | High | 2026-04-25T00:28:38Z |
| feat_bmi_z_score | DECIMAL | false | - | - | Z-score standardized value of bmi. | High | 2026-04-25T00:28:38Z |
| feat_height_scaled | DECIMAL | false | - | - | Scaled version of the height feature. | High | 2026-04-25T00:28:38Z |
| feat_weight_scaled | DECIMAL | false | - | - | Scaled version of the weight feature. | High | 2026-04-25T00:28:38Z |

## Evidence & Citations

### gender
1. field_name: 'gender' provides interpretable feature context.
2. enums: categorical values ['Female', 'Male'] provide semantic category clues.
3. type: VARCHAR supports categorical text values.
4. schema_comments: missing; confidence is based on available signals only.

### age
1. field_name: 'age' provides interpretable feature context.
2. enums: not provided for this field.
3. type: INTEGER aligns with a quantitative measurement feature.
4. schema_comments: missing; confidence is based on available signals only.

### height
1. field_name: 'height' provides interpretable feature context.
2. enums: not provided for this field.
3. type: DECIMAL aligns with a quantitative measurement feature.
4. schema_comments: missing; confidence is based on available signals only.

### weight
1. field_name: 'weight' provides interpretable feature context.
2. enums: not provided for this field.
3. type: DECIMAL aligns with a quantitative measurement feature.
4. schema_comments: missing; confidence is based on available signals only.

### family_history_with_overweight
1. field_name: 'family_history_with_overweight' provides interpretable feature context.
2. enums: categorical values ['no', 'yes'] provide semantic category clues.
3. type: VARCHAR supports categorical text values.
4. schema_comments: missing; confidence is based on available signals only.

### favc
1. field_name: 'favc' provides interpretable feature context.
2. enums: categorical values ['no', 'yes'] provide semantic category clues.
3. type: VARCHAR supports categorical text values.
4. schema_comments: missing; confidence is based on available signals only.

### fcvc
1. field_name: 'fcvc' provides interpretable feature context.
2. enums: compact numeric codes [1.0, 2.0, 3.0] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### ncp
1. field_name: 'ncp' provides interpretable feature context.
2. enums: compact numeric codes [1, 3, 4] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### caec
1. field_name: 'caec' provides interpretable feature context.
2. enums: categorical values ['Always', 'Frequently', 'Sometimes', 'no'] provide semantic category clues.
3. type: VARCHAR supports categorical text values.
4. schema_comments: missing; confidence is based on available signals only.

### smoke
1. field_name: 'smoke' provides interpretable feature context.
2. enums: categorical values ['no', 'yes'] provide semantic category clues.
3. type: VARCHAR supports categorical text values.
4. schema_comments: missing; confidence is based on available signals only.

### ch2o
1. field_name: 'ch2o' provides interpretable feature context.
2. enums: compact numeric codes [1.0, 2.0, 3.0] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### scc
1. field_name: 'scc' provides interpretable feature context.
2. enums: categorical values ['no', 'yes'] provide semantic category clues.
3. type: VARCHAR supports categorical text values.
4. schema_comments: missing; confidence is based on available signals only.

### faf
1. field_name: 'faf' provides interpretable feature context.
2. enums: compact numeric codes [0.0, 1.0, 2.0, 3.0] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### tue
1. field_name: 'tue' provides interpretable feature context.
2. enums: compact numeric codes [0, 1, 2] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### calc
1. field_name: 'calc' provides interpretable feature context.
2. enums: categorical values ['Frequently', 'Sometimes', 'no'] provide semantic category clues.
3. type: VARCHAR supports categorical text values.
4. schema_comments: missing; confidence is based on available signals only.

### mtrans
1. field_name: 'mtrans' provides interpretable feature context.
2. enums: categorical values ['Automobile', 'Bike', 'Motorbike', 'Public_Transportation', 'Walking'] provide semantic category clues.
3. type: VARCHAR supports categorical text values.
4. schema_comments: missing; confidence is based on available signals only.

### nobeyesdad
1. field_name: 'nobeyesdad' provides interpretable feature context.
2. enums: categorical values ['Insufficient_Weight', 'Normal_Weight', 'Obesity_Type_I', 'Obesity_Type_II', 'Obesity_Type_III', 'Overweight_Level_I'] provide semantic category clues.
3. type: VARCHAR supports categorical text values.
4. schema_comments: missing; confidence is based on available signals only.

### feat_mean_age_by_nobeyesdad
1. field_name: 'feat_mean_age_by_nobeyesdad' provides interpretable feature context.
2. enums: numeric code range is present but labels are not defined.
3. type: DECIMAL aligns with a quantitative measurement feature.
4. schema_comments: missing; confidence is based on available signals only.

### feat_mean_faf_by_nobeyesdad
1. field_name: 'feat_mean_faf_by_nobeyesdad' provides interpretable feature context.
2. enums: numeric code range is present but labels are not defined.
3. type: DECIMAL aligns with a quantitative measurement feature.
4. schema_comments: missing; confidence is based on available signals only.

### feat_mean_weight_by_gender
1. field_name: 'feat_mean_weight_by_gender' provides interpretable feature context.
2. enums: compact numeric codes [63.4882758621, 75.5901960784] suggest categorical encoding without labels.
3. type: DECIMAL aligns with a quantitative measurement feature.
4. schema_comments: missing; confidence is based on available signals only.

### feat_bmi
1. field_name: 'feat_bmi' provides interpretable feature context.
2. enums: not provided for this field.
3. type: DECIMAL aligns with a quantitative measurement feature.
4. schema_comments: missing; confidence is based on available signals only.

### feat_caloric_balance_proxy
1. field_name: 'feat_caloric_balance_proxy' provides interpretable feature context.
2. enums: numeric code range is present but labels are not defined.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### feat_caec_one_hot_always
1. field_name: 'feat_caec_one_hot_always' provides interpretable feature context.
2. enums: compact numeric codes [0, 1] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### feat_caec_one_hot_frequently
1. field_name: 'feat_caec_one_hot_frequently' provides interpretable feature context.
2. enums: compact numeric codes [0, 1] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### feat_caec_one_hot_no
1. field_name: 'feat_caec_one_hot_no' provides interpretable feature context.
2. enums: compact numeric codes [0, 1] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### feat_caec_one_hot_sometimes
1. field_name: 'feat_caec_one_hot_sometimes' provides interpretable feature context.
2. enums: compact numeric codes [0, 1] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### feat_calc_one_hot_frequently
1. field_name: 'feat_calc_one_hot_frequently' provides interpretable feature context.
2. enums: compact numeric codes [0, 1] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### feat_calc_one_hot_no
1. field_name: 'feat_calc_one_hot_no' provides interpretable feature context.
2. enums: compact numeric codes [0, 1] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### feat_calc_one_hot_sometimes
1. field_name: 'feat_calc_one_hot_sometimes' provides interpretable feature context.
2. enums: compact numeric codes [0, 1] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### feat_family_history_encoded
1. field_name: 'feat_family_history_encoded' provides interpretable feature context.
2. enums: compact numeric codes [0, 1] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### feat_favc_encoded
1. field_name: 'feat_favc_encoded' provides interpretable feature context.
2. enums: compact numeric codes [0, 1] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### feat_gender_encoded
1. field_name: 'feat_gender_encoded' provides interpretable feature context.
2. enums: compact numeric codes [0, 1] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### feat_mtrans_one_hot_automobile
1. field_name: 'feat_mtrans_one_hot_automobile' provides interpretable feature context.
2. enums: compact numeric codes [0, 1] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### feat_mtrans_one_hot_bike
1. field_name: 'feat_mtrans_one_hot_bike' provides interpretable feature context.
2. enums: compact numeric codes [0, 1] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### feat_mtrans_one_hot_motorbike
1. field_name: 'feat_mtrans_one_hot_motorbike' provides interpretable feature context.
2. enums: compact numeric codes [0, 1] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### feat_mtrans_one_hot_public_transportation
1. field_name: 'feat_mtrans_one_hot_public_transportation' provides interpretable feature context.
2. enums: compact numeric codes [0, 1] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### feat_mtrans_one_hot_walking
1. field_name: 'feat_mtrans_one_hot_walking' provides interpretable feature context.
2. enums: compact numeric codes [0, 1] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### feat_scc_encoded
1. field_name: 'feat_scc_encoded' provides interpretable feature context.
2. enums: compact numeric codes [0, 1] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### feat_smoke_encoded
1. field_name: 'feat_smoke_encoded' provides interpretable feature context.
2. enums: compact numeric codes [0, 1] suggest categorical encoding without labels.
3. type: INTEGER is available but only weakly constrains semantics.
4. schema_comments: missing; confidence is based on available signals only.

### feat_age_scaled
1. field_name: 'feat_age_scaled' provides interpretable feature context.
2. enums: not provided for this field.
3. type: DECIMAL aligns with a quantitative measurement feature.
4. schema_comments: missing; confidence is based on available signals only.

### feat_bmi_z_score
1. field_name: 'feat_bmi_z_score' provides interpretable feature context.
2. enums: not provided for this field.
3. type: DECIMAL aligns with a quantitative measurement feature.
4. schema_comments: missing; confidence is based on available signals only.

### feat_height_scaled
1. field_name: 'feat_height_scaled' provides interpretable feature context.
2. enums: not provided for this field.
3. type: DECIMAL aligns with a quantitative measurement feature.
4. schema_comments: missing; confidence is based on available signals only.

### feat_weight_scaled
1. field_name: 'feat_weight_scaled' provides interpretable feature context.
2. enums: not provided for this field.
3. type: DECIMAL aligns with a quantitative measurement feature.
4. schema_comments: missing; confidence is based on available signals only.

## Confidence Legend

- **High**: Multiple strong signals agree on field meaning.
- **Medium**: Some signals are present, but ambiguity remains.
- **Low**: Limited or conflicting signals; human clarification is required.

## Generation Notes

- Descriptions are AI-generated drafts for human review.
- Fields marked `[NEEDS CLARIFICATION]` require human attention before production use.
