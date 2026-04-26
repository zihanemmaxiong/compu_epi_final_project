# Atrial Fibrillation in ICU Heart Failure Patients (MIMIC-IV)

A retrospective cohort study of hospital-documented atrial fibrillation
(AF) and in-hospital outcomes among critically ill patients admitted
for acute heart failure (HF), using MIMIC-IV v3.1.

Harvard clinical epidemiology / health informatics final project,
Spring 2026.

---

## Research Question

**Primary.** Among adult patients admitted to the ICU for acute heart
failure who had no prior atrial fibrillation diagnosis in MIMIC-IV, is
AF documented during the index hospitalization associated with
increased in-hospital mortality, after adjusting for demographic,
clinical, and severity-of-illness factors?

**Secondary.** Does the association differ by HF subtype
(HFrEF vs HFpEF)? How does the AF effect change when extending the
outcome window to 30-day all-cause mortality (Cox regression)?

**Terminology note.** We use *hospital-documented AF* rather than
*new-onset AF* because ICD-based ascertainment cannot reliably
distinguish truly new AF from previously undiagnosed pre-existing AF
(per AHA 2023 Scientific Statement; Chyou et al. *Circulation*).

**Task 3 revision (Apr 2026).** The Task 2 analysis used the ICD code
`I48.x` as the AF exposure with no timestamp, which (1) prevented us
from fixing immortal-time bias in the Cox model and (2) mixed truly
new-onset AF with chronic AF that happened to be coded this admission.
Task 3 (`task3_revised_analysis.ipynb`) replaces this with a
**chartevents-anchored AF exposure** built from nurse-charted heart
rhythm records (`itemid=220048`), giving a precise timestamp for AF
onset. This enables a **time-varying Cox** that anchors AF status at
the moment it first appears, eliminating immortal-time bias. A
**chronic-AF proxy flag** is also built (rhythm AF ≤1 h after ICU
admission OR an AF-specific drug ordered ≤1 h after admittime) to
mark patients likely to have pre-existing AF. The same 9,463 patients
selected by `cohort.Rmd` are kept; no patients are excluded in the
revised primary analysis.

---

## Data Source & Compliance

- **Dataset:** MIMIC-IV v3.1 (not included in this repository)
- **Access:** credentialed via PhysioNet with signed DUA
- **IMPORTANT:** Per the MIMIC DUA, **no row-level patient data**
  (raw CSVs, derived analytic datasets, intermediate query results) may
  be committed to this repository, pasted into any LLM, or shared on
  public platforms. The `.gitignore` excludes all such files.
- Aggregate statistics, ICD code lists, SQL logic, and methodology are
  compliant with the DUA and may be discussed openly.

---

## Repository Structure

```
final_project/
├── README.md                             # this file
├── PROJECT_CONTEXT.md                    # full design doc + decisions
├── cohort_construction.md                # cohort build methodology
├── lit_review_AF_definition_gap.md       # definition debate scan
├── lit_review_methodology_crosstable.md  # cross-paper methods table
├── extract_hf_cohort.Rmd                 # main extraction pipeline
├── feasibility_check.Rmd                 # early feasibility analysis
├── data_desciptive.Rmd                   # descriptive analysis
├── task1_descriptive_analysis.ipynb      # Task 1 — Table 1, distributions
├── task2_statistical_analysis.ipynb      # Task 2 — original logistic + Cox (ICD AF)
├── task3_revised_analysis.ipynb          # Task 3 — chartevents AF, time-varying Cox
├── final_project.Rproj                   # RStudio project file
└── data/                                 # MIMIC analytic dataset
```

---

## Setup

### Prerequisites

- R ≥ 4.1
- RStudio (recommended)
- MIMIC-IV v3.1 CSV files placed in `./data/` (not included)

### R packages

```r
install.packages(c("duckdb", "DBI", "dplyr", "readr",
                   "tableone", "survival", "ggplot2"))
# For the informatics layer:
install.packages(c("xgboost", "SHAPforxgboost", "pROC", "dcurves"))
```

### Database engine

We use DuckDB (embedded, no server required) to read the `.csv.gz`
files directly as views — no import step, no schema DDL, works on
laptops with 8 GB RAM despite the 30 GB chartevents table.

---

## Analytic Pipeline

The extraction and analysis proceed as staged `stg_*` temp tables in
DuckDB, each countable independently for the cohort flow diagram.

### 1. Cohort construction (`extract_hf_cohort.Rmd`)

```
stg_acute_hf       ← acute HF admissions (Yan 2023 ICD codes)
     ↓ join demographics
stg_hf_admits      ← + age, sex, race, discharge info
     ↓ + ICU info
stg_icu_first      ← total ICU LOS per admission
     ↓ apply exclusions
stg_qualifying     ← age ≥ 18, ICU ≥ 24h, not AMA/hospice
     ↓ first per patient
stg_index          ← earliest qualifying admission per patient
     ↓ drop prior-AF patients
stg_final          ← + has_af_index outcome label
```

### 2. Covariate extraction

Additional staging tables join into `stg_analytic_final`:

- `stg_subtype` — HFrEF / HFpEF / mixed flags (ICD-based)
- `stg_comorb` — 10 cardiovascular comorbidity flags
  (HTN, DM, CKD, CAD, COPD, valvular, PAD, stroke history,
  pulmonary HTN, cardiomyopathy)
- `stg_careunit` — first ICU care unit (collapsed to Cardiac /
  Medical / Surgical-Other)
- `stg_labs_24h` — 10 first-24h lab extremes (WBC, Hb, platelets,
  creatinine, BUN, sodium, potassium, glucose, anion gap, RDW)
- `stg_vitals_24h` — 7 first-24h vital extremes (HR, SBP, DBP, MBP,
  RR, SpO2, temperature)
- `stg_final_surv` — 30-day all-cause mortality outcome for Cox

Final merged analytic dataset is exported as `analytic_dataset.csv`
(gitignored).

### 3. Modeling plan

**Primary: multivariable logistic regression** (Task 3 revision)
- Outcome: in-hospital mortality (binary)
- Exposure: `has_af_chartevents` — AF rhythm ever documented in
  `chartevents itemid=220048` during the stay (rhythm-based, replaces
  the ICD-only `has_af_index` used in Task 2)
- Confounders pre-specified from an *a priori* causal DAG (not
  stepwise / AIC)
- Three nested models reported: unadjusted (Model 1), + demographics
  (Model 2), + comorbidities + HF subtype + ICU type (Model 3, fully
  adjusted)
- The Task 2 mediator-adjusted Model 4 (24-h labs/vitals) is
  intentionally **excluded** from the revised primary analysis: those
  variables are downstream of AF and adjusting for them estimates a
  *direct* effect, not the *total* effect we want here

**Primary: time-varying Cox proportional hazards** (Task 3 revision)
- Outcome: 365-day all-cause mortality (time-to-event,
  right-censored at horizon; `patients.dod` for post-discharge deaths)
- Exposure: AF status flips from 0 → 1 at `first_af_time` (the first
  AF rhythm record). Each patient contributes 1 or 2 rows in
  start/stop format — **eliminates immortal-time bias**
- Adjustments mirror logistic Model 3
- Schoenfeld test for AF: p = 0.92 (proportional hazards assumption
  satisfied)

**Sensitivity:**
- S1: exclude `chronic_af_proxy = 1` patients (rhythm AF ≤ 1 h or
  AF drug ≤ 1 h)
- S2: redefine exposure as new-onset AF only
  (`has_af_chartevents=1 AND chronic_af_proxy=0`)
- Both sensitivities use the same Model 3 covariate set as the primary
- Kaplan-Meier landmark analysis at 48 h for visual confirmation
- (Planned, not in Task 3) propensity score / IPTW; E-value for
  unmeasured confounding

### 4. Cohort flow (MIMIC-IV v3.1)

| Step | Admissions | Patients |
|---|---:|---:|
| Acute HF admissions (Yan 2023 codes) | 66,008 | 26,391 |
| Joined with demographics | 66,008 | 26,391 |
| Age ≥ 18, ICU ≥ 24h, not AMA/hospice | 15,553 | 11,894 |
| First qualifying admission per patient | 11,894 | 11,894 |
| No prior AF (final cohort) | **9,463** | **9,463** |

Outcomes in final cohort:
- Hospital-documented AF (ICD `has_af_index`, Task 2): 3,776 (39.9%)
- Chartevents AF (`has_af_chartevents`, Task 3): 3,346 (35.4%)
- Chronic-AF proxy (`chronic_af_proxy`, Task 3 transparency flag): 1,378 (14.6%)
- New-onset AF only (`af_new_onset`, Task 3): 2,072 (21.9%)
- In-hospital deaths: 1,265 (13.4%)
- 365-day all-cause deaths: 3,156 (33.4%)

ICD AF and chartevents AF disagree in 1,764 patients (~19% of cohort) —
1,097 ICD-positive but chartevents-negative (likely chronic AF that
did not manifest this admission; in-hospital mortality 8.6% — close to
the no-AF baseline of 9.6%) and 667 chartevents-positive but ICD-negative
(likely under-coded acute AF; in-hospital mortality 24.6%). This
non-differential misclassification dilutes the Task 2 ICD-based OR.

---

## Final Analytic Dataset — Codebook

73 columns (54 base + 19 added by Task 3), one row per patient =
one index admission. Exported as `data/data.csv`. Grain:
`(subject_id, hadm_id)` is the unique key.

Each table below pairs the **technical column name** (used in code, e.g.
`has_ckd`) with the **display label** (shown in regression result tables
and figures, e.g. *"Chronic kidney disease"*).

### Identifiers & timing

| Variable | Type | Description |
|---|---|---|
| `subject_id` | int | MIMIC patient id |
| `hadm_id` | int | Index hospital admission id |
| `admittime` | datetime | Hospital admission time |
| `dischtime` | datetime | Hospital discharge time |
| `deathtime` | datetime | In-hospital death time (NA if discharged alive) |
| `icu_intime` | datetime | First ICU entry of index admission — **anchor for the 24h window** |
| `icu_outtime` | datetime | Last ICU exit of index admission |
| `icu_total_los` | float | Total ICU length of stay in days (summed across ICU stays within admission) |
| `rn` | int | Row-number from first-qualifying-admission pick; = 1 for all rows in final cohort |

### Demographics

| Variable | Label | Type | Description |
|---|---|---|---|
| `gender` / `female` | Female sex | 0/1 | Female = 1 (derived from `gender == 'F'`) |
| `race` / `race_group` | Race: \<level\> | chr | Collapsed to White / Black / Hispanic / Asian / Other; **White = reference** |
| `insurance` / `insurance_group` | Insurance: \<level\> | chr | Medicare / Medicaid / Private / Other / Missing; **Medicare = reference** |
| `anchor_age` | — | int | Age at `anchor_year` (MIMIC deidentification anchor) |
| `anchor_year` | — | int | Reference year for `anchor_age` |
| `age_at_admit` | Age (per year) | int | Age at index admission = `anchor_age + year(admittime) − anchor_year` |

### Exposure

| Variable | Label | Type | Description |
|---|---|---|---|
| `has_af_index` | Atrial fibrillation | 0/1 | **Hospital-documented AF during index admission** (ICD-9 427.31/427.32, ICD-10 I48.x) — primary exposure |

### HF subtype (from index-admission diagnoses)

Not mutually exclusive. A row with all three = 0 has unspecified HF only (I50.9 / 428.9).

| Variable | Label | Type | Definition |
|---|---|---|---|
| `is_hfref` | HF, reduced EF (HFrEF) | 0/1 | Systolic HF — ICD-10 I50.21/22/23, ICD-9 428.21/22/23 |
| `is_hfpef` | HF, preserved EF (HFpEF) | 0/1 | Diastolic HF — ICD-10 I50.31/32/33, ICD-9 428.31/32/33 |
| `is_hfmixed` | HF, mixed/other | 0/1 | Combined / specified — ICD-10 I50.41/42/43, I50.811/812/813; ICD-9 428.41/42/43 |

### Comorbidities (history-of flags)

All binary 0/1. Defined as **any qualifying ICD code in any admission with `admittime ≤ index admittime`** (so prior admissions count). Source: `diagnoses_icd`.

| Variable | Label | Description |
|---|---|---|
| `has_htn` | Hypertension | Hypertension |
| `has_dm` | Diabetes mellitus | Diabetes mellitus |
| `has_ckd` | Chronic kidney disease | Chronic kidney disease |
| `has_cad` | Coronary artery disease | Coronary artery disease |
| `has_copd` | COPD | Chronic obstructive pulmonary disease |
| `has_valvular` | Valvular disease | Valvular heart disease |
| `has_pad` | Peripheral arterial disease | Peripheral arterial disease |
| `has_stroke_hx` | Stroke / TIA history | Stroke or TIA history |
| `has_pulm_htn` | Pulmonary hypertension | Pulmonary hypertension |
| `has_cardiomyopathy` | Cardiomyopathy | Cardiomyopathy (non-ischemic) |

### ICU care unit (first ICU of index admission)

| Variable | Label | Type | Description |
|---|---|---|---|
| `first_careunit` | — | chr | Raw unit name (CCU, CVICU, MICU, SICU, MICU/SICU, …) |
| `careunit_cat` | ICU unit: \<level\> | chr | Collapsed: **Cardiac** (CCU, CVICU) / **Medical** (MICU, MICU/SICU) / **Surgical/Other** |
| `icu_cardiac` (derived) | Cardiac ICU | 0/1 | Indicator for `careunit_cat == 'Cardiac'` |
| `icu_medical` (derived) | Medical ICU | 0/1 | Indicator for `careunit_cat == 'Medical'` |

### First-24h labs

Extreme value within 24h of `icu_intime`. Source: `labevents` (fluid = 'Blood'). MIMIC default units, no conversion.

| Variable | Label | Units | Extreme | Rationale |
|---|---|---|---|---|
| `wbc_max` | WBC max (K/µL) | K/µL | MAX | inflammation / infection burden |
| `hemoglobin_min` | Hemoglobin min (g/dL) | g/dL | MIN | worst anemia |
| `platelets_min` | Platelets min (K/µL) | K/µL | MIN | worst thrombocytopenia |
| `creatinine_max` | Creatinine max (mg/dL) | mg/dL | MAX | worst renal function |
| `bun_max` | BUN max (mg/dL) | mg/dL | MAX | worst azotemia / volume status |
| `sodium_min` | Sodium min (mEq/L) | mEq/L | MIN | worst hyponatremia (HF prognostic) |
| `potassium_max` | Potassium max (mEq/L) | mEq/L | MAX | worst hyperkalemia (arrhythmia risk) |
| `glucose_max` | Glucose max (mg/dL) | mg/dL | MAX | stress hyperglycemia |
| `anion_gap_max` | Anion gap max | mEq/L | MAX | worst metabolic acidosis |
| `rdw_max` | RDW max (%) | % | MAX | RDW (established HF mortality marker) |

### First-24h vitals

Extreme within 24h of `icu_intime`. Source: `chartevents`. NIBP and arterial-line itemids are pooled for BP. Temperature converted to °C when sourced in °F.

| Variable | Label | Units | Extreme | MIMIC itemids |
|---|---|---|---|---|
| `heart_rate_max` | Heart rate max (bpm) | bpm | MAX | 220045 |
| `sbp_min` | SBP min (mmHg) | mmHg | MIN | 220179, 220050 |
| `dbp_min` | DBP min (mmHg) | mmHg | MIN | 220180, 220051 |
| `mbp_min` | MBP min (mmHg) | mmHg | MIN | 220181, 220052 |
| `rr_max` | RR max (breaths/min) | breaths/min | MAX | 220210 |
| `spo2_min` | SpO2 min (%) | % | MIN | 220277 |
| `temp_max_c` | Temperature max (°C) | °C | MAX | 223761 (°F→°C), 223762 (°C) |

### Outcomes

**Primary** (Section 2.1, logistic regression):

| Variable | Label | Type | Description |
|---|---|---|---|
| `hospital_expire_flag` | In-hospital death | 0/1 | Died before hospital discharge (from `admissions`) |

**Secondary** (Section 2.8 Cox at 3-/6-/12-month horizons):

| Variable | Label | Type | Description |
|---|---|---|---|
| `dod` | — | date | Date of death from `patients.dod`; used to derive multi-horizon mortality |
| `death_30d` | 30-day death | 0/1 | Event flag, died within 30 days of `admittime` (also `death_60d`, `death_90d`, `death_180d`, `death_365d` derived in Section 2.8) |
| `time_to_event_30d` | Time to event 30d | float | Days from `admittime` to event or censoring; capped at 30 (analogues at other horizons) |

### Task 3 additions (chartevents AF, chronic-AF proxy, clean outcomes)

19 columns added by `task3_revised_analysis.ipynb`. All derived from
authoritative timestamps re-pulled from `raw_data/admissions.csv.gz`,
`icustays.csv.gz`, and `patients.csv.gz` (the original
`admittime / icu_intime / dod` columns in this CSV were written by R
with 2-digit year format and parse inconsistently — Task 3 leaves them
intact for backward compatibility but uses the freshly-pulled
timestamps internally for all new derivations).

**Chartevents-anchored AF exposure**

| Variable | Label | Type | Description |
|---|---|---|---|
| `has_af_chartevents` | Atrial fibrillation (chartevents) | 0/1 | AF or atrial flutter rhythm ever recorded in `chartevents` (`itemid=220048`) during the stay — primary exposure in Task 3 |
| `first_af_time` | — | datetime | Timestamp of the first AF/Aflut rhythm record (NA if never) |
| `hrs_to_first_af` | — | float | Hours from `icu_intime` to `first_af_time` |
| `n_af_records` | — | int | Number of AF/Aflut rhythm records during the stay |
| `n_rhythm_records` | — | int | Number of any rhythm records (denominator for monitoring intensity) |

**Chronic-AF proxy (transparency flag, no exclusion)**

| Variable | Label | Type | Description |
|---|---|---|---|
| `af_at_admission` | — | 0/1 | First AF rhythm recorded ≤ 1 h after `icu_intime` (came in already in AF) |
| `af_drug_at_admit` | — | 0/1 | An AF-specific drug (amiodarone / digoxin / sotalol / dofetilide / dronedarone / propafenone / flecainide) ordered ≤ 1 h after `admittime` |
| `chronic_af_proxy` | — | 0/1 | `af_at_admission OR af_drug_at_admit` — likely-chronic AF flag |
| `af_new_onset` | Atrial fibrillation (new-onset only) | 0/1 | `has_af_chartevents=1 AND chronic_af_proxy=0` — used as exposure in Sensitivity 2 |

**Clean outcomes (re-computed from authoritative `deathtime` / `dod`)**

| Variable | Label | Type | Description |
|---|---|---|---|
| `death_inhosp_clean` | — | 0/1 | `deathtime` falls in `[admittime, dischtime]` |
| `days_to_death_clean` | — | float | Days from `icu_intime` to `dod` (NA if alive) |
| `event_30d` `event_90d` `event_180d` `event_365d` | — | 0/1 | Death within horizon h, anchored at `icu_intime` |
| `t2event_30d` `t2event_90d` `t2event_180d` `t2event_365d` | — | float | Time-to-event in days = `min(days_to_death_clean, h)` with a 1-hour floor (replaces the ad-hoc `clip(lower=0.5)` in the Task 2 build) |

---

## Running the pipeline

1. Place MIMIC-IV v3.1 CSVs in `./data/`
2. Open `final_project.Rproj` in RStudio
3. Open `extract_hf_cohort.Rmd` and run chunks in order
4. The final chunk writes `analytic_dataset.csv` to the project root
   (gitignored)

First-time chart/labevents extraction takes 5–15 minutes due to the
size of MIMIC-IV `chartevents.csv.gz` (~30 GB uncompressed).

---

## Current Status

- [x] Cohort construction
- [x] Outcome variables (in-hospital + 30-/90-/180-/365-day mortality)
- [x] Demographics, HF subtype, comorbidities, ICU care unit
- [x] First-24h labs (10 extremes)
- [x] First-24h vitals (7 extremes)
- [x] Merged analytic dataset exported
- [x] Task 1: Table 1 + distributions
- [x] Task 2: original logistic + Cox (ICD AF)
- [x] Task 3: chartevents AF, time-varying Cox, chronic-AF proxy,
      KM landmark, sensitivity analyses
- [ ] Causal DAG drafted (informal version baked into Task 3 covariate selection)
- [ ] Propensity score / IPTW sensitivity (planned)
- [ ] XGBoost + SHAP informatics layer (planned)
- [ ] Paper writing and presentation slides

---

## Team

- **Zihan Xiong** — data extraction, wrangling, literattures, DAG
- **Vicky** — statistical analysis, predictive modelling (if necessary),
  SQL/R implementation

---

## Key References

- Yan M, et al. Development and validation of a prediction model for
  in-hospital death in patients with heart failure and atrial
  fibrillation. *BMC Cardiovasc Disord.* 2023;23:505. (cohort
  definition source)
- Chyou JY, et al. Atrial Fibrillation Occurring During Acute
  Hospitalization: AHA Scientific Statement. *Circulation.*
  2023;147:e676–e698. (terminology authority)
- Persaud P, et al. Validity of ICD-10 codes for atrial
  fibrillation/flutter in critically ill patients with sepsis.
  *Anaesth Crit Care Pain Med.* 2024;43:101398. (ICD PPV 59.1% for
  incident AF)
- Bedford JP, et al. New-onset atrial fibrillation in intensive care:
  epidemiology and outcomes. *Eur Heart J Acute Cardiovasc Care.*
  2022;11:620. (methodological template)
- Zhang HD, et al. Impact of New-Onset Atrial Fibrillation on
  Mortality in Critically Ill Patients. *Clin Epidemiol.*
  2024;16:811–822. (closest MIMIC-IV analog)
- Riley RD, et al. Calculating the sample size required for
  developing a clinical prediction model. *BMJ.* 2020;368:m441.
  (sample size methodology)

See `lit_review_methodology_crosstable.md` for the full cross-paper
methods table.
