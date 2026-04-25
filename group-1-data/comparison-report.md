# Comparison Report — obesity_dataset_engineered

**Generated:** 2026-04-25T00:28:38Z
**Pipeline Output:** data_dictionary.md
**Ground Truth:** group1_ground_truth.md

## Summary
| Metric | Value |
|--------|-------|
| Total Fields | 43 |
| Fields with Full Metadata | 25 |
| Fields with Limited Metadata | 18 |
| Fields with No Metadata | 0 |

## Field-by-Field Comparison
| Field | Ground Truth | Skill Output | Metadata Available |
|-------|-------------|--------------|-------------------|
| gender | Self-reported gender category for the individual (Female or Male) | Self-reported gender category for the individual. | ⚠️ Limited |
| age | Age of the individual in years | Age of the individual in years. | ⚠️ Limited |
| height | Height measurement of the individual in meters | Height measurement for the individual. | ⚠️ Limited |
| weight | Body weight of the individual in kilograms | Weight measurement for the individual. | ⚠️ Limited |
| family_history_with_overweight | Indicates whether the individual has a family history of being overweight (yes or no) | Indicates whether family history includes overweight conditions. | ✅ Full |
| favc | Indicates whether the individual frequently consumes high-calorie food (yes or no) | Indicates whether the individual frequently consumes high-calorie food. | ✅ Full |
| fcvc | Frequency score for vegetable consumption on an ordinal scale of 1 to 3 | Frequency score for vegetable consumption. | ⚠️ Limited |
| ncp | Number of main meals consumed per day | Number of main meals consumed per day. | ✅ Full |
| caec | Frequency category for eating food between main meals (Always, Frequently, Sometimes, or no) | Frequency category for eating food between main meals. | ✅ Full |
| smoke | Indicates whether the individual smokes (yes or no) | Indicates whether the individual smokes. | ⚠️ Limited |
| ch2o | Daily water consumption level score on an ordinal scale of 1 to 3 | Daily water consumption level score. | ⚠️ Limited |
| scc | Indicates whether the individual monitors calorie intake (yes or no) | Indicates whether the individual monitors calorie intake. | ✅ Full |
| faf | Physical activity frequency score on an ordinal scale of 0 to 3 | Physical activity frequency score. | ⚠️ Limited |
| tue | Daily technology use time score on an ordinal scale of 0 to 2 | Daily technology-use time score. | ⚠️ Limited |
| calc | Frequency category for alcohol consumption (Frequently, Sometimes, or no) | Frequency category for alcohol consumption. | ⚠️ Limited |
| mtrans | Primary transportation mode used by the individual (Automobile, Bike, Motorbike, Public Transportation, or Walking) | Primary transportation mode used by the individual. | ✅ Full |
| nobeyesdad | Obesity level classification label for the individual across seven categories from Insufficient Weight to Obesity Type III | Obesity level classification label for the individual. | ✅ Full |
| feat_mean_age_by_nobeyesdad | Average age in years of all subjects in the same obesity classification as this row computed using group-by aggregation | Mean age value computed within each nobeyesdad group. | ✅ Full |
| feat_mean_faf_by_nobeyesdad | Average physical activity frequency score of all subjects in the same obesity class as this row computed using group-by aggregation | Mean faf value computed within each nobeyesdad group. | ✅ Full |
| feat_mean_weight_by_gender | Average body weight in kilograms for all subjects of the same gender as this row computed using group-by aggregation | Mean weight value computed within each gender group. | ✅ Full |
| feat_bmi | Body Mass Index computed as weight in kilograms divided by height in meters squared | Engineered body mass index feature derived from height and weight. | ✅ Full |
| feat_caloric_balance_proxy | Engineered proxy score representing the difference between vegetable consumption frequency and physical activity frequency | Engineered proxy score representing caloric intake versus expenditure balance. | ✅ Full |
| feat_caec_one_hot_always | Binary indicator that equals 1 if the individual always eats between meals and 0 otherwise | One-hot indicator showing whether caec equals always. | ✅ Full |
| feat_caec_one_hot_frequently | Binary indicator that equals 1 if the individual frequently eats between meals and 0 otherwise | One-hot indicator showing whether caec equals frequently. | ✅ Full |
| feat_caec_one_hot_no | Binary indicator that equals 1 if the individual does not eat between meals and 0 otherwise | One-hot indicator showing whether caec equals no. | ✅ Full |
| feat_caec_one_hot_sometimes | Binary indicator that equals 1 if the individual sometimes eats between meals and 0 otherwise | One-hot indicator showing whether caec equals sometimes. | ✅ Full |
| feat_calc_one_hot_frequently | Binary indicator that equals 1 if the individual frequently consumes alcohol and 0 otherwise | One-hot indicator showing whether calc equals frequently. | ✅ Full |
| feat_calc_one_hot_no | Binary indicator that equals 1 if the individual does not consume alcohol and 0 otherwise | One-hot indicator showing whether calc equals no. | ✅ Full |
| feat_calc_one_hot_sometimes | Binary indicator that equals 1 if the individual sometimes consumes alcohol and 0 otherwise | One-hot indicator showing whether calc equals sometimes. | ✅ Full |
| feat_family_history_encoded | Numeric label encoding of family history with overweight where no equals 0 and yes equals 1 | Numeric encoded representation of family history categories. | ✅ Full |
| feat_favc_encoded | Numeric label encoding of frequent high-caloric food consumption where no equals 0 and yes equals 1 | Numeric encoded representation of favc categories. | ⚠️ Limited |
| feat_gender_encoded | Numeric label encoding of gender where Female equals 0 and Male equals 1 | Numeric encoded representation of gender categories. | ⚠️ Limited |
| feat_mtrans_one_hot_automobile | Binary indicator that equals 1 if the individual primarily uses an automobile for transportation and 0 otherwise | One-hot indicator showing whether mtrans equals automobile. | ✅ Full |
| feat_mtrans_one_hot_bike | Binary indicator that equals 1 if the individual primarily uses a bike for transportation and 0 otherwise | One-hot indicator showing whether mtrans equals bike. | ✅ Full |
| feat_mtrans_one_hot_motorbike | Binary indicator that equals 1 if the individual primarily uses a motorbike for transportation and 0 otherwise | One-hot indicator showing whether mtrans equals motorbike. | ✅ Full |
| feat_mtrans_one_hot_public_transportation | Binary indicator that equals 1 if the individual primarily uses public transportation and 0 otherwise | One-hot indicator showing whether mtrans equals public transportation. | ✅ Full |
| feat_mtrans_one_hot_walking | Binary indicator that equals 1 if the individual primarily walks as their mode of transportation and 0 otherwise | One-hot indicator showing whether mtrans equals walking. | ✅ Full |
| feat_scc_encoded | Numeric label encoding of calorie monitoring behavior where no equals 0 and yes equals 1 | Numeric encoded representation of scc categories. | ⚠️ Limited |
| feat_smoke_encoded | Numeric label encoding of smoking status where no equals 0 and yes equals 1 | Numeric encoded representation of smoke categories. | ⚠️ Limited |
| feat_age_scaled | Age in years rescaled to the range 0 to 1 using min-max scaling | Scaled version of the age feature. | ⚠️ Limited |
| feat_bmi_z_score | BMI standardized to mean 0 and standard deviation 1 using z-score standardization | Z-score standardized value of bmi. | ⚠️ Limited |
| feat_height_scaled | Height in meters rescaled to the range 0 to 1 using min-max scaling | Scaled version of the height feature. | ⚠️ Limited |
| feat_weight_scaled | Weight in kilograms rescaled to the range 0 to 1 using min-max scaling | Scaled version of the weight feature. | ⚠️ Limited |
