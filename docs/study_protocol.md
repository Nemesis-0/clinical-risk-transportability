# Study Protocol

## 1. Purpose and Status

This document records the study design, primary analysis plan, and
analysis-ordering rules used in the clinical-risk transportability
project.

It is intended as a protocol and analysis-governance record rather than
as a formal prospective preregistration.

The central design principle is separation between:

1. model development in Health System A;
2. locked external validation in Health System B;
3. post-validation diagnostic and model-updating analyses.

Later findings are not used to retroactively modify the primary
external-validation experiment.


## 2. Research Question

The primary research question is:

> How well do probabilistic sepsis risk models developed in one health
> system transport to a different health system?

The study additionally asks:

- how discrimination and probability calibration change across systems;
- what observable forms of distribution shift accompany external
  performance degradation;
- whether simple target-system recalibration can repair probability
  reliability without refitting the underlying prediction models;
- whether external performance varies across selected demographic or
  care-context subgroups.


## 3. Data Source and System Roles

The study uses the publicly available PhysioNet/Computing in Cardiology
Challenge 2019 sepsis dataset.

The dataset contains records from two health systems.

### Health System A

Health System A is the sole development environment.

It is used for:

- cohort-definition decisions;
- prediction-horizon selection;
- feature inclusion and exclusion;
- feature engineering;
- missingness representation;
- physiologic plausibility rules;
- model-family selection;
- preprocessing design;
- hyperparameter tuning;
- internal validation;
- final model fitting.

### Health System B

Health System B is reserved for locked external validation.

Before the System A modeling pipeline is frozen, System B is not used
for:

- feature selection;
- feature engineering;
- model selection;
- hyperparameter tuning;
- calibration adjustment;
- performance-driven cohort modification.

Post-validation analyses in System B are conducted only after the
primary external predictions have been generated and evaluated.


## 4. Primary Prediction Task

### 4.1 Landmark

The prediction landmark is fixed at:

**ICU hour 6**

Predictors may use only information observed at or before ICU hour 6.

No post-landmark measurement is used in the prediction representation.

### 4.2 Outcome Window

The primary outcome is incident sepsis onset during:

$$
(6,18]
$$

hours after ICU admission.

For patient $i$,

$$
Y_i
=
\mathbf{1}
\left(
6 < t_{\mathrm{sepsis},i} \le 18
\right).
$$

The prediction target is therefore:

$$
P(
\text{incident sepsis onset during }(6,18]
\mid
\text{information available through hour 6}
).
$$

The 12-hour post-landmark horizon is selected using System A and frozen
before examination of System B predictive performance.


## 5. Outcome Reconstruction

The Challenge `SepsisLabel` becomes positive six hours before the
reconstructed clinical sepsis-onset time.

For patients with an observed transition from:

`SepsisLabel = 0`

to:

`SepsisLabel = 1`

clinical onset is reconstructed as:

$$
t_{\mathrm{sepsis}}
=
t_{\mathrm{first\ positive\ label}}
+
6.
$$

Patients whose first available observation is already positive have a
left-truncated onset time that cannot be reconstructed exactly.

These patients are excluded from the primary incident-onset analysis.


## 6. Primary Cohort Eligibility

Patients are eligible for the landmark analysis if:

1. ICU hour 6 is observed;
2. sepsis onset is not left truncated;
3. no reconstructed sepsis onset has occurred at or before ICU hour 6;
4. the primary outcome can be classified using observed follow-up.

A positive case is a patient with reconstructed sepsis onset during:

$$
(6,18].
$$

A patient without an observed event in the prediction window is
classified as negative only if observation extends through ICU hour 18.

Patients with insufficient follow-up and no observed incident event are
not classified as negative.

This rule prevents incomplete follow-up from being interpreted as
absence of sepsis.


## 7. Frozen Feature Design

Feature design is conducted using Health System A only.

The primary predictor representation uses:

- longitudinal vital-sign summaries;
- longitudinal laboratory summaries;
- explicit measurement-availability indicators;
- static demographic variables;
- ICU-unit context.

Longitudinal predictors must have at least 10% availability in eligible
System A patients by the landmark to enter the primary representation.

Five highly sparse variables are excluded:

- `Bilirubin_total`;
- `Fibrinogen`;
- `TroponinI`;
- `Bilirubin_direct`;
- `EtCO2`.

For retained vital signs, the primary summaries are:

- last;
- mean;
- minimum;
- maximum;
- availability.

For retained laboratory variables, the primary summaries are:

- last;
- availability.

Static numeric predictors are:

- age;
- gender;
- hospital-admission timing relative to ICU admission.

ICU-unit context is represented as:

- Unit1;
- Unit2;
- Unknown reference state.

The final frozen prediction representation contains 84 model columns.

Longitudinal slopes and measurement counts are not part of the primary
prediction representation.

Measurement counts may be examined later as post-validation
distribution-shift diagnostics.


## 8. Physiologic Plausibility Rules

Clearly implausible physiologic values identified during the System A
feature audit are converted to missing using variable-specific rules.

These rules are frozen in System A and applied unchanged to System B.

No new cleaning rule is introduced in response to System B model
performance or distributional anomalies.


## 9. Candidate Models

Three model families are evaluated:

1. Ridge logistic regression;
2. Elastic-net logistic regression;
3. LightGBM.

The comparison is intentionally limited to two regularized linear
models and one nonlinear tree-based model.

No additional model family is introduced after examination of System B
performance.


## 10. Internal Validation in System A

Internal model evaluation uses nested cross-validation.

Five outer stratified folds generate out-of-fold predictions.

Within each outer training fold:

- preprocessing is fit using training data only;
- model hyperparameters are selected using training data only;
- the held-out outer fold is used only for evaluation.

For the linear models, imputation and scaling are estimated within the
training folds.

LightGBM retains native missing-value handling.

Primary internal metrics include:

- AUROC;
- AUPRC;
- Brier score;
- log loss;
- calibration-in-the-large (CITL);
- joint calibration intercept;
- calibration slope.

Patient-level bootstrap resampling is used to quantify uncertainty and
to support paired model comparisons.


## 11. Model Freeze Before External Validation

After internal validation:

- each model is tuned on System A;
- each model is fit using the full eligible System A cohort;
- preprocessing behavior is fixed;
- the 84-column feature specification is fixed;
- model metadata and fitted objects are frozen.

Only these frozen models are used for the primary System B external
validation.


## 12. Primary External Validation

The frozen System A cohort logic and feature representation are applied
unchanged to Health System B.

Before prediction, System B is audited only for structural compatibility
with the frozen pipeline.

The primary System B analysis does not include:

- model refitting;
- feature reselection;
- target-system scaling;
- target-system recalibration;
- performance-driven cohort alteration.

The primary external predictions therefore represent locked,
unmodified transport of the System A models.

External performance is evaluated using:

- AUROC;
- AUPRC;
- Brier score;
- log loss;
- calibration-in-the-large (CITL);
- joint calibration intercept;
- calibration slope;
- calibration curves.

Patient-level bootstrap resampling is used for uncertainty estimation.

Paired bootstrap comparisons are used for direct model comparisons
within System B.

Because outcome prevalence differs substantially between Systems A and B,
raw Brier score, log loss, and AUPRC are reported within each system but
their cross-system absolute differences are not treated as direct
transportability effects.


## 13. Primary Versus Post-Validation Analyses

The locked System B external validation is the primary transportability
analysis.

The following analyses occur only after primary external validation and
are treated as secondary diagnostic, sensitivity, or model-updating
analyses:

1. cross-system distribution-shift diagnostics, including
   domain-separability analysis;
2. target-system recalibration;
3. subgroup sensitivity analysis.

Results from these analyses do not modify the frozen primary external
validation.

## 14. Post-Validation Distribution-Shift Diagnostics

After primary external validation, Systems A and B may be compared with
respect to:

- outcome prevalence;
- measurement availability;
- observed clinical-value distributions;
- measurement intensity;
- normalized measurement density;
- demographic composition;
- ICU-unit context;
- selected variable-specific anomalies.

Domain classifiers may also be used to quantify observable separation
between the two systems.

These analyses are descriptive.

They are not interpreted as identifying causal mechanisms of prediction
failure.


## 15. Post-Validation Measurement Anomalies

Unexpected variable distributions identified in System B after external
validation may be audited separately.

Such findings are documented but do not trigger retrospective
modification of:

- the frozen feature specification;
- the primary System B cohort;
- the primary external predictions.

In particular, suspected measurement-definition or scale heterogeneity
is treated as a transportability diagnostic unless an error in the
frozen analysis itself is demonstrated.


## 16. Target-System Recalibration

Recalibration is a secondary model-updating analysis performed after
locked external validation.

The underlying System A prediction models remain unchanged.

Two strategies are evaluated:

### Intercept-only updating

$$
\operatorname{logit}(p_{\mathrm{updated}})
=
\alpha
+
\operatorname{logit}(p_{\mathrm{original}})
$$

### Intercept-and-slope updating

$$
\operatorname{logit}(p_{\mathrm{updated}})
=
\alpha
+
\beta
\operatorname{logit}(p_{\mathrm{original}})
$$

Primary evaluation of recalibration uses five-fold stratified
cross-fitting within System B.

Recalibration parameters are estimated on four folds and applied to the
held-out fold.

This procedure generates an updated prediction for each patient without
using that patient's outcome to estimate the corresponding recalibration
mapping.

The recalibration analysis is interpreted as target-system updating.

It is not considered independent validation of the updated models in a
new health system.


## 17. Subgroup Sensitivity Analysis

Subgroup analysis is performed after primary external validation.

Evaluated dimensions are limited to:

- age <65 versus age ≥65;
- Gender = 0 versus Gender = 1;
- ICU-unit context: Unit1, Unit2, or Unknown.

Age and gender are treated as the demographic sensitivity dimensions.

ICU-unit context is treated as a secondary care-context sensitivity
analysis.

Subgroup analyses use the original locked System B predictions.

They do not replace the primary external-validation estimates.

Subgroup-specific uncertainty is quantified using bootstrap resampling.

Focused AUROC contrasts are interpreted as sensitivity analyses rather
than multiplicity-adjusted confirmatory interaction tests.


## 18. Analysis Ordering

The intended analysis sequence is:

1. System A cohort audit;
2. System A feature and missingness audit;
3. System A internal model development and validation;
4. final model freeze;
5. locked System B external validation;
6. distribution-shift diagnostics;
7. target-system recalibration;
8. subgroup sensitivity analysis.

This sequence is used to distinguish primary evaluation from later
diagnostic or exploratory information.


## 19. Protection Against Retrospective Revision

Findings discovered after primary external validation are not used to
retroactively redefine the primary experiment.

Examples include:

- strong System A–B measurement-availability differences;
- differences in measurement intensity;
- domain-classification results;
- suspected calcium measurement heterogeneity;
- ICU-context subgroup patterns.

Such results may motivate interpretation or future research, but the
locked external-validation results remain unchanged.


## 20. Interpretation Boundaries

The study evaluates transportability between two health-system
environments represented in a single public dataset.

It therefore does not establish transportability to arbitrary hospitals,
time periods, or patient populations.

Distribution-shift diagnostics identify associations rather than causal
mechanisms.

Target-system recalibration is evaluated within System B using
cross-fitting and is not independently validated in a third environment.

Subgroup analyses are sensitivity analyses and may have substantial
uncertainty because the number of external events is limited.

The scientific objective is to characterize prediction reliability
under cross-system transport rather than to optimize performance for the
original Challenge leaderboard.