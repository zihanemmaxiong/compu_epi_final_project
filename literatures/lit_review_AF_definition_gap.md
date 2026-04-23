# Literature Scoping: new-onset vs hospital-documented AF — definition gap

*Scoping scan, 2026-04-21. Focus: how the literature handles the ambiguity between truly new-onset AF and previously undiagnosed pre-existing AF when using ICD codes.*

## TL;DR

The field has moved sharply, in the last 5 years, away from treating
hospital-documented AF as either a clean "incident" event or a dismissible
"transient/secondary" episode — the 2023 AHA Scientific Statement (Chyou
et al.) explicitly recommends abandoning the term "secondary AF" and uses
"AF occurring during acute hospitalization" as an umbrella that deliberately
does not resolve the new-onset vs previously-undiagnosed ambiguity.
ICD-10 ascertainment of AF itself is reasonably good (pooled PPV ~88%,
sensitivity ~80%; meta-analysis by Jiang et al. 2020), but ICD-based
separation of *incident* from *prevalent* AF is substantially weaker, with
the best recent validation in critically ill sepsis patients (Persaud et
al. 2024) reporting PPV only 59.1% for "incident" AF even when codes were
dated during the index admission. The dominant current approaches are
(a) lookback-period + first-ever-code definitions with known
misclassification (Chamberlain et al. 2022), (b) POA-flag-based algorithms
with acknowledged documentation artifacts, and (c) chart-review gold-standard
validation in subsamples. The literature converges on three points:
a non-trivial minority of "new-onset" AF in acute-care databases is really
previously undiagnosed paroxysmal AF; much of the rest recurs after
discharge (Lubitz 2015, Walkey 2014, Sibley 2025); and effect-size estimates
for "new-onset AF" are definition-sensitive but **no paper has cleanly
quantified how much the estimate shifts across definitions in the same ICU
cohort** — that is the open gap this project sits in.

## Papers (grouped by theme)

### Theme 1: Definition & terminology papers

**Chyou JY, Barkoudah E, Dukes JW, et al. Atrial Fibrillation Occurring
During Acute Hospitalization: A Scientific Statement From the AHA.
*Circulation* 2023;147(15):e676–e698. PMID 36912134.**

- AHA scientific statement (consensus).
- Formally deprecates "secondary AF" because it prejudges etiology and
  reversibility; reframes as "acute AF" / "AF occurring during acute
  hospitalization," explicitly acknowledging that first detection during
  hospitalization cannot reliably distinguish truly new AF from
  previously undiagnosed AF.
- *Why it matters:* single most authoritative document justifying our
  phrasing "hospital-documented AF" over "new-onset AF."

**2023 ACC/AHA/ACCP/HRS Guideline for Diagnosis and Management of AF
(Joglar JA et al.). *Circulation* 2023;149:e1–e156. PMID 38033089.**

- Introduces 4-stage AF framework (at-risk, pre-AF, AF with subtypes
  3A–3D, permanent); paroxysmal = ≥30 sec but terminating within 7 days.
- *Why it matters:* grounds our choice of paroxysmal vs persistent
  cut-offs; supports treating "new diagnosis" as a stage-transition event
  rather than a binary exposure.

**Noseworthy PA, Kaufman ES, Chen LY, et al. Subclinical and
Device-Detected Atrial Fibrillation: Pondering the Knowledge Gap.
*Circulation* 2019;140:e944–e963. PMID 31694402.** *(foundational)*

- Establishes that much AF is subclinical/asymptomatic and detected only
  on prolonged monitoring; implies any hospitalization-based detection
  systematically undercounts pre-existing AF.
- *Why it matters:* directly supports the claim that MIMIC "no prior AF"
  patients include an unknown fraction with previously undiagnosed
  paroxysmal AF.

**Sibley S, et al. Atrial fibrillation in critical illness: state of the
art. *Intensive Care Medicine* 2025;51(5):904–916. PMID 40323451.**

- Defines NOAF in ICU as "AF in patients with no prior history," flagging
  that "no prior history" is defined by records, not rhythm; highlights
  ~50% 1-year AF recurrence post-discharge.
- *Why it matters:* most current ICU-specific framing of the definition
  problem.

### Theme 2: POA flag / coding validation papers

**Persaud P, Rudoni MA, Duggal A, et al. Validity of ICD-10 codes for
atrial fibrillation/flutter in critically ill patients with sepsis.
*Anaesth Crit Care Pain Med* 2024;43(4):101398. PMID 38821159.**

- Validation (chart review gold standard, n=400 adjudicated from 6,554
  ICU sepsis admits).
- ICD-10 excellent for *any* AF/AFL (PPV 99.1%, Sens 93.0%) but poor at
  separating incident from prevalent — **for "incident" (code dated
  during index admission) PPV was only 59.1%**. Among 44 patients coded
  as incident, 18 had prior documentation of AF.
- *Why it matters:* directly quantifies the misclassification our
  MIMIC analysis inherits. Strongest empirical number for our
  limitations section.

**Chamberlain AM, Roger VL, Noseworthy PA, et al. Identification of
Incident Atrial Fibrillation From Electronic Medical Records.
*J Am Heart Assoc* 2022;11(7):e023237. PMID 35348008.**

- Olmsted County validation, n=6,177 ICD-9 + 200 ICD-10.
- Algorithm choice changes PPV by ~7 points and sensitivity by >20
  points. Best balance: 1 inpatient OR 2 outpatient codes >7 days apart
  within 1 year (PPV 69.9%, Sens 95.5%). Single-code algorithm PPV
  64.4%.
- *Why it matters:* canonical reference for algorithm-dependent
  misclassification magnitude.

**Jiang L, et al. Sensitivity, specificity, PPV and NPV of identifying AF
using administrative data: systematic review and meta-analysis.
*Clin Epidemiol* 2020;11:941–947. PMID 31933524.**

- Meta-analysis (24 studies). Pooled Sens 80%, Spec 98%, PPV 88%, NPV
  97% — but only 3 studies used rhythm monitoring as gold standard.
- *Why it matters:* the pooled number most commonly cited to defend
  ICD-based AF ascertainment. Use with caveats (validation-against-chart
  circularity).

**Jensen PN, Johnson K, Floyd J, Heckbert SR, Carnahan R, Dublin S.
Systematic review of validated methods for identifying AF using
administrative data. *Pharmacoepidemiol Drug Saf* 2012;21(Suppl 1):141–
147.** *(foundational)*

- 16 studies. PPV for ICD-9 427.31 = 70–96% (median 89%); sensitivity
  57–95% (median 79%). Flagged the field-wide absence of rigorous
  incidence (as opposed to prevalence) validation.
- *Why it matters:* historical anchor showing the incident-vs-prevalent
  gap has been known and unresolved for >13 years.

### Theme 3: Impact of definition choice on downstream findings

**Lubitz SA, Yin X, Rienstra M, et al. Long-Term Outcomes of Secondary
Atrial Fibrillation in the Community: The Framingham Heart Study.
*Circulation* 2015;131(19):1648–1655. PMID 25769640.** *(landmark)*

- Community cohort with physician-adjudicated AF.
- AF with a "secondary precipitant" (surgery, sepsis, AMI, etc.) recurs
  in 42%/56%/62% at 5/10/15 yrs — it is NOT a distinct benign entity.
  Stroke and mortality risks were comparable to primary AF.
- *Why it matters:* empirical basis for the 2023 AHA rejection of
  "secondary AF." Supports our framing that even truly new-onset AF
  carries prognostic weight.

**Walkey AJ, Hammill BG, Curtis LH, Benjamin EJ. Long-term Outcomes
Following Development of New-Onset AF During Sepsis. *Chest*
2014;146(5):1187–1195.** *(landmark)*

- Medicare 5% claims cohort (138,722 sepsis survivors).
- Among sepsis patients with first-diagnosed AF during hospitalization,
  AF recurs in ~55% at 5 years vs 15.5% in non-AF sepsis — strong
  indirect evidence that a large share of "new-onset" AF is really
  re-emergent paroxysmal AF.
- *Why it matters:* parallel exposure definition to ours; the
  ~40-percentage-point gap in recurrence is the practical magnitude of
  what "first coded" captures.

**Zhang HD, et al. Impact of New-Onset Atrial Fibrillation on Mortality
in Critically Ill Patients. *Clin Epidemiol* 2024;16:811–822.
PMID 39588013.**

- MIMIC-IV cohort study (n≈48,000). NOAF 24.1%, pre-existing AF 10.2%,
  no AF 65.7%. 1-yr mortality 34.0% (NOAF) vs 28.5% (pre-existing) vs
  20.7% (no AF). Explicitly distinguishes three groups.
- *Why it matters:* **closest methodologic analog to our project** —
  effectively doing what we are doing, on the same dataset. Journal
  is mid-tier (Dove Press) but method transferability is high.
  **This paper must be read before finalizing our framing.**

## Cross-paper synthesis

**Agreed:**

- ICD-10 has excellent PPV for *any* AF but poor specificity in
  distinguishing incident from prevalent.
- Hospital-documented AF recurs in the long term in a large fraction —
  it is not "transient."
- The term "secondary AF" should be abandoned (AHA 2023).
- Short in-hospital rhythm observation systematically under-ascertains
  pre-existing paroxysmal AF.

**Disagreed / unresolved:**

- Whether "new-onset AF during acute illness" should be treated as a
  distinct entity, a marker of future AF, or simply an earlier diagnosis
  of lifelong AF.
- Whether anticoagulation after secondary/acute AF reduces stroke
  (Walkey 2022 found no benefit; RCTs ongoing).

**Silent / open gap:**

- **No paper quantifies how effect-size estimates for in-hospital
  outcomes shift when the exposure definition changes** (e.g.,
  "first-ever ICD code" vs "POA-flag negative and first code" vs
  "chart-confirmed new AF"). This is the explicit methodological gap
  this project can probe via sensitivity analysis.
- No validated POA-flag algorithm specifically for AF in MIMIC-IV.

## Strongest 3 papers to read in full

1. **Chyou et al., *Circulation* 2023 (AHA Statement, PMID 36912134)** —
   mandatory; our terminology framework hangs on it.
2. **Persaud et al., *ACCPM* 2024 (PMID 38821159)** — the empirical
   PPV 59.1% for incident AF in ICU is the single most quotable number
   for our limitations section.
3. **Zhang HD et al., *Clin Epidemiol* 2024 (PMID 39588013)** — closest
   prior work in MIMIC-IV; directly affects our novelty positioning.

## Red flags / biases from this literature

- **Validation-against-chart circularity**: most PPV/Sens/Spec estimates
  use chart review as gold standard, but chart documentation often
  derives from the same ECG that triggered the ICD code.
- **Surveillance bias**: ICU patients have continuous telemetry; ward
  patients do not. "New-onset AF" in MIMIC-IV ICU is over-ascertained
  relative to the general hospital population.
- **Lookback bias**: "no prior AF" is operationally "no AF coded in the
  linked priors," which conflates truly AF-naïve patients with
  previously undiagnosed paroxysmal AF.
- **Language drift**: older papers use "secondary AF" and "new-onset AF"
  interchangeably; the 2023 AHA deprecation means older effect estimates
  may not map cleanly onto current classifications.

*11 papers total (5 in 5-yr window, 4 foundational/landmark, 2
borderline). Abstracts-only flagged in each entry where applicable.*
