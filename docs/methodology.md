# Methodology

## 1. Study Objective and Design

This study evaluates the transportability of clinical risk prediction
models across two health systems using the PhysioNet/Computing in
Cardiology Challenge 2019 sepsis dataset.

The central question is whether models developed in one health system
retain discrimination and probability calibration when transported to a
different health system with potentially different patient populations,
measurement practices, and documentation processes.

The study follows a deliberately separated development and external
validation design:

- **Health System A** is used for cohort design, feature specification,
  model development, hyperparameter tuning, and internal validation.
- **Health System B** is reserved for locked external validation and is
  not used for model selection or feature engineering before the System
  A analysis is frozen.
- Distribution-shift analyses, recalibration experiments, and subgroup
  sensitivity analyses are conducted only after the primary external
  validation has been completed.

This ordering is intended to preserve the interpretation of Health
System B as an external evaluation environment rather than an additional
development dataset.

The study focuses on transportability rather than leaderboard
optimization for the original Challenge task.

---

## 2. Data Source and Health-System Separation

The analysis uses the publicly available PhysioNet/Computing in
Cardiology Challenge 2019 dataset.

The original data contain longitudinal ICU measurements from two
separate health systems:

- **System A:** 20,336 patient records;
- **System B:** 20,000 patient records.

Each patient record contains hourly physiological measurements,
laboratory variables, demographic information, care-context variables,
and the Challenge `SepsisLabel`.

The two systems are treated as distinct deployment environments rather
than pooled into a single modeling dataset.

System A is the sole development environment.

System B remains untouched during:

- prediction-horizon selection;
- cohort-definition decisions;
- feature inclusion and exclusion;
- feature engineering;
- physiologic plausibility rules;
- model-family selection;
- hyperparameter tuning;
- internal model comparison.

Only after the System A modeling pipeline is finalized and frozen are
the locked models applied to System B.

---

## 3. Prediction Task

### 3.1 Landmark

Prediction is performed at **ICU hour 6**.

For each eligible patient, predictors are constructed using only
information available at or before ICU hour 6.

No measurements recorded after the landmark are included in the model
representation.

### 3.2 Prediction Horizon

The primary outcome is incident sepsis onset during:

```math
(6, 18]
```

hours after ICU admission.

Thus, the prediction task is:

```math
P(\text{incident sepsis onset in } (6,18]
\mid
\text{information available through hour 6})
```

The 12-hour post-landmark horizon was selected during System A cohort
design and frozen before any examination of System B model performance.

### 3.3 Reconstruction of Sepsis Onset

In the Challenge data, the first positive `SepsisLabel` occurs six hours
before the reconstructed clinical sepsis-onset time.

For patients with an observed transition from `SepsisLabel = 0` to
`SepsisLabel = 1`, the onset time is therefore reconstructed from the
first positive label according to the Challenge labeling convention.

Patients whose first available record already has
`SepsisLabel = 1` are treated as left-truncated with respect to onset
timing and are excluded from the primary incident-onset analysis.

### 3.4 Eligibility and Outcome Observability

A patient is eligible for the primary landmark analysis only if:

1. the patient is under observation at ICU hour 6;
2. the patient does not have a left-truncated sepsis onset;
3. reconstructed sepsis onset has not occurred at or before ICU hour 6;
4. the primary outcome can be classified without relying on unavailable
   future follow-up.

A positive outcome is assigned when reconstructed sepsis onset occurs
within the interval $(6,18]$.

For a negative outcome, follow-up must extend through the end of the
prediction horizon unless an incident event has already been observed.

This requirement prevents patients with insufficient post-landmark
follow-up from being incorrectly classified as non-events.

## 4. Feature Representation and Missingness

### 4.1 Candidate Predictors

Feature design is performed exclusively in Health System A.

The initial candidate set contains 39 clinical and demographic
predictors after excluding the outcome label and `ICULOS` from the model
feature space.

Longitudinal variables are retained for the primary representation when
they are observed in at least 10% of eligible System A patients by the
hour-6 landmark.

This rule is applied before examining System B.

Five extremely sparse variables are excluded from the primary model
representation:

- `Bilirubin_total`;
- `Fibrinogen`;
- `TroponinI`;
- `Bilirubin_direct`;
- `EtCO2`.

The retained longitudinal variables consist of:

- 7 vital-sign variables;
- 22 laboratory variables.

Static and care-context predictors are retained separately.

---

### 4.2 Landmark Feature Construction

All longitudinal summaries use measurements observed at or before ICU
hour 6.

For each retained vital sign, the following features are constructed:

- last observed value;
- mean;
- minimum;
- maximum;
- availability indicator.

For each retained laboratory variable, the following features are
constructed:

- last observed value;
- availability indicator.

Static numeric predictors are:

- age;
- gender;
- hospital-admission time relative to ICU admission.

The original `Unit1` and `Unit2` indicators are collapsed into a single
ICU-unit concept with three possible states:

- Unit1;
- Unit2;
- Unknown.

For model fitting, Unit1 and Unit2 are represented with indicator
columns and Unknown serves as the reference state.

The final frozen model matrix contains **84 predictors**.

The representation consists of:

- 35 vital-sign features;
- 44 laboratory features;
- 3 static numeric features;
- 2 ICU-unit indicator features.

---

### 4.3 Missingness Representation

Missingness is treated as part of the observed clinical data-generating
process rather than being discarded before modeling.

Each retained longitudinal variable therefore includes an explicit
availability indicator.

This allows the models to distinguish between:

1. a measured clinical value; and
2. the absence of a measurement before the landmark.

No global imputation is performed when the frozen patient-level feature
matrix is created.

For the linear models, imputation and scaling are estimated within
cross-validation training folds to prevent information leakage.

LightGBM is fit using the frozen feature values while preserving missing
values for the model's native missing-value handling.

---

### 4.4 Features Deliberately Excluded From the Primary Representation

The primary representation does not include longitudinal slopes or other
trend coefficients.

Although temporal trends may contain useful information, many laboratory
variables are observed too sparsely before hour 6 to support stable
patient-level trend estimates across the full cohort.

Measurement counts are also excluded from the primary prediction
representation.

Counts are retained for later distribution-shift diagnostics because
they may reflect health-system-specific measurement intensity and
documentation behavior.

This separation prevents post hoc observations about cross-system
measurement differences from altering the frozen prediction models.

---

### 4.5 Physiologic Plausibility Cleaning

A small number of clearly implausible physiologic values are converted
to missing using variable-specific plausibility rules defined during the
System A feature audit.

The same frozen rules are then applied unchanged to System B.

In System A, this cleaning affects only a small number of observations:

- 1 BaseExcess measurement;
- 3 DBP measurements;
- 1 FiO2 measurement;
- 5 MAP measurements;
- 1 temperature measurement.

These values occur across 11 patients.

After rebuilding the landmark feature matrix, 25 engineered feature
values differ as a consequence of the cleaning rules.

The purpose of this step is limited to removing obvious physiologic
implausibilities rather than harmonizing distributions between health
systems.

No cleaning rule is introduced after examination of System B external
performance.

---

### 4.6 Frozen Feature Specification

After completion of the System A feature and missingness audit, the
following are frozen before external validation:

- included raw variables;
- excluded raw variables;
- hour-6 aggregation rules;
- availability indicators;
- ICU-unit encoding;
- physiologic plausibility rules;
- final model-column ordering.

The identical 84-column representation is subsequently constructed in
System B.

No System B information is used to add, remove, redefine, or transform
predictors before the locked external evaluation.

## 5. Model Development and Internal Validation

### 5.1 Candidate Model Families

Three model families are evaluated in Health System A:

1. Ridge logistic regression;
2. Elastic-net logistic regression;
3. LightGBM gradient-boosted decision trees.

The model set is intentionally limited.

The objective is not to construct a large model zoo, but to compare two
regularized linear approaches with a nonlinear tree-based learner under
a common frozen cohort and feature representation.

No additional model family is introduced after examination of Health
System B performance.

---

### 5.2 Nested Internal Validation

Model development is performed using nested cross-validation within
Health System A.

Five outer stratified folds are used to generate out-of-fold predictions
for internal performance estimation.

Within each outer training set, model-specific hyperparameters are tuned
using only the corresponding training data.

The held-out outer fold is not used for:

- preprocessing estimation;
- hyperparameter selection;
- model fitting.

This process is repeated until every eligible System A patient has an
out-of-fold prediction from a model that was not trained on that patient.

The pooled out-of-fold predictions are used as the primary estimate of
internal predictive performance.

---

### 5.3 Preprocessing Within Cross-Validation

For Ridge and elastic-net logistic regression, preprocessing is learned
inside each cross-validation training fold.

This includes:

- numeric missing-value imputation;
- feature scaling;
- fitting of the regularized logistic regression model.

Preprocessing parameters estimated from a training fold are applied
unchanged to the corresponding held-out fold.

LightGBM is trained using the frozen feature representation while
preserving missing values for native tree-based missing-value handling.

No preprocessing parameter is estimated from the full dataset before
generation of internal out-of-fold predictions.

---

### 5.4 Internal Validation Metrics

Predictive performance is evaluated using complementary measures of
discrimination, probability accuracy, and calibration.

Reported metrics include:

- area under the receiver operating characteristic curve (AUROC);
- area under the precision-recall curve (AUPRC);
- Brier score;
- binary log loss;
- calibration-in-the-large (CITL);
- joint calibration intercept;
- calibration slope.

AUROC summarizes ranking discrimination.

AUPRC is reported because the outcome is rare, while recognizing that
its magnitude depends strongly on event prevalence.

Brier score and log loss summarize probabilistic predictive accuracy.

Calibration is evaluated using a logistic calibration model of the form:

```math
\mathrm{logit}(P(Y=1))
=
\alpha
+
\beta
\mathrm{logit}(\hat p)
```

where:

- $\alpha$ is the joint calibration intercept;
- $\beta$ is the calibration slope;
- $\hat p$ is the predicted probability.

Calibration-in-the-large (CITL) is estimated separately by fixing the
calibration slope at one:

```math
\mathrm{logit}(P(Y=1))
=
\alpha_{\mathrm{CITL}}
+
\mathrm{logit}(\hat p)
```

A CITL value of zero indicates correct average risk prediction.

Negative CITL indicates systematic overprediction, whereas positive CITL
indicates systematic underprediction.

This quantity is distinct from the intercept of the two-parameter
calibration model, in which the intercept and slope are estimated
jointly.

For numerical estimation, predicted probabilities are clipped to
$[10^{-6}, 1-10^{-6}]$ before logit transformation. CITL is obtained by
solving the intercept-only logistic likelihood score equation with slope
fixed at one. The joint calibration intercept and slope are obtained by
solving the two unpenalized logistic likelihood score equations
simultaneously. This score-equation implementation is used consistently
for internal validation, locked external validation, recalibration
evaluation, and subgroup calibration summaries.

A calibration slope below one indicates predictions that are too
dispersed on the log-odds scale, whereas a slope above one indicates
predictions that are insufficiently dispersed.

---

### 5.5 Internal Uncertainty and Model Comparisons

Patient-level bootstrap resampling is used to quantify uncertainty in
internal performance estimates.

Paired bootstrap comparisons are used when comparing models so that the
same resampled patients contribute predictions from both models.

Differences are evaluated for metrics including:

- Brier score;
- log loss;
- AUROC;
- AUPRC.

Model comparisons are interpreted using both point estimates and
bootstrap confidence intervals rather than point estimates alone.

---

### 5.6 Final System A Model Fitting

After completion of internal validation, each model is tuned and fit
using the complete eligible System A development cohort.

The final fitted models, feature specification, preprocessing behavior,
and model metadata are then frozen.

These frozen System A models are the only models used for the primary
Health System B external validation.

No System B outcome or performance information is used to alter the
models before external evaluation.


## 6. Locked External Validation in Health System B

### 6.1 Structural Compatibility Audit

Before model performance is evaluated, Health System B is checked for
structural compatibility with the frozen cohort-construction procedure.

The audit assesses:

- duplicated ICU-hour records;
- non-monotonic ICU-hour ordering;
- non-contiguous ICU-hour sequences;
- invalid outcome labels;
- reversions from positive to negative sepsis labels.

These checks are used only to verify that the frozen landmark and
outcome-construction logic can be applied consistently.

They do not influence model selection.

---

### 6.2 Application of the Frozen Cohort Definition

The same hour-6 landmark, outcome horizon, left-truncation handling, and
follow-up requirements defined in System A are applied unchanged to
System B.

The final eligible System B external-validation cohort contains 17,982
patients and 175 incident sepsis events.

No System B patient is added or excluded on the basis of model
predictions.

---

### 6.3 Frozen Feature Construction

The frozen System A feature specification is applied directly to System
B.

This includes the same:

- predictor inclusion rules;
- hour-6 aggregation functions;
- availability indicators;
- ICU-unit encoding;
- physiologic plausibility rules;
- final 84-column ordering.

The System B representation is constructed without redefining features
to improve cross-system agreement.

---

### 6.4 Primary External Prediction

The three frozen System A models are applied directly to the eligible
System B cohort.

During the primary external validation:

- no model parameter is refit;
- no feature is reselected;
- no System B scaling parameter is learned;
- no target-system recalibration is performed.

The resulting predictions therefore represent locked, unmodified
transported predictions from the System A development environment.

---

### 6.5 External Validation Metrics

The same core predictive metrics used during internal validation are
reported in System B:

- Brier score;
- log loss;
- AUROC;
- AUPRC;
- calibration-in-the-large;
- joint calibration intercept;
- calibration slope.

Calibration curves are also constructed to compare mean predicted risk
with observed event rates across groups of predicted risk.

Because outcome prevalence differs substantially between Systems A and
B, raw Brier score, log loss, and AUPRC values are not interpreted as
direct measures of transportability by simple comparison of their
absolute magnitudes across systems.

Greater emphasis is placed on:

- change in discrimination;
- systematic over- or underprediction;
- calibration slope;
- within-System-B model comparisons.

---

### 6.6 External Uncertainty and Model Comparisons

Patient-level bootstrap resampling is used to construct confidence
intervals for external performance.

Within System B, paired bootstrap comparisons evaluate differences
between LightGBM and the two linear models while preserving patient-level
pairing.

These within-System-B comparisons include differences in:

- Brier score;
- log loss;
- AUROC;
- AUPRC.

Internal and external performance estimates are also summarized side by
side to describe how model behavior changes during transport from
System A to System B, without treating prevalence-dependent metric
differences as direct transportability effects.


## 7. Cross-System Distribution-Shift Diagnostics

### 7.1 Purpose and Timing

Distribution-shift analyses are performed only after completion of the
locked Health System B external validation.

Their purpose is to characterize observable differences between the two
health systems that may be relevant to transportability.

These analyses are diagnostic and descriptive.

They are not used to modify the frozen primary models, and they do not
establish causal explanations for external performance degradation.

---

### 7.2 Outcome Prevalence Shift

The incidence of the primary outcome is compared between the eligible
System A and System B cohorts.

This comparison characterizes baseline-risk differences between the two
deployment environments.

No attempt is made to remove or normalize the prevalence difference
before primary external validation.

---

### 7.3 Measurement Availability Shift

For each longitudinal variable, the proportion of patients with at least
one measurement available by hour 6 is calculated separately in Systems
A and B.

Absolute percentage-point differences in availability are used to
identify variables whose measurement presence changes substantially
between systems.

This analysis directly examines whether the missingness and measurement
process itself differs across health systems.

---

### 7.4 Observed-Value Distribution Shift

Among patients with observed values, representative clinical summaries
are compared across systems.

For continuous variables, cross-system shift is described using:

- standardized mean differences;
- Kolmogorov-Smirnov statistics;
- quantile comparisons.

These diagnostics are intended to identify both broad distributional
changes and tail-specific differences.

They do not imply that observed differences reflect underlying
physiologic differences alone, because measurement definitions,
laboratory practices, and documentation processes may also differ.

---

### 7.5 Measurement-Intensity Shift

Raw pre-landmark measurement counts are compared between Systems A and
B.

Because the amount of observed pre-landmark history differs between the
systems, measurement intensity is additionally normalized by the number
of observed ICU rows available before the landmark.

This allows the analysis to distinguish, in part, between:

1. differences caused by longer or shorter available observation
   history; and
2. differences in how densely particular variables are measured or
   recorded.

Measurement counts are used only for distribution-shift diagnosis and
are not introduced into the frozen prediction models.

---

### 7.6 Demographic and Care-Context Shift

Static patient characteristics and ICU context are compared across
systems.

Evaluated variables include:

- age;
- gender;
- hospital-admission timing;
- ICU-unit representation.

This analysis distinguishes relatively modest demographic differences
from potentially larger differences in care-context or documentation
composition.

---

### 7.7 Calcium Measurement Audit

A dedicated post-validation audit is performed for calcium because the
System B distribution contains a substantial cluster of values that is
not present in System A.

The official Challenge variable definition is retained.

The observed pattern is therefore described conservatively as suspected
measurement-definition or scale heterogeneity.

The exact clinical or laboratory origin of the discrepancy is not
assumed.

Importantly, this observation does not trigger retrospective
modification of the frozen System B external-validation data or models.

---

### 7.8 Domain Separability

Supervised domain classifiers are used as post-validation diagnostics to
quantify how easily patients from Systems A and B can be distinguished
from their recorded representations.

Five representations are examined:

1. availability indicators only;
2. observed values and context;
3. the full frozen model representation;
4. a high-coverage representation based on longitudinal variables with
   at least 90% availability in both systems;
5. a stricter high-coverage clinical/basic-context representation.

The five longitudinal variables satisfying the 90% availability
criterion in both systems are:

- SBP;
- Resp;
- MAP;
- HR;
- O2Sat.

The high-coverage representation contains the frozen landmark summaries
and availability indicators for these variables together with static
patient variables and ICU-unit context.

The stricter representation removes availability indicators and
ICU-unit context and retains only:

- last, mean, minimum, and maximum summaries for the five high-coverage
  longitudinal variables;
- age;
- gender;
- hospital-admission timing.

The high-coverage analyses are post-validation sensitivity analyses
rather than feature-selection procedures for the primary prediction
models.

The domain label indicates whether a patient originates from System A or
System B.

Five-fold cross-validated logistic regression is used to estimate domain
AUROC.

High domain-classification AUROC indicates strong observable
cross-system separation but does not identify a causal mechanism for
prediction failure.

The 90% availability threshold is not used to revise the frozen
prediction feature set.


## 8. Target-System Recalibration

### 8.1 Objective

After locked external validation, a separate target-system model-updating
analysis evaluates whether probability calibration in Health System B
can be improved without refitting the underlying prediction models.

The original System A models remain unchanged.

Only the mapping from their predicted probabilities to updated
Health System B probabilities is estimated.

---

### 8.2 Intercept-Only Recalibration

Intercept-only recalibration applies:

```math
\mathrm{logit}(p_{\text{updated}})
=
\alpha
+
\mathrm{logit}(p_{\text{original}})
```

where $\alpha$ is estimated from target-system outcomes.

This approach corrects systematic over- or underprediction while fixing
the risk-scale coefficient at one.

It therefore primarily addresses calibration-in-the-large.

---

### 8.3 Intercept-and-Slope Recalibration

The second strategy estimates both an intercept and a slope:

```math
\mathrm{logit}(p_{\text{updated}})
=
\alpha
+
\beta
\mathrm{logit}(p_{\text{original}})
```

This approach can correct both:

- baseline-risk mismatch;
- excessive or insufficient dispersion of predicted risks.

The underlying System A prediction model is still not refit.

---

### 8.4 Cross-Fitted Evaluation in System B

Fitting and evaluating recalibration on the same complete System B cohort
would produce optimistically biased estimates of recalibration
performance.

Therefore, the primary recalibration evaluation uses five-fold
stratified cross-fitting within System B.

For each fold:

1. recalibration parameters are estimated using the other four folds;
2. the fitted recalibration transformation is applied to the held-out
   fold;
3. held-out updated probabilities are stored.

After all five folds are processed, every System B patient has a
recalibrated probability obtained without using that patient's outcome
to fit the recalibration transformation.

The held-out predictions are then pooled for evaluation.

A full-System-B recalibration fit may be examined as a parameter sanity
check, but it is not used as the primary estimate of updated predictive
performance.

---

### 8.5 Recalibration Evaluation

Original and cross-fitted updated predictions are compared using:

- Brier score;
- log loss;
- AUROC;
- AUPRC;
- calibration-in-the-large;
- joint calibration intercept;
- calibration slope;
- mean predicted risk.

The main objective is improvement in probability reliability rather than
improvement in discrimination.

Because each cross-validation fold receives a separately estimated
recalibration transformation, pooled cross-fitted AUROC and AUPRC may
change slightly even though each within-fold transformation is
monotonic.

Such small pooled discrimination changes are not interpreted as the
primary effect of recalibration.

---

### 8.6 Bootstrap Assessment of Recalibration Effects

Paired patient-level bootstrap resampling is used to quantify uncertainty
in performance differences between original and cross-fitted
recalibrated predictions.

Primary attention is given to changes in:

- Brier score;
- log loss.

The original and updated predictions are resampled together to preserve
within-patient pairing.

These intervals quantify uncertainty in the generated cross-fitted
prediction sets.

They do not represent a full nested bootstrap in which recalibration
parameters are re-estimated within every bootstrap replicate.

---

### 8.7 Incremental Value of Slope Updating

Intercept-and-slope recalibration is compared directly with
intercept-only recalibration.

For each model, the difference is defined as:

```math
\Delta
=
\text{performance}_{\text{intercept+slope}}
-
\text{performance}_{\text{intercept-only}}
```

For Brier score and log loss, negative values therefore favor estimation
of the additional slope parameter.

Paired bootstrap confidence intervals are used to quantify uncertainty
in the incremental effect of slope updating beyond baseline-risk
correction alone.

---

### 8.8 Calibration-Curve Visualization

Calibration curves are constructed for:

- original external predictions;
- cross-fitted intercept-only predictions;
- cross-fitted intercept-and-slope predictions.

Patients are grouped by predicted risk for each prediction set, and mean
predicted risk is compared with observed event frequency.

These plots are interpreted together with formal calibration-in-the-large
and calibration-slope estimates, particularly because the external
outcome is rare.


## 9. Subgroup Sensitivity Analysis

### 9.1 Objective

After completion of primary external validation and recalibration
analyses, subgroup sensitivity analyses evaluate whether transported
model performance varies across a limited set of clinically interpretable
Health System B strata.

These analyses do not influence model selection, feature engineering, or
the primary external-validation conclusions.

The objective is sensitivity assessment rather than discovery of
statistically significant subgroup interactions.

---

### 9.2 Subgroup Definitions

The evaluated dimensions are:

#### Age

- age <65 years;
- age ≥65 years.

#### Gender

- Gender = 0;
- Gender = 1.

#### ICU-unit context

- Unit1;
- Unit2;
- Unknown.

ICU-unit status is reconstructed from the frozen model representation.

Patients with neither Unit1 nor Unit2 indicator are classified as
Unknown.

Age and gender are treated as the primary demographic sensitivity
analyses.

ICU-unit context is treated as a secondary care-context analysis.

---

### 9.3 Feasibility Audit

Before subgroup model performance is interpreted, the following are
summarized for each subgroup:

- number of patients;
- number of events;
- event rate.

This step is used to assess whether subgroup-specific estimates are
supported by sufficient event counts.

Particular caution is applied to ICU-context groups because they contain
fewer events than the age and gender strata.

---

### 9.4 Subgroup Performance

The original locked System B predictions are used for subgroup analysis.

Recalibrated probabilities are not substituted for the primary subgroup
sensitivity analysis.

For each model and subgroup, the following are reported:

- number of patients;
- number of events;
- event rate;
- mean predicted risk;
- Brier score;
- log loss;
- AUROC;
- AUPRC;
- calibration-in-the-large;
- joint calibration intercept;
- calibration slope.

---

### 9.5 Bootstrap Uncertainty Within Subgroups

Patient-level bootstrap resampling is used to estimate uncertainty for
subgroup-specific:

- AUROC;
- AUPRC;
- calibration-in-the-large;
- calibration slope.

One thousand bootstrap replicates are used for these subgroup-specific
intervals.

Smaller subgroup event counts are expected to produce wider confidence
intervals, particularly for calibration-slope estimates.

---

### 9.6 Direct AUROC Contrasts

Subgroup-specific confidence intervals do not directly test whether two
subgroups differ from one another.

Focused bootstrap contrasts are therefore constructed for AUROC.

The evaluated contrasts are:

- age ≥65 minus age <65;
- Gender = 1 minus Gender = 0;
- Unit2 minus Unit1;
- Unknown minus Unit1;
- Unknown minus Unit2.

For each contrast, patients are resampled independently within the two
disjoint subgroups.

Two thousand bootstrap replicates are used.

The contrast is defined as:

```math
\Delta \text{AUROC}
=
\text{AUROC}_{A}
-
\text{AUROC}_{B}
```

A confidence interval containing zero is interpreted as insufficient
evidence of a clear AUROC difference in this sensitivity analysis.

AUPRC is not used as the primary direct subgroup contrast because its
absolute magnitude is strongly affected by subgroup-specific event
prevalence.

---

### 9.7 Interpretation of Subgroup Analyses

The subgroup analyses are descriptive sensitivity analyses rather than
multiplicity-adjusted confirmatory interaction tests.

Observed subgroup patterns are therefore interpreted cautiously.

In particular, associations between care-context documentation and model
performance are not interpreted as evidence that documentation
differences cause transportability failure.


## 10. Analysis Ordering and Protection Against Post Hoc Model Revision

The analysis is intentionally ordered to separate model development,
external evaluation, and post-validation investigation.

The sequence is:

1. System A cohort definition;
2. System A feature and missingness audit;
3. System A model development and internal validation;
4. freezing of the final System A models;
5. locked System B external validation;
6. post-validation distribution-shift diagnosis;
7. target-system recalibration;
8. subgroup sensitivity analysis.

This ordering is central to the study design.

Information discovered in later analyses is not used to retroactively
alter earlier frozen analyses.

Examples include:

- System B measurement-availability shifts;
- suspected calcium measurement heterogeneity;
- domain-classification results;
- ICU-context subgroup findings.

These observations are analyzed as transportability diagnostics rather
than used to redefine the primary external-validation experiment.


## 11. Reproducibility and Frozen Analysis Artifacts

### 11.1 Machine-Readable Specification Records

Core study specifications are documented separately from the analysis
notebooks under `configs/`.

The repository includes YAML records for:

- cohort and prediction-task rules;
- feature definitions and representation rules;
- model-development and validation specifications.

These YAML files provide machine-readable records of the frozen analysis
specification. They do not serve as runtime configuration files for the
completed analysis.

The notebooks remain the executable implementation of the study, while
the YAML records make the principal frozen design decisions easier to
inspect and audit.

---

### 11.2 Frozen Results

Finalized analysis artifacts are organized under:

`results/frozen/`

This directory separates frozen outputs from exploratory analysis.

Locally generated frozen artifacts include:

- model metadata;
- fitted System A models;
- System A out-of-fold predictions;
- System B external predictions;
- System B cross-fitted recalibrated predictions;
- internal and external performance tables;
- distribution-shift diagnostics;
- recalibration summaries;
- subgroup sensitivity summaries.

Patient-level prediction files are excluded from public version control,
whereas summary tables, model metadata, and fitted model objects are
retained in the repository.

---

### 11.3 Figures

Core figures are stored separately under:

`figures/`

These include visualizations of:

- measurement-availability shift;
- domain separability;
- external calibration;
- recalibration;
- subgroup AUROC contrasts.

---

### 11.4 Data Separation

The repository distinguishes:

- `data/raw/`
- `data/processed/`
- `data/metadata/`

Raw source data are kept separate from derived analytic datasets.

Processed patient-level feature matrices correspond to the frozen
analysis representation used in model development and validation.

Raw source data and patient-level processed feature matrices are excluded
from public version control and are regenerated locally as needed.

---

### 11.5 Executable Analysis Workflow

The chronological analysis is implemented in seven notebooks:

1. `01_cohort_audit.ipynb`;
2. `02_feature_missingness_audit.ipynb`;
3. `03_system_a_internal_validation.ipynb`;
4. `04_system_b_external_validation.ipynb`;
5. `05_distribution_shift.ipynb`;
6. `06_recalibration.ipynb`;
7. `07_subgroup_sensitivity.ipynb`.

The notebooks contain the executable implementation of cohort
construction, feature engineering, model development, validation,
distribution-shift analysis, recalibration, and subgroup sensitivity.

Their numerical ordering preserves the intended separation between
development, locked external evaluation, and post-validation analyses.


## 12. Methodological Scope and Interpretation

This study is designed to evaluate transportability between two health
systems represented in the Challenge 2019 dataset.

Several interpretation constraints remain important.

First, the analysis includes only two source environments.

Observed System A-to-System B transportability therefore does not
guarantee performance in additional hospitals, time periods, or patient
populations.

Second, the Health System B external-validation cohort contains only
175 incident sepsis events.

Although bootstrap uncertainty is reported, estimates of calibration
slope and subgroup performance remain imprecise in smaller strata.

Third, the post-validation distribution-shift analyses are diagnostic.

They can identify strong differences in measurement availability,
recorded values, measurement intensity, and care-context composition,
but they cannot establish which differences causally produce model
performance degradation.

Fourth, cross-fitted recalibration estimates the potential value of
target-system probability updating using data from System B.

It does not constitute independent external validation of the
recalibrated models in a third health system or later time period.

Finally, the study evaluates a deliberately limited set of model
families.

Its primary scientific objective is not to identify the highest possible
Challenge score, but to characterize how clinically oriented prediction
models behave when transported across health-system environments.