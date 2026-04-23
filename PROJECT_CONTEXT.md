# Project Context: AF in ICU HF Patients (MIMIC-IV)

> **Purpose:** Onboarding doc for Claude Code / new AI sessions / team collaboration. Captures key decisions, open questions, and the current state of thinking so a new session doesn't need to re-derive context.

---

## 1. Project Overview

- **Course:** Harvard clinical epidemiology / health informatics final project
- **Deadline:** April 27, 2026 (~1 week out from 2026-04-20)
- **Deliverables:** 5–10 page written analysis + 15–30 min presentation
- **Team:**
  - **Zihan** (PhD student, first author): study design, modeling strategy, writing, DAG, methods
  - **Vicky** (labmate): data extraction, cleaning, statistical analysis, SQL/R implementation
- **Both self-identify as beginners** — toolchain decisions should prioritize simplicity
- **Dataset:** MIMIC-IV v3.1 (local `.csv.gz` files already downloaded)

---

## 2. Research Question (Current Version)

### 2.1 Title (draft)
*Atrial Fibrillation Documented During Hospitalization for Acute Heart Failure and In-Hospital Outcomes Among Critically Ill Patients Without Prior AF: A Retrospective Cohort Study Using MIMIC-IV*

### 2.2 Primary question
Among adult ICU patients admitted for acute heart failure with **no prior AF** recorded in MIMIC-IV, is AF documented during the index hospitalization associated with increased in-hospital mortality and prolonged length of stay, after adjusting for demographics, clinical factors, and severity of illness?

### 2.3 Secondary question
Does the association differ by HF subtype (HFrEF vs HFpEF)?

### 2.4 Important terminology
- Use **"hospital-documented AF"**, NOT "new-onset" or "incident" AF — ICD codes cannot definitively distinguish truly new-onset from previously undiagnosed pre-existing AF. This terminology choice is a methodological strength to highlight.
- Avoid causal language ("drivers", "causes") — study design is observational and does not support causal inference. DAG is required.

---

## 3. Novelty Assessment (Critical Issue)

### 3.1 The problem
The current framing is **methodologically close to Yan et al. 2023** (the paper whose ICD definitions we adapted). An observant reviewer/professor will ask: *"What is the delta from Yan 2023?"*

### 3.2 The proposed fix: add an informatics layer
**Primary analysis (epi backbone):**
- Multivariable logistic regression for AF → in-hospital mortality, adjusted OR (confounder selection driven by DAG, NOT by stepwise/AIC)
- This partially replicates/confirms Yan 2023 — acts as sanity check

**Novel layer (informatics value-add):**
- Train an **XGBoost** model to predict hospital-documented AF using **first-24h ICU features** (strict temporal ordering: predictors must precede outcome)
- Apply **SHAP** to identify which early-ICU phenotypes flag high AF risk
- Framing: *"We extend Yan et al. by developing and interpreting an early-warning prediction model for hospital-documented AF using SHAP"*

### 3.3 Why this works
- Reuses existing cohort/covariate work (no wasted effort)
- Combines traditional epi (logistic regression, DAG, adjusted OR) with informatics (ML + interpretability) — matches course's clinical-epi-+-informatics framing
- Feasible in ~1 week with beginner team

---

## 4. Sample Size Considerations

### 4.1 Model 1 (logistic regression for AF → mortality)
- Follows **Events Per Variable (EPV) ≥ 10–20** rule (Peduzzi 1996 / Riley 2020)
- Expected mortality ~10–15% in ICU HF population → ~500–1,000 deaths in final cohort
- Supports ~25–100 parameters → **more than sufficient** for ~25–30 covariates
- **Not the bottleneck.**

### 4.2 Model 2 (XGBoost + SHAP predicting AF)
- Rule of thumb: minority class **≥ 500 events** for stable XGBoost + SHAP, **≥ 1,000** is comfortable
- Expected AF prevalence in cohort: ~15–25% → expected ~600–1,200 AF events in final cohort
- **Likely sufficient for overall cohort**, but HFrEF/HFpEF subgroup SHAP may be tight due to HF subtype ICD-coding specificity issues

### 4.3 Decision rule (to apply after Vicky pulls cohort)
- If AF events **< 500** → drop XGBoost, use **penalized logistic regression (LASSO)** with coefficient interpretation instead
- If AF events **500–1,000** → XGBoost OK overall, but subgroup SHAP is descriptive only
- If AF events **≥ 1,000** → full plan including subgroup SHAP is feasible

---

## 5. Cohort Definition (From Workflow Doc)

### 5.1 PECO
| | |
|---|---|
| Population | Adults (≥18) admitted to ICU for acute HF, with no prior AF diagnosis in MIMIC-IV. First qualifying admission per patient only. |
| Exposure | AF documented during index hospitalization (binary). |
| Comparison | HF ICU patients without AF documented during index hospitalization. |
| Outcome | Primary: in-hospital mortality (binary). Secondary: ICU LOS, hospital LOS. |

### 5.2 ICD Codes (from Yan et al. 2023)

**Acute HF (inclusion):**
- ICD-9: `42821, 42822, 42823, 42831, 42832, 42833, 42841, 42842, 42843`
- ICD-10: `I5021, I5022, I5023, I5031, I5032, I5033, I5041, I5042, I5043, I50811, I50812, I50813`

**HF subtype classification:**
- Systolic (HFrEF): `I5021-23`, `42821-23`
- Diastolic (HFpEF): `I5031-33`, `42831-33`
- Combined (mixed): `I5041-43`, `42841-43`

**Atrial Fibrillation:**
- ICD-9: `42731`
- ICD-10: `I480, I481, I482, I4891`

### 5.3 Inclusion / Exclusion

**Include:**
- Age ≥ 18 at admission
- Acute HF diagnosis during index admission
- ≥ 1 ICU stay during index admission
- First qualifying HF admission per patient

**Exclude:**
- Prior AF diagnosis in any MIMIC-IV admission before index
- ICU stay < 24 hours
- Missing survival/discharge info
- Comfort-measures-only or discharged AMA

### 5.4 Data windows
- **Baseline:** first 24h after ICU `intime` (first recorded measurement per variable)
- **Exposure ascertainment:** full index hospitalization + all prior MIMIC admissions (for AF history)
- **Outcome window:** ICU admission → hospital discharge or in-hospital death

---

## 6. Variables to Extract

### 6.1 Outcomes
| Variable | Source | Type |
|---|---|---|
| In-hospital mortality | `admissions.hospital_expire_flag` | binary |
| ICU LOS | `icustays` (outtime − intime) | continuous (days) |
| Hospital LOS | `admissions` (dischtime − admittime) | continuous (days) |

### 6.2 Covariates
- **Demographics:** age, sex, race (White/Black/Other), insurance
- **Vitals (first 24h):** HR, SBP, DBP, RR, temp, SpO2
- **Severity scores:** SAPS-II, GCS, Charlson (use MIMIC-IV derived tables if available)
- **Labs (first 24h, first measurement):** WBC, Hb, platelets, creatinine, BUN, Na, K, glucose, anion gap, INR
- **Comorbidities:** COPD, DM, HTN, CKD, CAD (from `diagnoses_icd`)
- **ICU info:** `first_careunit` (CCU, CVICU, MICU, SICU, Other)
- **Treatments (first 24h, binary):** ventilation, vasopressor, diuretic, beta-blocker, anticoagulation

---

## 7. Tooling Decisions

### 7.1 Database / query engine: **DuckDB** (decided)
**Rejected alternatives and why:**
- MSSQL: requires writing ~30 `CREATE TABLE` DDLs, solving `BULK INSERT` encoding/quote issues, building indexes. MIT-LCP has no official MSSQL build script. Would cost ~0.5–1 day of pure ops work.
- PostgreSQL: requires local server setup, not worth it for a solo/pair analysis.
- BigQuery: requires extra PhysioNet access request step + Google Cloud account. Workable but unnecessary when we have local CSVs.
- R/pandas direct CSV read: `chartevents.csv.gz` is ~30 GB, will crash any laptop.

**Why DuckDB wins:**
- Reads `.csv.gz` directly (no decompression, no import step)
- Embedded — no server, installs as R/Python library (`install.packages("duckdb")`)
- Standard SQL — all previously-written queries work unchanged
- Column-store + query optimizer handles 30 GB tables in 8 GB RAM
- DataGrip has native DuckDB support (Vicky keeps her IDE workflow)
- Setup: **~15–30 min vs. MSSQL's 0.5–1 day**

### 7.2 Analysis language: **R** (decided)
- Both team members more comfortable in R than Python
- `duckdb` R package exists; `xgboost` + `SHAPforxgboost` available

### 7.3 Working file structure (recommended)
```
~/mimic-iv/
├── hosp/
│   ├── admissions.csv.gz
│   ├── patients.csv.gz
│   ├── diagnoses_icd.csv.gz
│   ├── d_icd_diagnoses.csv.gz
│   ├── labevents.csv.gz
│   └── prescriptions.csv.gz
├── icu/
│   ├── icustays.csv.gz
│   └── chartevents.csv.gz
└── mimic.duckdb          # persistent DuckDB file we build
```

---

## 8. DuckDB Setup Script (for Vicky to run)

```r
install.packages(c("duckdb", "DBI", "dplyr"))
library(duckdb); library(DBI)

con <- dbConnect(duckdb::duckdb(), dbdir = "~/mimic-iv/mimic.duckdb")

# Small tables → load as TABLE
dbExecute(con, "CREATE TABLE patients AS SELECT * FROM read_csv_auto('~/mimic-iv/hosp/patients.csv.gz')")
dbExecute(con, "CREATE TABLE admissions AS SELECT * FROM read_csv_auto('~/mimic-iv/hosp/admissions.csv.gz')")
dbExecute(con, "CREATE TABLE diagnoses_icd AS SELECT * FROM read_csv_auto('~/mimic-iv/hosp/diagnoses_icd.csv.gz')")
dbExecute(con, "CREATE TABLE d_icd_diagnoses AS SELECT * FROM read_csv_auto('~/mimic-iv/hosp/d_icd_diagnoses.csv.gz')")
dbExecute(con, "CREATE TABLE icustays AS SELECT * FROM read_csv_auto('~/mimic-iv/icu/icustays.csv.gz')")

# Large tables → VIEW only (DuckDB reads on query)
dbExecute(con, "CREATE VIEW labevents AS SELECT * FROM read_csv_auto('~/mimic-iv/hosp/labevents.csv.gz')")
dbExecute(con, "CREATE VIEW chartevents AS SELECT * FROM read_csv_auto('~/mimic-iv/icu/chartevents.csv.gz')")
dbExecute(con, "CREATE VIEW prescriptions AS SELECT * FROM read_csv_auto('~/mimic-iv/hosp/prescriptions.csv.gz')")

dbListTables(con)
```

---

## 9. Sanity-Check SQL Queries (NOT YET RUN — first priority)

These four queries establish the cohort flow. Run before any modeling work. **Report only aggregate counts back — never row-level data.**

### Query 1: broad HF (any HF diagnosis)
```sql
SELECT COUNT(DISTINCT hadm_id) AS n_hf_admissions
FROM diagnoses_icd
WHERE (icd_version = 9 AND (
    icd_code LIKE '428%'
    OR icd_code IN ('40201','40211','40291','40401','40403','40411','40413','40491','40493')
  ))
  OR (icd_version = 10 AND icd_code LIKE 'I50%');
```

### Query 2: acute HF only
```sql
SELECT COUNT(DISTINCT hadm_id) AS n_acute_hf_admissions
FROM diagnoses_icd
WHERE (icd_version = 9 AND icd_code IN (
    '42821','42822','42823','42831','42832','42833','42841','42842','42843'
  ))
  OR (icd_version = 10 AND icd_code IN (
    'I5021','I5022','I5023','I5031','I5032','I5033',
    'I5041','I5042','I5043','I50811','I50812','I50813'
  ));
```

### Query 3: ICU acute HF
```sql
WITH acute_hf AS (
  SELECT DISTINCT hadm_id FROM diagnoses_icd
  WHERE (icd_version = 9 AND icd_code IN (
      '42821','42822','42823','42831','42832','42833','42841','42842','42843'
    ))
    OR (icd_version = 10 AND icd_code IN (
      'I5021','I5022','I5023','I5031','I5032','I5033',
      'I5041','I5042','I5043','I50811','I50812','I50813'
    ))
)
SELECT
  COUNT(DISTINCT icu.hadm_id) AS n_icu_acute_hf,
  COUNT(DISTINCT icu.subject_id) AS n_unique_patients
FROM icustays icu
INNER JOIN acute_hf hf ON icu.hadm_id = hf.hadm_id;
```

### Query 4: ICU acute HF WITH any AF (no prior-AF exclusion yet)
```sql
WITH
acute_hf AS (
  SELECT DISTINCT hadm_id FROM diagnoses_icd
  WHERE (icd_version = 9 AND icd_code IN (
      '42821','42822','42823','42831','42832','42833','42841','42842','42843'
    ))
    OR (icd_version = 10 AND icd_code IN (
      'I5021','I5022','I5023','I5031','I5032','I5033',
      'I5041','I5042','I5043','I50811','I50812','I50813'
    ))
),
icu_hf AS (
  SELECT DISTINCT icu.hadm_id
  FROM icustays icu
  INNER JOIN acute_hf hf ON icu.hadm_id = hf.hadm_id
),
af_any AS (
  SELECT DISTINCT hadm_id FROM diagnoses_icd
  WHERE (icd_version = 9 AND icd_code = '42731')
     OR (icd_version = 10 AND icd_code IN ('I480','I481','I482','I4891'))
)
SELECT
  COUNT(DISTINCT icu_hf.hadm_id) AS n_total,
  COUNT(DISTINCT CASE WHEN af.hadm_id IS NOT NULL THEN icu_hf.hadm_id END) AS n_with_af,
  ROUND(100.0 * COUNT(DISTINCT CASE WHEN af.hadm_id IS NOT NULL THEN icu_hf.hadm_id END)
        / COUNT(DISTINCT icu_hf.hadm_id), 2) AS pct_with_af
FROM icu_hf
LEFT JOIN af_any af ON icu_hf.hadm_id = af.hadm_id;
```

### Expected results (for sanity)
| Query | Expected range |
|---|---|
| Q1 (any HF) | 40,000 – 60,000 |
| Q2 (acute HF) | 15,000 – 25,000 |
| Q3 (ICU acute HF) | 6,000 – 10,000 |
| Q4 (any AF among Q3) | 30–45% of Q3 = 2,000–4,000 |

*Still to come: "final cohort after excluding prior AF" query.*

---

## 10. Statistical Analysis Plan

### 10.1 Descriptive
- Cohort flow diagram (one row per inclusion/exclusion step, with counts)
- Table 1: baseline characteristics by AF status
  - Continuous: mean (SD) or median (IQR)
  - Categorical: n (%)
  - Tests: χ² / Wilcoxon (or t-test where appropriate)

### 10.2 Primary model: multivariable logistic regression
- **Outcome:** in-hospital mortality (binary)
- **Exposure:** hospital-documented AF (binary)
- **Confounders:** selected by DAG, NOT stepwise/AIC
- **Report:** adjusted OR + 95% CI
- **Secondary outcomes:** LOS (use linear regression on log-transformed LOS or negative binomial)

### 10.3 Informatics layer: XGBoost + SHAP
- **Task:** predict hospital-documented AF from first-24h ICU features
- **Strict temporal ordering:** all predictors must precede the AF outcome
- **Validation:** 5-fold cross-validation or train/test split
- **Metrics:** AUC, calibration, Brier score
- **Class imbalance:** use `scale_pos_weight` (DO NOT use SMOTE — controversial in clinical ML)
- **Interpretation:** SHAP global feature importance + individual dependence plots
- **Explicitly NOT causal** — SHAP importance ≠ causal effect

### 10.4 Subgroup: HFrEF vs HFpEF
- Run separately in each subgroup
- **Downgrade to descriptive** if events per subgroup < 500

---

## 11. Outstanding Tasks (Priority Order)

1. **[Vicky] Install DuckDB + build `mimic.duckdb` file** — est. 30 min
2. **[Vicky] Run the 4 sanity-check queries** — est. 15 min
3. **[Vicky → Zihan] Report aggregate counts** (not row-level data) to validate cohort size assumptions
4. **[Zihan] Draw the DAG** — required by professor, currently missing. AF = exposure, mortality = outcome. Need to identify confounders vs mediators (e.g. is vasopressor use a confounder or a mediator?)
5. **[Vicky] Write full cohort extraction SQL** (apply all inclusion/exclusion + prior-AF exclusion)
6. **[Vicky] Extract Table 1 variables**
7. **[Zihan] Start literature matrix** (for introduction)
8. **[Both] Primary analysis → informatics layer → write-up → slides**

---

## 12. Compliance Notes (Important)

- MIMIC data is credentialed (PhysioNet DUA signed)
- **Do NOT paste row-level MIMIC data into any LLM** (ChatGPT, Claude, etc.) — violates DUA
- Aggregate statistics (counts, means, ORs) **are OK** to discuss
- SQL code, schema info, ICD code lists **are OK** — these are not patient data
- Methodology discussions **are OK** — this is how we've been working

---

## 13. Key References

- **Yan et al. 2023** (cohort definition source): https://pmc.ncbi.nlm.nih.gov/articles/PMC10566083/
- **Lubitz SA et al. 2010** — AF in CHF, Heart Fail Clin
- **Rao VN et al. 2025** — AHA Get With The Guidelines AF and HF registries
- **Carlisle MA et al. 2019** — "HF and AF, Like Fire and Fury", JACC: Heart Failure
- **Riley et al. 2020** — sample size for prediction models
- **MIT-LCP mimic-code repo:** https://github.com/MIT-LCP/mimic-code (use their derived concepts for Charlson, SAPS-II, first-day labs/vitals)

---

*Last updated: 2026-04-20 (generated from claude.ai conversation with Zihan)*
