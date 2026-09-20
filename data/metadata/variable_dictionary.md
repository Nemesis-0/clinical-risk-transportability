# Variable Dictionary

This file documents the source variables used by the
clinical-risk transportability analysis and records how each variable is
handled in the frozen hour-6 prediction pipeline.

Variable names and units follow the official
PhysioNet/Computing in Cardiology Challenge 2019 documentation.

Official source:

- PhysioNet Challenge 2019:
  https://physionet.org/content/challenge-2019/1.0.0/
- Dataset DOI: `10.13026/v64v-d857`

The official Challenge release contains 40 input variables:

- 8 vital-sign variables;
- 26 laboratory variables;
- 6 demographic or administrative variables;

plus the outcome column `SepsisLabel`.

The descriptions and units below preserve the official Challenge
conventions. They are not independently harmonized or reinterpreted.


## Project-Specific Conventions

The primary prediction landmark is **ICU hour 6**.

Only information available at or before ICU hour 6 is used to construct
predictors.

Longitudinal variables are retained for the primary prediction
representation when they are observed in at least 10% of eligible
Health System A patients by the landmark.

Five sparse source variables are excluded from the primary model:

- `EtCO2`
- `Bilirubin_direct`
- `Bilirubin_total`
- `TroponinI`
- `Fibrinogen`

The frozen model representation contains 84 predictors:

- 35 vital-sign features;
- 44 laboratory features;
- 3 static numeric features;
- 2 ICU-unit indicator features.

The exact final feature names and column ordering are stored in:

```text
results/frozen/metadata/system_a_model_specification.json
```

The source-level construction rules are stored in:

```text
data/metadata/feature_specification.csv
```


## Status Labels

The tables below use the following project-specific status labels:

- **Included** — contributes directly to the frozen primary model
  representation.
- **Excluded: sparse** — excluded by the System-A-only landmark
  availability rule.
- **Context source** — used to construct a derived model representation
  rather than used directly as a raw model column.
- **Time/index only** — used for temporal alignment, cohort construction,
  or landmark logic, but not as a model predictor.
- **Outcome only** — used to construct the study outcome and never used
  as a predictor.


## Vital Signs

| Variable | Description | Official unit | Primary-model status | Frozen representation |
|---|---|---|---|---|
| `HR` | Heart rate | beats/min | Included | `last`, `mean`, `min`, `max`, `available` |
| `O2Sat` | Pulse oximetry | % | Included | `last`, `mean`, `min`, `max`, `available` |
| `Temp` | Temperature | °C | Included | `last`, `mean`, `min`, `max`, `available` |
| `SBP` | Systolic blood pressure | mm Hg | Included | `last`, `mean`, `min`, `max`, `available` |
| `MAP` | Mean arterial pressure | mm Hg | Included | `last`, `mean`, `min`, `max`, `available` |
| `DBP` | Diastolic blood pressure | mm Hg | Included | `last`, `mean`, `min`, `max`, `available` |
| `Resp` | Respiratory rate | breaths/min | Included | `last`, `mean`, `min`, `max`, `available` |
| `EtCO2` | End-tidal carbon dioxide | mm Hg | Excluded: sparse | None |

For every retained vital sign, the derived model columns follow the
pattern:

```text
<variable>__last
<variable>__mean
<variable>__min
<variable>__max
<variable>__available
```

This produces 35 vital-sign predictors.


## Laboratory Variables

| Variable | Description | Official unit | Primary-model status | Frozen representation |
|---|---|---|---|---|
| `BaseExcess` | Measure of excess bicarbonate | mmol/L | Included | `last`, `available` |
| `HCO3` | Bicarbonate | mmol/L | Included | `last`, `available` |
| `FiO2` | Fraction of inspired oxygen | % | Included | `last`, `available` |
| `pH` | pH | N/A | Included | `last`, `available` |
| `PaCO2` | Arterial carbon-dioxide partial pressure | mm Hg | Included | `last`, `available` |
| `SaO2` | Arterial oxygen saturation | % | Included | `last`, `available` |
| `AST` | Aspartate transaminase | IU/L | Included | `last`, `available` |
| `BUN` | Blood urea nitrogen | mg/dL | Included | `last`, `available` |
| `Alkalinephos` | Alkaline phosphatase | IU/L | Included | `last`, `available` |
| `Calcium` | Calcium | mg/dL | Included | `last`, `available` |
| `Chloride` | Chloride | mmol/L | Included | `last`, `available` |
| `Creatinine` | Creatinine | mg/dL | Included | `last`, `available` |
| `Bilirubin_direct` | Direct bilirubin | mg/dL | Excluded: sparse | None |
| `Glucose` | Serum glucose | mg/dL | Included | `last`, `available` |
| `Lactate` | Lactic acid | mg/dL | Included | `last`, `available` |
| `Magnesium` | Magnesium | mmol/dL | Included | `last`, `available` |
| `Phosphate` | Phosphate | mg/dL | Included | `last`, `available` |
| `Potassium` | Potassium | mmol/L | Included | `last`, `available` |
| `Bilirubin_total` | Total bilirubin | mg/dL | Excluded: sparse | None |
| `TroponinI` | Troponin I | ng/mL | Excluded: sparse | None |
| `Hct` | Hematocrit | % | Included | `last`, `available` |
| `Hgb` | Hemoglobin | g/dL | Included | `last`, `available` |
| `PTT` | Partial thromboplastin time | seconds | Included | `last`, `available` |
| `WBC` | Leukocyte count | count × 10^3/µL | Included | `last`, `available` |
| `Fibrinogen` | Fibrinogen | mg/dL | Excluded: sparse | None |
| `Platelets` | Platelet count | count × 10^3/µL | Included | `last`, `available` |

For every retained laboratory variable, the derived model columns follow
the pattern:

```text
<variable>__last
<variable>__available
```

This produces 44 laboratory predictors.


## Demographic and Administrative Variables

| Variable | Official definition | Primary-model status | Project handling |
|---|---|---|---|
| `Age` | Age in years; the Challenge uses 100 for patients aged 90 or older | Included | Used directly as `Age` |
| `Gender` | Female = 0, Male = 1 | Included | Used directly as `Gender` |
| `Unit1` | Administrative identifier for MICU | Context source | Combined with `Unit2` into derived ICU-unit context |
| `Unit2` | Administrative identifier for SICU | Context source | Combined with `Unit1` into derived ICU-unit context |
| `HospAdmTime` | Hours between hospital admission and ICU admission | Included | Used directly as `HospAdmTime` |
| `ICULOS` | ICU length of stay in hours since ICU admission | Time/index only | Used for longitudinal alignment and landmark/cohort logic; excluded from model predictors |


## ICU-Unit Encoding

The source variables `Unit1` and `Unit2` are converted into a
three-level project-specific ICU-unit representation:

- Unit1;
- Unit2;
- Unknown.

Unknown is the reference state.

The two frozen model columns are:

```text
ICU_unit_Unit1
ICU_unit_Unit2
```

The raw `Unit1` and `Unit2` columns are not retained as separate model
predictors after this encoding.


## Outcome Variable

### `SepsisLabel`

`SepsisLabel` is the Challenge outcome label.

For a patient meeting the Challenge sepsis definition, the label becomes
positive six hours before the reconstructed clinical sepsis-onset time.
For non-sepsis patients, it remains zero.

The present study does **not** use `SepsisLabel` directly as a predictor.

Instead, clinical onset is reconstructed from the first positive label
according to the Challenge labeling convention:

$$
t_{\mathrm{sepsis}}
=
t_{\mathrm{first\ positive\ label}}
+
6.
$$

The primary project outcome is incident reconstructed sepsis onset during
ICU hours $(6,18]$, using information available through ICU hour 6.

Patients whose first available row is already positive are treated as
left-truncated with respect to onset timing and are excluded from the
primary incident-onset analysis.

Patients with reconstructed onset at or before ICU hour 6 are also
excluded from the primary incident-risk cohort.

For patients without an observed event in the prediction window,
follow-up must extend through ICU hour 18 before they can be classified
as negative.


## Missingness and Availability Indicators

The Challenge source files use missing values when a measurement is not
recorded for a given hourly interval.

The project preserves missingness in the patient-level feature matrices.

Each retained longitudinal variable receives an explicit availability
indicator describing whether the variable was observed by the hour-6
landmark.

No global imputation is performed during feature construction.

For the downstream prediction models:

- Ridge logistic regression and elastic-net logistic regression use
  imputation and scaling estimated within training folds;
- LightGBM uses its native missing-value handling.

Availability indicators are part of the frozen primary model
representation.


## Measurement Summaries

All longitudinal summaries use measurements recorded at or before ICU
hour 6.

For retained vital signs:

- `last` is the most recent observed value by the landmark;
- `mean` is the mean of observed pre-landmark values;
- `min` is the minimum observed pre-landmark value;
- `max` is the maximum observed pre-landmark value;
- `available` records whether any value was observed by the landmark.

For retained laboratory variables:

- `last` is the most recent observed value by the landmark;
- `available` records whether any value was observed by the landmark.

Longitudinal slopes are not included in the primary representation.

Measurement counts are also excluded from the primary prediction model
and are reserved for post-validation distribution-shift diagnostics.


## Physiologic Plausibility Cleaning

A small number of clearly implausible physiologic values identified
during the Health System A feature audit are converted to missing using
frozen variable-specific rules.

The same rules are applied unchanged to Health System B.

No cleaning rule is introduced in response to System B model performance
or post-validation distributional findings.


## Calcium Analysis Note

The official Challenge documentation lists `Calcium` in mg/dL.

During the post-validation System A/B distribution-shift analysis,
System B showed a substantial cluster of very low recorded calcium values
that was not observed in System A.

The available data do not establish the origin of this discrepancy.

The project therefore describes it conservatively as **suspected
measurement-definition or scale heterogeneity**.

No specific alternative calcium measurement type, unit conversion, or
laboratory mechanism is assumed, and the finding does not trigger
retrospective modification of the frozen external-validation analysis.


## Frozen Feature Accounting

The primary model representation contains:

| Component | Number of predictors |
|---|---:|
| Vital-sign summaries and availability indicators | 35 |
| Laboratory last values and availability indicators | 44 |
| Static numeric variables | 3 |
| ICU-unit indicators | 2 |
| **Total** | **84** |

The exact 84-column order is treated as frozen and is recorded in
`results/frozen/metadata/system_a_model_specification.json`.

The frozen System A variable set, feature-engineering logic, and column
order are applied unchanged to Health System B.


## Interpretation Notes

This dictionary separates three concepts:

1. the **official Challenge variable definition**;
2. the **project-specific decision** about whether the variable enters the
   primary predictor representation;
3. the **derived model columns** created from retained source variables.

This distinction is important because the final model contains 84
engineered predictors even though the original Challenge source files
contain 40 input variables plus `SepsisLabel`.
