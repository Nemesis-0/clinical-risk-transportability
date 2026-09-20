# Data Directory

This directory contains source-data locations, derived analytic datasets,
and metadata used by the clinical-risk transportability analysis.

The project uses the publicly available
**PhysioNet/Computing in Cardiology Challenge 2019 sepsis dataset**.

Raw Challenge data are not redistributed by this repository.


## Directory Structure

```text
data/
├── metadata/
│   ├── feature_specification.csv
│   └── variable_dictionary.md
├── processed/
│   ├── system_a_features.csv
│   └── system_b_features.csv
├── raw/
│   └── training/
│       ├── training_setA/
│       └── training_setB/
└── README.md
```

Raw source files and patient-level processed feature matrices are
excluded from version control and are regenerated locally as needed.


## `raw/`

The `raw/` directory contains the original source data obtained from the
PhysioNet/Computing in Cardiology Challenge 2019 release.

Expected local structure:

```text
data/raw/training/
├── training_setA/
└── training_setB/
```

The two training sets are treated as separate health-system
environments:

- `training_setA/` → **Health System A**
- `training_setB/` → **Health System B**

Health System A is the development environment.

Health System B is reserved for locked external validation.


### Raw-Data Policy

Raw Challenge files should be obtained directly from the official data
source.

They should not be redistributed through this repository.

The original downloaded files should remain unchanged.

All cohort construction, cleaning, feature engineering, and downstream
analysis are performed separately so that the source data remain
auditable.


## `processed/`

The `processed/` directory contains patient-level feature matrices
constructed from the raw longitudinal records.

Expected local processed datasets are:

```text
data/processed/system_a_features.csv
data/processed/system_b_features.csv
```

These patient-level files are excluded from version control and can be
reconstructed from the source data using the analysis notebooks.


### `system_a_features.csv`

This file contains the frozen landmark representation for eligible
Health System A patients.

System A is used for:

- feature design;
- internal validation;
- model development;
- final model fitting.

The file contains:

- `patient_id`;
- `outcome`;
- 84 frozen model predictors.

The resulting analytic table therefore contains 86 columns when the
identifier and outcome are included.


### `system_b_features.csv`

This file contains the corresponding frozen landmark representation for
eligible Health System B patients.

The System A feature specification is applied unchanged to System B.

No predictor is added, removed, or redefined in response to System B
model performance.

The file contains:

- `patient_id`;
- `outcome`;
- the same 84 frozen model predictors used in System A.


## Prediction Landmark and Outcome

Both processed feature matrices correspond to the same fixed prediction
landmark:

**ICU hour 6**

Only information available at or before ICU hour 6 is used to construct
predictors.

The primary outcome is incident sepsis onset during ICU hours
$(6,18]$.

Patients with left-truncated onset timing are excluded from the primary
incident-onset analysis.

Patients with reconstructed onset at or before the landmark are also
excluded.

Patients without an observed event are classified as negative only when
follow-up extends through ICU hour 18.


## Feature Representation

The frozen representation contains 84 predictors derived from:

- vital signs;
- laboratory measurements;
- explicit measurement-availability indicators;
- static demographic variables;
- hospital-admission timing;
- ICU-unit context.

The final representation contains:

- 35 vital-sign features;
- 44 laboratory features;
- 3 static numeric features;
- 2 ICU-unit indicators.


### Vital Signs

Seven vital-sign variables are retained:

- `HR`
- `O2Sat`
- `Temp`
- `SBP`
- `MAP`
- `DBP`
- `Resp`

For each retained vital sign, the representation includes:

- last observed value;
- mean;
- minimum;
- maximum;
- availability indicator.

This produces 35 vital-sign features.


### Laboratory Variables

Twenty-two laboratory variables are retained:

- `BaseExcess`
- `HCO3`
- `FiO2`
- `pH`
- `PaCO2`
- `SaO2`
- `AST`
- `BUN`
- `Alkalinephos`
- `Calcium`
- `Chloride`
- `Creatinine`
- `Glucose`
- `Lactate`
- `Magnesium`
- `Phosphate`
- `Potassium`
- `Hct`
- `Hgb`
- `PTT`
- `WBC`
- `Platelets`

For each retained laboratory variable, the representation includes:

- last observed value;
- availability indicator.

This produces 44 laboratory features.


### Static Numeric Variables

Static numeric predictors are:

- `Age`
- `Gender`
- `HospAdmTime`

These contribute three model features.


### ICU-Unit Context

The original `Unit1` and `Unit2` fields are converted into a three-level
ICU-unit concept:

- Unit1;
- Unit2;
- Unknown.

Unknown is treated as the reference state.

The model representation contains two indicator columns:

- `ICU_unit_Unit1`
- `ICU_unit_Unit2`


## Feature Selection

Feature design is performed using Health System A only.

The initial candidate set contains 39 clinical and demographic variables
after excluding the outcome label and `ICULOS`.

Longitudinal variables are retained when they are observed in at least
10% of eligible System A patients by the hour-6 landmark.

Five highly sparse variables are excluded from the primary prediction
representation:

- `Bilirubin_total`
- `Fibrinogen`
- `TroponinI`
- `Bilirubin_direct`
- `EtCO2`

Longitudinal slopes and measurement counts are not included in the
primary prediction models.

Measurement counts are reserved for post-validation distribution-shift
diagnostics.


## Missingness

Missingness is retained as part of the clinical data representation.

Each retained longitudinal variable includes an explicit availability
indicator.

No global imputation is performed when the processed feature matrices
are created.

For downstream modeling:

- Ridge and elastic-net logistic regression use imputation and scaling
  estimated within training folds;
- LightGBM retains native missing-value handling.

This prevents preprocessing information from leaking across validation
boundaries.


## Physiologic Plausibility Cleaning

A small number of clearly implausible physiologic measurements identified
during the System A feature audit are converted to missing using frozen
variable-specific rules.

The same rules are applied unchanged to System B.

No new cleaning rule is introduced in response to System B prediction
performance or post-validation distributional findings.


## `metadata/`

The `metadata/` directory contains documentation required to interpret
the processed feature matrices.


### `feature_specification.csv`

Contains the frozen feature specification used to construct the model
matrix.

It documents the source variables and feature-construction rules
underlying the frozen 84-column representation.


### `variable_dictionary.md`

Provides human-readable descriptions of source variables and relevant
analysis conventions.

This file complements the machine-readable feature specification and
the detailed methodology documentation.


## Data Lineage

The primary data flow is:

```text
PhysioNet / CinC 2019 raw data
        ↓
data/raw/training/training_setA/
        ↓
System A cohort and feature audit
        ↓
data/processed/system_a_features.csv
        ↓
System A internal validation
        ↓
Final System A model freeze
        ↓
data/raw/training/training_setB/
        ↓
Frozen cohort and feature logic applied unchanged
        ↓
data/processed/system_b_features.csv
        ↓
Locked external validation
        ↓
Post-validation distribution-shift analysis
        ↓
Target-system recalibration
        ↓
Subgroup sensitivity
```

Patient-level prediction artifacts generated from these datasets are
stored locally under:

```text
results/frozen/predictions/
```

Final analysis tables are stored under:

```text
results/frozen/tables/
```

Patient-level prediction files are excluded from version control.


## Analysis Governance

The data workflow follows several separation rules:

1. Health System A is used for cohort and feature development.
2. Health System B is not used to select or redefine primary predictors.
3. The System A feature specification is frozen before primary System B
   model evaluation.
4. The frozen System A feature construction and cleaning rules are
   applied unchanged to System B.
5. System B distribution-shift findings do not retroactively alter the
   processed data used for locked external validation.
6. Recalibration is treated as a separate post-validation model-updating
   analysis.
7. Subgroup analyses use the original locked System B predictions.

These rules preserve the distinction between development, external
validation, and post-validation investigation.


## Rebuilding Processed Data

The processed datasets are generated through the chronological notebook
workflow.

The most directly relevant notebooks are:

```text
notebooks/01_cohort_audit.ipynb
notebooks/02_feature_missingness_audit.ipynb
notebooks/04_system_b_external_validation.ipynb
```

`01_cohort_audit.ipynb` defines the primary landmark cohort, reconstructed
sepsis-onset logic, outcome horizon, and outcome-observability rules.

`02_feature_missingness_audit.ipynb` defines and freezes the System A
feature representation and generates the System A processed feature
matrix.

`04_system_b_external_validation.ipynb` applies the frozen cohort and
feature logic to Health System B before locked external prediction.

Additional methodological details are documented in:

```text
docs/study_protocol.md
docs/methodology.md
```


## Version-Control Boundaries

Files under `data/` represent source or derived analytic data.

Raw and patient-level processed data are intentionally excluded from the
public Git repository.

They are conceptually separate from:

```text
results/frozen/
```

which contains frozen model objects, analysis metadata, summary tables,
and a local `predictions/` subdirectory. Patient-level prediction artifacts
are generated locally and excluded from public version control.

These files are also conceptually separate from:

```text
figures/
```

which contains publication-oriented visualizations.

This separation helps preserve reproducibility while avoiding
redistribution of source or derived patient-level data.