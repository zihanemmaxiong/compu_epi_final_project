# Cohort Construction: Acute HF ICU Patients Without Prior AF (MIMIC-IV v3.1)

**Engine:** DuckDB (in-memory), reading MIMIC-IV `.csv.gz` files directly as views.
**Output:** `stg_final` — one row per patient (index admission), with a binary
`has_af_index` outcome flag.

---

## Overall Pipeline

Each step is materialized as a DuckDB `TEMP TABLE` so that (a) every stage can be
counted independently for the cohort flow diagram and (b) a bad step can be
rebuilt in isolation without rerunning the full pipeline.

```
stg_acute_hf       ← acute HF admissions (ICD filter)
     ↓ join demographics
stg_hf_admits      ← + age, sex, race, discharge info
     ↓ + ICU info
stg_icu_first      ← total ICU LOS per admission
     ↓ apply exclusions
stg_qualifying     ← age≥18 & ICU≥24h & not AMA/hospice & has dischtime
     ↓ first per patient
stg_index          ← earliest qualifying admission per patient
     ↓ drop prior-AF patients
stg_final          ← + has_af_index outcome label
```

---

## Step-by-Step Rationale

### 1. `stg_acute_hf` — Define the exposure universe

```sql
SELECT DISTINCT hadm_id, subject_id
FROM diagnoses_icd
WHERE (icd_version = 9  AND icd_code IN ('42821', ...))
   OR (icd_version = 10 AND icd_code IN ('I5021', ...));
```

- **Start from `diagnoses_icd`, not `admissions`.** `diagnoses_icd` is in long
  format (multiple rows per admission). We `DISTINCT` down to one row per
  `(hadm_id, subject_id)` up front so that downstream joins cannot fan out.
- **Use exact ICD code lists, not `LIKE '428%'`.** Yan et al. 2023 defines
  *acute* HF as only the 18 specific codes (`42821-23 / 31-33 / 41-43` in
  ICD-9 and the corresponding `I50*` in ICD-10). A `LIKE` pattern would pull
  in unspecified and chronic HF, inflating the cohort by roughly 40%.

### 2. `stg_hf_admits` — Attach demographics

```sql
INNER JOIN admissions a USING (hadm_id, subject_id)
INNER JOIN patients   p USING (subject_id)
```

- **`INNER JOIN` on both tables.** An admission without a matching
  `admissions` or `patients` row is incomplete; we drop it rather than carry
  `NULL` demographics forward. In MIMIC-IV this never fires in practice, but
  it is defensive.
- **Age at admission** is computed as
  `anchor_age + EXTRACT(YEAR FROM admittime) - anchor_year`, the standard
  MIMIC-IV formula (anchor_age is the patient's age in `anchor_year`; the
  admission year may differ).

### 3. `stg_icu_first` — Aggregate ICU stays *before* filtering

```sql
SUM(los) AS icu_total_los
GROUP BY subject_id, hadm_id
```

- **Aggregate first, filter later.** DuckDB aggregates CSV scans much faster
  than joins do, and we need a single ICU-LOS value per admission before we
  can apply the ≥24h cutoff.
- **`SUM(los)` across all ICU stays in the admission.** A patient may cycle
  through multiple ICUs (e.g., CCU → step-down → CCU again). The `SUM`
  represents total ICU exposure during the hospitalization. If the protocol
  is revised to require a *single* ICU stay ≥ 24h, change to `MAX(los)` or
  order-by-intime-limit-1 semantics.

### 4. `stg_qualifying` — Apply all "hard" inclusion/exclusion criteria

```sql
WHERE h.age_at_admit >= 18
  AND icu.icu_total_los >= 1.0
  AND h.dischtime IS NOT NULL
  AND (h.discharge_location IS NULL
       OR h.discharge_location NOT IN
           ('AGAINST ADVICE', 'HOSPICE'));
```

- **All conditions are `AND`, so order does not matter** — but they are
  written in a logical reading order: patient → ICU → data completeness →
  clinical exclusions.
- **The `IS NULL OR NOT IN (...)` pattern** is deliberate. In SQL, `NULL NOT
  IN (...)` evaluates to `NULL` (not `TRUE`), which would filter the row
  out. The explicit `IS NULL` branch keeps rows with an unknown
  `discharge_location` instead of silently dropping them. Tighten to plain
  `NOT IN` if the protocol calls for strict exclusion of unknown disposition.
- **Prior-AF exclusion is *not* applied here.** Prior AF depends on the
  index admission's `admittime`, but the index is itself defined as "the
  first *qualifying* admission" — a circular dependency. The resolution is
  to pick qualifying admissions first, then pick the index within them,
  then compute "prior" relative to the index.

### 5. `stg_index` — Pick the first qualifying admission per patient

```sql
ROW_NUMBER() OVER (PARTITION BY subject_id ORDER BY admittime) AS rn
... WHERE rn = 1
```

- **Window function beats `GROUP BY ... MIN()` + self-join.** One pass over
  the data, no join, fast in DuckDB.
- **`ORDER BY admittime`** operationalizes the PECO requirement "first
  qualifying HF admission per patient" as chronologically earliest.

### 6. `stg_prior_af` — The subtle step

```sql
FROM stg_index i
INNER JOIN admissions a2 ON a2.subject_id = i.subject_id
INNER JOIN stg_af_dx af  ON af.hadm_id     = a2.hadm_id
WHERE a2.admittime < i.admittime
```

Three non-obvious choices:

- **Join the full `admissions` table, not `stg_hf_admits`.** A patient may
  have an AF code in an earlier, non-HF hospitalization. That earlier
  admission is absent from `stg_hf_admits`, but it still establishes AF
  history. Using `admissions` as the base preserves the full prior record.
- **Strict inequality `a2.admittime < i.admittime`.** AF coded during the
  index admission itself is the *outcome* of this study, not prior history.
  The strict `<` excludes the index itself from the "prior" lookback.
- **Collapse to `DISTINCT subject_id`.** We exclude *patients*, not
  admissions. One qualifying prior AF in any earlier admission is enough to
  drop the patient, regardless of how many prior admissions they had.

### 7. `stg_final` — Outcome flag and final exclusion

```sql
SELECT i.*,
  CASE WHEN af.hadm_id IS NOT NULL THEN 1 ELSE 0 END AS has_af_index
FROM stg_index i
LEFT JOIN stg_af_dx af ON af.hadm_id = i.hadm_id
WHERE i.subject_id NOT IN (SELECT subject_id FROM stg_prior_af);
```

- **`LEFT JOIN`, not `INNER JOIN`.** The study needs both AF and non-AF
  patients; an inner join would silently drop the comparator group. The
  `CASE` expression turns presence/absence of a matching AF row into a 0/1
  outcome flag.
- **`NOT IN` subquery for the prior-AF exclusion.** DuckDB optimizes this
  as an anti-semi-join; the query planner handles it efficiently without
  manual rewriting.

---

## Two Core Design Principles

### 1. Filter order follows data dependency, not epidemiological narrative

The PECO description reads as "include age ≥ 18 → exclude prior AF →
require ICU ≥ 24h". The SQL order has to be different:

1. Join demographics → needed to compute `age_at_admit`.
2. Aggregate ICU stays → needed to compute `icu_total_los`.
3. Apply age, ICU, and AMA/hospice filters → defines "qualifying".
4. Pick earliest qualifying per patient → defines "index".
5. Compare against all prior admissions → computes "prior AF".

Each step must run after its inputs exist.

### 2. Every stage must be independently countable

The TEMP-TABLE-per-stage pattern is not just for readability; it is what
makes the cohort flow diagram trivial. The flow query is a single
`UNION ALL` of `COUNT(DISTINCT ...)` over each `stg_*` table — no filter
logic is re-executed, and the counts are guaranteed consistent with the
final cohort.

---

## Terminology Note

The exposure is labeled **hospital-documented AF**, not "new-onset AF" or
"incident AF". ICD codes cannot distinguish truly new AF from previously
undiagnosed pre-existing AF that was first coded during the index
hospitalization. This choice is explicit and defended in the methods
(PROJECT_CONTEXT §2.4).

---

## Cohort Flow (MIMIC-IV v3.1 results, 2026-04-20)

| Step | Description | Admissions | Patients |
|---|---|---:|---:|
| 1 | Acute HF admissions (Yan 2023 codes) | 66,008 | 26,391 |
| 2 | Joined with demographics | 66,008 | 26,391 |
| 3 | Age ≥ 18, ICU ≥ 24h, not AMA/hospice, has dischtime | 15,553 | 11,894 |
| 4 | First qualifying admission per patient | 11,894 | 11,894 |
| 5 | No prior AF (final cohort) | 9,463 | 9,463 |

**Outcome counts in final cohort:**
- Hospital-documented AF: **3,776 (39.9%)**
- In-hospital deaths: **1,265 (13.4%)**

Both numbers exceed the ≥ 1,000 events threshold from PROJECT_CONTEXT §4.3,
supporting the full analytic plan (multivariable logistic regression +
XGBoost + SHAP, including HFrEF/HFpEF subgroup analyses).
