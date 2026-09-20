# Clinical Risk Transportability Across Health Systems

## External Validation, Distribution Shift, and Recalibration for Sepsis Risk Prediction

This repository studies how probabilistic clinical risk models behave
when transported from one health-system environment to another.

Using the PhysioNet/Computing in Cardiology Challenge 2019 sepsis
dataset, prediction models are developed and internally validated in
**Health System A**, frozen, and then evaluated without modification in
**Health System B**.

The project focuses on a practical question:

> When a clinical risk model is moved from one health system to another,
> what aspects of predictive reliability are preserved, what aspects
> fail, and how much of that failure can be repaired through simple
> target-system updating?

The analysis emphasizes:

- locked external validation;
- discrimination and probability calibration;
- cross-system measurement and missingness shift;
- health-system domain separability;
- target-system recalibration;
- subgroup sensitivity.


## Study Design

The study deliberately separates model development from external
evaluation.

### Health System A

System A is the sole development environment.

It is used for:

- cohort definition;
- prediction-horizon selection;
- feature specification;
- preprocessing design;
- model selection;
- hyperparameter tuning;
- internal validation;
- final model fitting.

### Health System B

System B is reserved for locked external validation.

Before the System A pipeline is frozen, System B is not used for:

- feature selection;
- feature engineering;
- model selection;
- hyperparameter tuning;
- calibration adjustment;
- performance-driven cohort modification.

Only after the primary external evaluation is complete are
distribution-shift diagnostics, recalibration analyses, and subgroup
sensitivity analyses performed.

This ordering prevents post-validation findings from being used to
retroactively redefine the primary external experiment.


## Prediction Task

Prediction is performed at a fixed landmark:

**ICU hour 6**

Only information available at or before hour 6 is used.

The primary outcome is incident sepsis onset during:

$$
(6,18]
$$

hours after ICU admission.

The target is therefore:

$$
P(
\text{incident sepsis onset during }(6,18]
\mid
\text{information available through hour 6}
).
$$

Patients are eligible only if:

1. ICU hour 6 is observed;
2. sepsis onset is not left truncated;
3. no reconstructed sepsis onset has occurred at or before hour 6;
4. the outcome during the prediction window can be classified from
   observed follow-up.

Patients without an observed event are classified as negative only when
follow-up extends through ICU hour 18.


## Data

The analysis uses the publicly available
[**PhysioNet/Computing in Cardiology Challenge 2019 sepsis dataset**](https://physionet.org/content/challenge-2019/1.0.0/).

Raw cohort sizes:

| Health system | Raw patients |
|---|---:|
| System A | 20,336 |
| System B | 20,000 |

Final analytic cohorts:

| Health system | Eligible patients | Incident events | Event rate |
|---|---:|---:|---:|
| System A | 18,699 | 371 | 1.98% |
| System B | 17,982 | 175 | 0.97% |

Raw Challenge data are not redistributed in this repository.

Users should obtain the original dataset from the
[official PhysioNet Challenge 2019 dataset page](https://physionet.org/content/challenge-2019/1.0.0/)
before reproducing the analysis.


## Feature Representation

Feature design is performed in System A only.

The frozen representation contains **84 predictors** derived from:

- vital-sign summaries;
- laboratory measurements;
- explicit measurement-availability indicators;
- age, gender, and hospital-admission timing;
- ICU-unit context.

Longitudinal variables are retained when they are observed in at least
10% of eligible System A patients by the hour-6 landmark.

The final representation contains:

- 35 vital-sign features;
- 44 laboratory features;
- 3 static numeric features;
- 2 ICU-unit indicators.

Longitudinal slopes and measurement counts are not included in the
primary prediction models.

Measurement counts are used only in later distribution-shift
diagnostics.

Detailed feature definitions are documented in
`docs/methodology.md` and `data/metadata/feature_specification.csv`.


## Models

Three model families are evaluated:

1. **Ridge logistic regression**
2. **Elastic-net logistic regression**
3. **LightGBM**

The objective is not exhaustive algorithm benchmarking.

Instead, the study compares regularized linear models with a nonlinear
tree-based model under a common cohort, feature representation, and
validation framework.


## Internal Validation

System A model development uses nested stratified cross-validation.

Five outer folds generate out-of-fold predictions.

Within each outer training set:

- preprocessing is estimated using training data only;
- hyperparameters are selected using training data only;
- the held-out outer fold is used only for evaluation.

Ridge and elastic-net models use fold-specific imputation and scaling.

LightGBM retains native missing-value handling.

Performance is evaluated using:

- AUROC;
- AUPRC;
- Brier score;
- log loss;
- calibration-in-the-large;
- joint calibration intercept;
- calibration slope.


## Locked External Validation

After internal validation, the full System A pipeline is frozen.

The frozen components include:

- cohort logic;
- feature definitions;
- feature ordering;
- preprocessing behavior;
- physiologic plausibility rules;
- hyperparameters;
- fitted model objects.

The same pipeline is then applied unchanged to System B.

During primary external validation:

- no model is refit;
- no feature is reselected;
- no System B scaling parameter is learned;
- no recalibration is performed;
- no performance-driven cohort modification is made.

System B therefore functions as an external evaluation environment
rather than an additional development dataset.


## Main Results

### Internal Validation

| Model | Brier | Log loss | AUROC | AUPRC | CITL | Joint calibration intercept | Calibration slope |
|---|---:|---:|---:|---:|---:|---:|---:|
| Ridge | 0.01936 | 0.09372 | 0.6845 | 0.0426 | -0.002 | -0.670 | 0.818 |
| Elastic Net | 0.01930 | 0.09325 | 0.6890 | 0.0460 | -0.001 | -0.451 | 0.877 |
| LightGBM | 0.01927 | 0.09277 | 0.6954 | 0.0459 | 0.002 | -0.066 | 0.981 |

Internal discrimination was similar across models.

LightGBM had the highest internal AUROC point estimate and a calibration
slope closest to one.


### External Discrimination

All three models lost discrimination after transport to System B.

| Model | System A AUROC | System B AUROC | Change |
|---|---:|---:|---:|
| Ridge | 0.6845 | 0.6098 | -0.0747 |
| Elastic Net | 0.6890 | 0.6171 | -0.0719 |
| LightGBM | 0.6954 | 0.6538 | -0.0416 |


### External Calibration and Probability Accuracy

| Model | Brier | Log loss | AUROC | AUPRC | CITL | Joint calibration intercept | Calibration slope |
|---|---:|---:|---:|---:|---:|---:|---:|
| Ridge | 0.00988 | 0.05674 | 0.6098 | 0.0156 | -0.600 | -2.364 | 0.546 |
| Elastic Net | 0.00979 | 0.05546 | 0.6171 | 0.0165 | -0.458 | -2.000 | 0.616 |
| LightGBM | 0.00967 | 0.05484 | 0.6538 | 0.0205 | -0.542 | -0.266 | 1.070 |

All three models overpredicted absolute risk in System B.

Mean predicted risk was approximately:

- Ridge: 1.74%;
- Elastic Net: 1.52%;
- LightGBM: 1.66%;

compared with an observed event rate of approximately 0.97%.

The linear models also developed substantial calibration-slope
distortion.

LightGBM retained a slope close to one, suggesting that its relative risk
scale was better preserved despite baseline-risk miscalibration.

Because event prevalence differs substantially between Systems A and B,
raw Brier score, log loss, and AUPRC are not interpreted as direct
transportability measures by comparing their absolute values across
systems.


## Cross-System Distribution Shift

The external performance change occurred alongside substantial
differences in the recorded-data environments.

Among 29 longitudinal variables:

- 22 differed by at least 5 percentage points in availability;
- 15 differed by at least 10 percentage points;
- 9 differed by at least 20 percentage points.

The largest availability shifts included HCO3, BaseExcess, Chloride,
DBP, and FiO2.

Observed-value distributions also differed for multiple variables,
including calcium, oxygenation measures, and blood-pressure variables.

The System B calcium distribution contained a substantial cluster of
very low values not observed in System A.

Because the origin of this discrepancy cannot be established from the
available data, it is treated conservatively as suspected
measurement-definition or scale heterogeneity.

This post-validation finding does not alter the frozen external
analysis.


## Domain Separability

Post-validation domain classifiers quantify how easily Systems A and B
can be distinguished from the recorded data.

| Representation | Cross-validated domain AUROC |
|---|---:|
| Availability indicators only | 0.988 |
| Observed values and context | 0.819 |
| Full frozen representation | 0.994 |
| High-coverage representation | 0.766 |
| Strict high-coverage clinical/basic context | 0.706 |

Measurement availability alone almost completely separated the two
systems.

However, meaningful separation remained even after restricting the
representation to high-coverage clinical variables and basic context.

These analyses characterize observable distribution shift and are not
interpreted as proving a causal mechanism for prediction degradation.


## Target-System Recalibration

After locked external validation, two recalibration strategies are
evaluated without refitting the underlying System A models:

- intercept-only recalibration;
- intercept-and-slope recalibration.

Evaluation uses five-fold stratified cross-fitting within System B.

For each fold, recalibration parameters are estimated on the other four
folds and applied to held-out patients.

### Main Findings

For Ridge and Elastic Net:

- intercept updating corrected baseline-risk mismatch;
- estimating an additional slope provided further Brier-score improvement.

For LightGBM:

- intercept-only updating produced essentially all observed benefit;
- estimating an additional slope provided little additional improvement.

The result is consistent with the external calibration pattern:

- the linear models showed both baseline-risk and risk-scale
  miscalibration;
- LightGBM primarily showed baseline-risk miscalibration.

The recalibrated models have not been independently validated in a third
health system.


## Subgroup Sensitivity

Post-validation subgroup analyses evaluate external performance across:

- age <65 versus age ≥65;
- Gender = 0 versus Gender = 1;
- ICU-unit context: Unit1, Unit2, or Unknown.

Age and gender showed relatively stable discrimination.

The most consistent degradation appeared among patients with unknown
ICU-unit context.

This finding is treated as a care-context sensitivity result rather than
evidence of a causal subgroup effect.


## Interpretation

The analysis supports four main observations.

First, internal validation did not fully describe performance after
cross-health-system transport.

Second, discrimination and calibration failed differently across model
classes.

Third, the target environment differed substantially in how clinical
information was measured and recorded.

Fourth, some external probability miscalibration was repairable without
retraining the full prediction model.

The analyses do **not** establish that measurement-process shift caused
the loss of predictive performance.

Rather, prediction degradation occurred alongside substantial observable
differences between the two recorded-data environments.


## Repository Structure

```text
clinical-risk-transportability/
├── configs/
│   ├── cohort.yaml
│   ├── features.yaml
│   └── models.yaml
│
├── data/
│   ├── metadata/
│   │   ├── feature_specification.csv
│   │   └── variable_dictionary.md
│   ├── processed/
│   │   ├── system_a_features.csv
│   │   └── system_b_features.csv
│   ├── raw/
│   │   └── training/
│   │       ├── training_setA/
│   │       └── training_setB/
│   └── README.md
│
├── docs/
│   ├── manuscript.md
│   ├── methodology.md
│   └── study_protocol.md
│
├── figures/
│   ├── system_a_b_availability_shift.png
│   ├── system_a_b_domain_separability.png
│   ├── system_b_external_calibration.png
│   ├── system_b_recalibration_calibration_curves.png
│   └── system_b_subgroup_auroc_contrasts.png
│
├── notebooks/
│   ├── 01_cohort_audit.ipynb
│   ├── 02_feature_missingness_audit.ipynb
│   ├── 03_system_a_internal_validation.ipynb
│   ├── 04_system_b_external_validation.ipynb
│   ├── 05_distribution_shift.ipynb
│   ├── 06_recalibration.ipynb
│   └── 07_subgroup_sensitivity.ipynb
│
├── results/
│   └── frozen/
│       ├── metadata/
│       ├── models/
│       │   ├── elastic_net_system_a.joblib
│       │   ├── lightgbm_system_a.joblib
│       │   └── ridge_system_a.joblib
│       ├── predictions/
│       └── tables/
│
├── .gitignore
├── README.md
└── requirements.txt
```

Raw data, processed patient-level feature matrices, and patient-level
prediction files are excluded from version control and are regenerated
locally as needed.


## Analysis Workflow

The notebooks preserve the chronological analysis sequence:

```text
01_cohort_audit.ipynb
        ↓
02_feature_missingness_audit.ipynb
        ↓
03_system_a_internal_validation.ipynb
        ↓
04_system_b_external_validation.ipynb
        ↓
05_distribution_shift.ipynb
        ↓
06_recalibration.ipynb
        ↓
07_subgroup_sensitivity.ipynb
```

Conceptually:

```text
System A design
    ↓
System A feature freeze
    ↓
System A internal validation
    ↓
Final model freeze
    ↓
Locked System B external validation
    ↓
Post-validation distribution-shift analysis
    ↓
Target-system recalibration
    ↓
Subgroup sensitivity
```

Later analyses are not used to retroactively modify the locked external
experiment.


## Reproducing the Analysis

### 1. Obtain the Source Data

Obtain the PhysioNet/Computing in Cardiology Challenge 2019 training
data from the
[official PhysioNet Challenge 2019 dataset page](https://physionet.org/content/challenge-2019/1.0.0/).

Place the health-system datasets under:

```text
data/raw/training/training_setA/
data/raw/training/training_setB/
```

Raw data should not be committed to the public repository.


### 2. Create a Python Environment

On macOS or Linux:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

On Windows:

```bash
py -3 -m venv .venv
.venv\Scripts\activate
```

The finalized notebook workflow was executed under **Python 3.13.1**.

Frozen serialized model objects should be loaded using the pinned
dependency versions in `requirements.txt`, because compatibility of
serialized model artifacts is not guaranteed across library versions.

### 3. Install Dependencies

```bash
python -m pip install -r requirements.txt
```


### 4. Run the Notebooks

Execute notebooks `01` through `07` in numerical order.

Finalized outputs are stored under:

```text
results/frozen/
```


## Frozen Artifacts

`results/frozen/` contains finalized analysis artifacts.

### `metadata/`

Frozen model and analysis metadata.

### `models/`

Final fitted System A models.

### `predictions/`

Generated locally during reproduction and excluded from version control.
These patient-level artifacts include:

- System A out-of-fold predictions;
- locked System B external predictions;
- System B cross-fitted recalibrated predictions.

### `tables/`

Final analysis tables for:

- internal validation;
- external validation;
- paired model comparisons;
- distribution shift;
- domain separability;
- recalibration;
- subgroup sensitivity.


## Figures

Publication-oriented figures are stored under `figures/`.

Figures include:

- `system_b_external_calibration.png`
- `system_a_b_availability_shift.png`
- `system_a_b_domain_separability.png`
- `system_b_recalibration_calibration_curves.png`
- `system_b_subgroup_auroc_contrasts.png`


## Documentation

### `docs/study_protocol.md`

Records the study design, frozen analysis decisions, source/target
system roles, and analysis ordering.

It functions as a study protocol and analysis-governance record rather
than a formal prospective preregistration.


### `docs/methodology.md`

Contains the detailed methodological specification for:

- cohort construction;
- feature representation;
- preprocessing;
- model development;
- validation;
- calibration;
- distribution-shift diagnostics;
- recalibration;
- subgroup analysis.


### `docs/manuscript.md`

Contains the publication-oriented manuscript describing the study,
methods, results, interpretation, and limitations.


## Reproducibility Principles

The project follows several analysis-governance rules:

- System B is not used for primary model development.
- Primary feature definitions are frozen before external evaluation.
- Preprocessing is estimated inside training folds during internal
  validation.
- Primary external predictions are generated before post-validation
  analyses.
- Distribution-shift findings do not retroactively modify the frozen
  external experiment.
- Recalibration is separated from primary external validation.
- Subgroup analyses are treated as sensitivity analyses.
- Final outputs are stored separately from exploratory results.


## Limitations

The study evaluates transportability between only two health-system
environments represented in a single public dataset.

Additional limitations include:

- 175 incident events in the external cohort;
- uncertainty in smaller subgroup analyses;
- descriptive rather than causal distribution-shift diagnostics;
- unresolved causes of selected measurement anomalies;
- no independent third-system validation of recalibrated models;
- a deliberately limited set of prediction algorithms;
- use of landmark summaries rather than a full longitudinal sequence
  model.

The objective is transportability characterization rather than
exhaustive model benchmarking.


## Notes on Use

This repository is intended for research and methodological study.

The models and analyses are **not intended for direct clinical deployment
or patient-care decision-making**.

Any deployment-oriented use would require additional external validation,
governance, calibration assessment, and evaluation in the intended
clinical environment.