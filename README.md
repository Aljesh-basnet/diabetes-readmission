# Diabetes 30-Day Readmission Analysis

> **Status:** In progress. Data audit complete; cohort derivation and feature engineering underway.

## Question
Among adult inpatients with diabetes, which factors are associated with readmission within 30 days of discharge, and how well can that risk be flagged at discharge?

## Data
- **Source:** [Diabetes 130-US Hospitals for Years 1999-2008](https://archive.ics.uci.edu/dataset/296/diabetes+130-us+hospitals+for+years+1999-2008) (UCI Machine Learning Repository; also on Kaggle).
- 101,766 inpatient encounters for 71,518 patients across 130 US hospitals; 50 variables covering demographics, admission details, ICD-9 diagnoses, lab tests, 23 diabetes medications and readmission status.
- Raw data is not stored in this repository. Download `diabetic_data.csv` and `IDs_mapping.csv` into `data/raw/`.
- **Reference study:** Strack B, et al. (2014). *Impact of HbA1c Measurement on Hospital Readmission Rates: Analysis of 70,000 Clinical Database Patient Records.* BioMed Research International.

## Outcome
30-day readmission (`readmitted == "<30"`) = 1; readmission after 30 days or no readmission = 0. The 30-day window is the standard hospital quality measure and matches the reference study.

| readmitted | Encounters | % |
|---|---|---|
| NO | 54,864 | 53.9 |
| >30 | 35,545 | 34.9 |
| <30 | 11,357 | 11.2 |

Because a model predicting "no readmission" for everyone would be 88.8% accurate, performance will be reported with AUROC, PR-AUC and calibration rather than accuracy.

## Data audit: key findings
Every issue is logged with record counts and a decision in [`docs/qc_findings_log.md`](docs/qc_findings_log.md).

1. **Default CSV loading would have deleted the main exposure.** The dataset uses the text `"None"` to mean *test not performed*. pandas' default reader converts it to missing, which wrongly flagged 84,748 HbA1c values (83.3%) as missing while leaving the real missing marker (`?`) as text. Data is loaded with `keep_default_na=False, na_values=["?"]`, and counts were verified before and after.
2. **Repeat patients.** 16,773 patients (23.5%) have more than one encounter and account for 46.2% of all rows (one patient has 40). The analysis keeps each patient's first encounter, so the same person cannot appear in both training and test data.
3. **Missing data.** Weight is 96.9% missing and is dropped. Medical specialty (49.1%), payer code (39.6%) and race (2.2%) are kept with "Missing" as its own category. Missingness in specialty and payer code is not meaningfully related to readmission (<1 point difference), but **encounters with missing race are readmitted less often (8.3% vs 11.2%)**, so it is not random.
4. **Lab testing is rare.** HbA1c was measured in 16.7% of encounters and serum glucose in 5.3%.
5. **Skewed visit counts.** Prior-year emergency visits have a median of 0 and a 99th percentile of 3, but a maximum of 76. Extreme values are real patients and are retained; they are capped for logistic regression only.

## Cohort derivation
| Step | Rule | Encounters | Removed | 30-day readmission % |
|---|---|---|---|---|
| 0 | All encounters | 101,766 | | 11.2 |
| 1 | Exclude expired or hospice discharges | TODO | TODO | TODO |
| 2 | Exclude unknown/invalid gender | TODO | TODO | TODO |
| 3 | Keep first encounter per patient | TODO | TODO | TODO |

Assumption: `encounter_id` increases over time, so the lowest ID is the earliest encounter (the dataset has no dates).

## Analysis plan
Written and committed before any modelling: [`docs/analysis_plan.md`](docs/analysis_plan.md).

- [x] Data audit and QC log
- [ ] Cohort derivation
- [ ] Feature engineering (ICD-9 grouping, HbA1c groups, medication changes)
- [ ] Table 1: baseline characteristics by readmission status
- [ ] Replicate Strack et al.: adjusted association between HbA1c testing and readmission
- [ ] Prediction: logistic regression and XGBoost, with calibration and SHAP
- [ ] Dashboard

## Key findings
TODO. Every number reported here comes from the code in this repository.

## Repository structure
```
├── data/
│   ├── raw/                 # source CSVs (not committed)
│   └── processed/           # derived files (not committed)
├── docs/
│   ├── analysis_plan.md
│   ├── data_dictionary.md
│   └── qc_findings_log.md
├── notebooks/
│   ├── 01_data_audit.ipynb
│   └── 02_features.ipynb
├── sql/01_cohort.sql
├── src/
└── outputs/                 # tables and figures
```

## How to reproduce
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```
Place the two source files in `data/raw/`, then run the notebooks in order.

## Limitations
- US data from 1999-2008, coded in ICD-9; findings do not transfer directly to current Canadian hospitals.
- No social determinants of health and no post-discharge information.
- Readmissions to hospitals outside the network are not captured.
- No hospital identifier, so differences in recording practice between hospitals (for example, the race missingness pattern) cannot be examined.
- Encounter order is inferred from `encounter_id`, as the data contains no dates.