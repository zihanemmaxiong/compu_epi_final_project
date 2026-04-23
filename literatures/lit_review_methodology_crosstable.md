# Cross-paper methodology table: AF in HF / ICU populations

*Methodology-extraction scan, 2026-04-21. Focus: 13 observational studies
used to inform variable selection and adjustment strategy.*

## TL;DR

Across 13 extracted observational studies (2018–2025), typical cohort
sizes range widely (n~1,300 single-center to n~4.6M nationwide), and most
define AF via ICD-9/10 codes from administrative/EHR data, with only a
handful explicitly separating new-onset vs pre-existing AF. Adjustment
sets converge on **age, sex, HF subtype/LVEF, diabetes, hypertension,
CKD, COPD, CAD/prior MI, and an ICU severity score (SAPS II / APS III /
SOFA / Elixhauser)**; divergence comes from inconsistent inclusion of
lab values (RDW, anion gap, lactate), anticoagulation, and natriuretic
peptides. Effect sizes for AF (new-onset or on admission) on mortality
cluster around adjusted OR/HR 1.1–1.7, with consistently larger effects
in HFpEF than HFrEF. Methods converge on multivariable Cox / logistic
regression + propensity matching; they diverge on (a) whether AF type
(paroxysmal vs persistent, symptomatic vs asymptomatic) is modeled,
(b) how HF subtype is defined (ICD vs LVEF cutoff), and (c) whether
anticoagulation / rate-vs-rhythm control are confounders or mediators.

**For our MIMIC-IV project**, the dominant template is a Cox/logistic
model with age + sex + Charlson/Elixhauser + SAPS II/SOFA + specific
comorbidities + ICU interventions. Our main methodological novelty
should be **restricting to patients without prior AF and using
in-hospital AF timing as the exposure**.

## Paper list (sorted by relevance to our project)

- **Zhang HD 2024, *Clin Epidemiol*** — MIMIC-IV, new-onset vs
  pre-existing AF, 48,018 ICU patients; **closest template to our
  project**.
- **Bedford JP 2022, *Eur Heart J Acute Cardiovasc Care*** — English
  RISK-II ICU database, new-onset AF vs matched controls; in-hospital
  mortality OR 1.50.
- **Zhang/Zuo/Jiao 2024 *Front CVM*** — MIMIC-IV AF cohort, 13,330 ICU
  patients, useful covariate set.
- **Bai Y 2022, *BMC Cardiovasc Disord*** — MIMIC-IV acute HF in ICU,
  predicts AF occurrence; useful for feature selection.
- **Saksena S 2023, *Europace* (TOPCAT post-hoc)** — Propensity-matched
  HFpEF + AF; 23-variable confounder list.
- **Menichelli D 2025, *JAHA* (START registry)** — 10,369 AF patients
  stratified by HF phenotype; PSM.
- **Jobs A 2019, *Clin Res Cardiol*** — Single-center ADHF; AF on
  admission HR differs HFpEF vs HFrEF.
- **Zafrir B 2018, *Eur Heart J* (ESC-HF LT Registry)** — 14,964 HF
  patients, canonical AF × HF-phenotype paper.
- **Kroshian G 2024, *J Cardiovasc Electrophysiol* (meta-analysis)** —
  22 studies, 248k patients; pooled HRs for AF mortality by HF subtype.
- **Goyal P 2018, *Int J Cardiol* (NIS)** — 4.6M acute HF
  hospitalizations; in-hospital mortality by AF × EF.
- **Paula SB 2024, *Cureus*** — Portuguese non-cardiac ICU, 1,357
  patients, PSM; null after matching.
- **Luo J 2021, *Front CVM* (NOAFCAMI-SH)** — AMI without prior AF;
  exact exposure-restriction logic we will use.
- **Suarez-Dono 2023, *Sci Reports* (CHRONIBERIA)** — Internal Medicine
  inpatient cohort.

## Cross-paper extraction table

| # | Citation | Cohort | N | HF def | AF def | Timing | Outcome | Model | Confounders | Effect size (CI) | Key limitation |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Zhang 2024 *Clin Epidemiol* | MIMIC-IV | 48,018 | ICD; any HF | ICD, 3 groups: no/pre-existing/new-onset | First ICU admission | 1-yr mortality | Cox | Age, sex, comorbid, SOFA/SAPS II/APS III/OASIS, labs, vent, vasopressor | NOAF vs no AF HR 1.16 (1.10–1.21) | Single-center; no AF duration |
| 2 | Bedford 2022 *EHJ ACC* | RISK-II (UK ICU + HES) | 4,615 NOAF + 27,690 ctrl | HES ICD prior | New ICD dx during ICU, no prior | Index ICU stay | In-hospital mortality | Logistic + Cox | Age, sex, comorbid, ICNARC severity, admit reason | aOR 1.50 (1.38–1.63) | Spec over sens; no anticoag data |
| 3 | Zhang/Zuo/Jiao 2024 *Front CVM*\* | MIMIC-IV AF | 13,330 | ICD (HF 45%) | ICD-9/10, all AF | ICU admit | 28-d/90-d/1-yr mort | Cox | Demographics, vitals, labs, SAPS II/APS III/OASIS, meds | Aspirin HR 0.64 (0.55–0.74) | Retrospective; no BMI |
| 4 | Bai 2022 *BMC Cardiovasc Disord*\* | MIMIC-IV acute HF ICU | ~2,334 | ICD acute HF | ICD new AF in ICU | Post-admit | AF prediction | Logistic nomogram | Age, HR, hemoglobin, BUN, INR, RDW | AUC 0.73 | Not outcome-focused |
| 5 | Saksena 2023 *Europace* | TOPCAT Americas | 1,765 (584 matched pairs) | LVEF ≥45%, HFpEF | Hx AF or on ECG | Baseline | CV death, HF hosp/prog | Logistic + PSM | 23 vars: demog, comorb, meds | Pump-failure death HR 1.95 (1.05–3.62) | Post-hoc; no AF burden |
| 6 | Menichelli 2025 *JAHA* | START registry (Italy) | 10,369 | Hx hosp/sx + LVEF cut | Documented AF | 720 d f/u | All-cause mortality | Cox + PSM + Fine-Gray | Sex, age, BMI, DOAC, HTN, DM, CAD, anemia, eGFR, cancer, COPD, cirrhosis, smoking, dementia | HFpEF HR 1.49 (1.14–1.94) | No BNP; White pop |
| 7 | Jobs 2019 *Clin Res Cardiol*\* | German single-center | 1,286 | HFrEF/HFpEF clin | Never/hx/on admission | 3-y f/u | All-cause mort | Cox | Pt + Rx characteristics (chart) | AF on admit HR 1.64 (1.32–2.04); HFpEF 2.16 | Single-center; no ICU data |
| 8 | Zafrir 2018 *EHJ* | ESC-HFA LT Registry | 14,964 | LVEF phenotype | Documented AF | 2.2 y | Death + HF hosp | Multivariable Cox | Age, sex, comorbid, meds, NYHA | HFpEF HR 1.37 (1.15–1.62); HFrEF null | AF type not distinguished |
| 9 | Kroshian 2024 *J Cardiovasc EP* (meta) | 22 studies | 248,323 | HFrEF/mrEF/pEF | Mixed | Variable | All-cause mort | Random-effects HR pool | Study-level | Overall HR 1.13 (1.07–1.21); HFpEF 1.16 | Heterogeneity |
| 10 | Goyal 2018 *Int J Cardiol* | NIS (US) | 4,641,890 | ICD-9 acute HFpEF/rEF | ICD-9 427.31 | Hospitalization | In-hospital mort | Logistic | Age, sex, race, payer, hosp char, Elixhauser | HFpEF OR 1.10 (1.08–1.11); HFrEF 0.93 | Admin coding; no echo |
| 11 | Paula 2024 *Cureus*\* | Portuguese non-card ICU | 1,357 | Comorbid | Chronic or new ICU AF | 54-mo admits | Hosp / 1-y / 2-y mort | Logistic + PSM | Age, SAPS II, HF, renal, COPD, DM, HTN, sex, sepsis | OR 1.10 (0.72–1.66) null | Single-center; small |
| 12 | Luo 2021 *Front CVM* (NOAFCAMI-SH) | AMI single-center | 2,399 (278 NOAF) | n/a (AMI) | NOAF ≥30s during hosp | Index AMI | All-cause mort | Cox + PSM + IPW | GRACE, age, sex, SBP, HR, Killip, LVEF, meds | Asx NOAF HR 1.61 (1.09–2.37) | Modest events; low OAC |
| 13 | Suarez-Dono 2023 *Sci Reports* | Iberian Internal Med | 1,406 | Not primary | Documented AF (any) | 1-y post-DC | 1-y mortality | Multivariable logistic | Age, sex, neoplasia, Barthel | OR 1.49 (1.17–1.91) | AF type not distinguished |

\* Full text accessed. All other rows had full-text-level extraction
unless noted.

## Variable selection recommendations for our project

### Must include (≥ 3 papers, strong evidence)

- Age
- Sex
- Hypertension
- Diabetes mellitus
- Chronic kidney disease / eGFR
- Coronary artery disease / prior MI
- COPD / chronic lung disease
- Charlson / Elixhauser comorbidity index
- ICU severity score (**SAPS II strongly preferred**; SOFA and APS III
  also common)
- HF subtype / LVEF (HFpEF vs HFrEF vs HFmrEF)
- Beta-blocker use
- Anticoagulation status

### Optional / DAG-driven (1–2 papers)

- RDW, anion gap, BUN, creatinine, lactate
- Mechanical ventilation, vasopressor/inotrope use
- Antiarrhythmic drugs
- BMI, smoking
- Admission type (medical vs surgical), reason for ICU admission
- Anemia / hemoglobin, INR
- Barthel index / functional status
- Sepsis on admission

### Rarely measured but worth discussing

- **Natriuretic peptides (BNP/NT-proBNP)** — flagged as missing in
  START, TOPCAT, MIMIC studies
- AF burden / duration
- Echocardiographic metrics beyond LVEF (LA volume, diastolic
  dysfunction)
- Rhythm-vs-rate control strategy (**likely mediator, not confounder**)
- GRACE / CHA2DS2-VASc scores

## HF subtype handling across papers

HF subtype is the single most consistent effect-modifier: meta-analysis
(Kroshian 2024), ESC-HFA registry (Zafrir 2018), NIS (Goyal 2018), START
(Menichelli 2025), and Jobs 2019 all find AF is associated with higher
mortality in HFpEF but null or attenuated in HFrEF. Subtype is defined
by LVEF threshold (typically <40 / 40–49 / ≥50) in cohort studies and
by ICD codes in administrative datasets (ICD-9 428.31/428.33).

**Implication for our project**: MIMIC-IV does not always have LVEF, so
we should expect ICD-based subtype and acknowledge misclassification —
or restrict sensitivity analyses to patients with recorded echo.

## Exposure definition patterns

Most papers rely on ICD codes without timestamping, conflating prevalent
and incident AF. The high-quality exceptions — **Zhang 2024 (MIMIC-IV),
Bedford 2022 (RISK-II), Luo 2021 (NOAFCAMI-SH)** — explicitly restrict
to patients with no prior AF diagnosis and define exposure as first
documented AF during the index hospitalization. This is the design we
should replicate. The 30-second ECG duration threshold (Luo 2021) is
the only clinically rigorous definition we saw; administrative cohorts
accept any ICD code with no duration criterion.

## Biggest methodological red flags

- Most administrative cohorts cannot separate new-onset from
  pre-existing AF without explicit chart logic; "any AF ICD code" biases
  toward the null.
- LVEF is frequently missing in ICU/administrative cohorts, forcing HF
  subtype by ICD — a known noisy proxy.
- Anticoagulation status and rhythm/rate-control strategy are often
  either ignored or adjusted for, even though they may be on the causal
  pathway (**mediator bias**).
- Single-center ICU studies report wildly different effect sizes (null
  to HR 2.16), suggesting unmeasured confounding and selection
  heterogeneity.
- Few studies distinguish symptomatic vs asymptomatic NOAF despite
  evidence it matters (Luo 2021).

## Strongest 3 papers to read in full

1. **Zhang HD et al. 2024, *Clin Epidemiol*** — direct methodological
   template: MIMIC-IV, explicit no-AF / pre-existing / new-onset
   trichotomy, Cox with ICU severity scores. **Read first.**
2. **Bedford JP et al. 2022, *EHJ ACC*** — cleanest exposure definition
   (HES + CMP linkage, new dx only), logistic for in-hospital mortality
   + matched controls; use as the reporting template.
3. **Saksena S et al. 2023, *Europace*** — exemplary 23-variable
   propensity-score confounder list; use as checklist for what we
   should ideally adjust for.
