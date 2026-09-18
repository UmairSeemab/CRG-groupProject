# CRG Group Project: CpG-Based Machine Learning for Melanoma Stage Classification

This project evaluates whether CpG methylation-related measurements can distinguish low-stage from high-stage melanoma and whether a compact CpG marker panel can retain useful predictive information.

The updated workflow compares regularized linear models, bagged decision trees, gradient-boosting models, and compact CpG panels. Model development is performed only in the development cohort using nested stratified cross-validation. Final locked models are evaluated once on an untouched 20% holdout set.

## Main updated notebook

| Notebook | Purpose | Google Colab |
|---|---|---|
| [`CpG_step_by_step_XGBoost_LightGBM.ipynb`](CpG_step_by_step_XGBoost_LightGBM.ipynb) | Complete EDA, model comparison, CpG panel selection, XGBoost, LightGBM, held-out evaluation and exports | [Open in Colab](https://colab.research.google.com/github/UmairSeemab/CRG-groupProject/blob/main/CpG_step_by_step_XGBoost_LightGBM.ipynb) |

The notebook first looks for `Xsurv.csv` locally. If the file is not present, it automatically downloads:

```text
https://raw.githubusercontent.com/UmairSeemab/CRG-groupProject/main/Xsurv.csv
```

No manual data-path editing is normally required when the repository files are on the `main` branch.

## Dataset

The supplied [`Xsurv.csv`](Xsurv.csv) contains:

| Item | Value |
|---|---:|
| Patients | 320 |
| Predictor variables | 199 |
| CpG variables | 197 |
| Clinical predictors | AGE, SEX |
| Outcome | Stage |
| Low-stage observations | 157 |
| High-stage observations | 163 |
| Development cohort | 256 patients, 80% |
| Held-out test cohort | 64 patients, 20% |

The first CSV column is used as the row identifier and is not treated as a predictor. Stage 1 is internally recoded to 0 and Stage 2 to 1, making high stage the positive class.

The supplied file does not document the CpG transformation, assay platform, genome build, AGE units, or mapping of the SEX codes. These variables are therefore used as supplied without inventing metadata.

## Analysis workflow

The notebook follows this sequence:

```text
Xsurv.csv
    |
    v
Data audit
    |
    v
80/20 stratified development-test split
    |
    +------------------------------+
    |                              |
    v                              v
Development set                 Test set
256 patients                    64 patients
    |                              |
    v                              |
EDA on development data            |
    |                              |
    v                              |
5 x 3 nested stratified CV          |
    |                              |
    v                              |
Model + hyperparameter selection    |
    |                              |
    v                              |
Lock models and CpG panel           |
    |                              |
    +----------------------------->|
                                   v
                         One-time held-out evaluation
                                   |
                                   v
                       Metrics + bootstrap intervals
```

The held-out test set is not used for exploratory analysis, feature selection, hyperparameter tuning, model selection, or threshold optimization.

## Exploratory data analysis

EDA is restricted to the development cohort and includes:

- dataset structure, missingness, duplicate and constant-column checks
- AGE and SEX distributions
- CpG value distribution
- pairwise CpG correlations
- identification of highly correlated CpG pairs
- PCA of standardized CpG measurements

PCA is used only for visualization. PCA components are not used as classifier inputs.

## Machine-learning models

The updated analysis evaluates 15 predefined model configurations.

### Clinical-only models

Predictors: `AGE` and `SEX`.

1. Clinical elastic-net logistic regression
2. Clinical random forest
3. Clinical XGBoost
4. Clinical LightGBM

### CpG-only models

Predictors: all 197 CpGs.

5. CpG elastic-net logistic regression
6. CpG random forest
7. CpG XGBoost
8. CpG LightGBM

### Combined models

Predictors: `AGE`, `SEX`, and all 197 CpGs.

9. Combined elastic-net logistic regression
10. Combined random forest
11. Combined XGBoost
12. Combined LightGBM

### Compact CpG-panel models

13. 5-CpG panel + logistic regression
14. 10-CpG panel + logistic regression
15. 20-CpG panel + logistic regression

The compact panels use training-only `SelectKBest(f_classif)` feature ranking inside the machine-learning pipeline.

## Why these model families are included

| Model | Reason for inclusion |
|---|---|
| Elastic-net logistic regression | Regularized linear model suitable for many potentially correlated CpG predictors. It combines L1- and L2-type shrinkage. |
| Random forest | Nonlinear tree ensemble that can model interactions without feature scaling. |
| XGBoost | Regularized sequential gradient boosting that can capture nonlinear relationships and predictor interactions. |
| LightGBM | Efficient histogram-based gradient boosting that provides an additional nonlinear boosting approach for tabular data. |
| Compact CpG logistic panels | Tests whether a smaller and more interpretable marker set can retain useful discriminatory information. |

The purpose is controlled model comparison rather than an exhaustive benchmark of every possible machine-learning algorithm.

## Hyperparameter tuning

The tuning grids are intentionally small because the development cohort contains only 256 patients.

### Elastic-net logistic regression

```text
C = 0.03, 0.3, 3
l1_ratio = 0.25, 0.75
```

### Random forest

```text
n_estimators = 150
max_depth = 3 or None
min_samples_leaf = 3 or 8
```

### XGBoost

Fixed settings include 200 trees, subsampling, column subsampling and histogram tree construction.

Tuned parameters:

```text
max_depth = 2 or 3
learning_rate = 0.03 or 0.10
```

### LightGBM

Fixed settings include 200 estimators, minimum child samples and column subsampling.

Tuned parameters:

```text
num_leaves = 7 or 15
learning_rate = 0.03 or 0.10
```

### Compact CpG panels

```text
Panel size = 5, 10 or 20 CpGs
Logistic-regression C = 0.03, 0.3 or 3
```

All scaling, feature selection and parameter fitting occur inside the corresponding training partitions.

## Evaluation strategy

### 1. Stratified 80/20 holdout split

The full dataset is divided into:

- 80% development data: 256 patients
- 20% held-out test data: 64 patients

Stratification preserves the low-stage/high-stage distribution.

### 2. Nested stratified cross-validation

Model development uses:

```text
Outer CV = 5 folds
Inner CV = 3 folds
```

The inner loop selects hyperparameters using ROC-AUC. The outer loop estimates development-set performance of the complete tuning procedure.

This design reduces optimistic bias that would occur if model tuning and performance estimation used the same validation data.

### 3. Primary and secondary metrics

Primary model-selection metric:

```text
ROC-AUC
```

Secondary development metrics:

- balanced accuracy
- sensitivity for high stage
- specificity for low stage

Threshold-dependent metrics use a prespecified probability threshold of `0.5`. The threshold is not optimized on the test set.

## Model locking before test evaluation

Before the held-out test set is evaluated, the notebook locks:

1. the compact CpG panel selected by the prespecified panel rule
2. the highest-AUC model overall
3. the strongest clinical-only model
4. the strongest full-CpG model
5. the best XGBoost configuration
6. the best LightGBM configuration

Duplicate roles are automatically removed when the same model satisfies more than one category.

The compact-panel rule selects the smallest panel whose mean nested-CV ROC-AUC is within `0.02` of the best compact panel. This is a practical selection rule and is not a statistical noninferiority test.

## Held-out test evaluation

Locked models are fitted using the complete development cohort and then evaluated once on the untouched 64-patient test cohort.

Reported test metrics include:

- ROC-AUC
- average precision
- balanced accuracy
- sensitivity
- specificity
- Brier score
- ROC curves
- precision-recall curves
- confusion matrix for the selected compact panel

### Bootstrap uncertainty

The notebook uses:

```text
2,000 stratified bootstrap resamples
```

The 2.5th and 97.5th percentiles provide conditional 95% bootstrap intervals for held-out performance.

These intervals quantify sampling uncertainty in the held-out patients conditional on the fitted models. They do not capture all uncertainty from model selection, preprocessing, or unknown upstream generation of `Xsurv.csv`.

## Generated figures

All figures are written to `Xsurv_results/`.

| Figure | Description |
|---|---|
| `01_split.png` | Development-test class distribution |
| `02_distributions.png` | Clinical and CpG distributions in development data |
| `03_correlations.png` | CpG correlation analysis |
| `04_pca.png` | Exploratory PCA of development-set CpGs |
| `05_nested_comparison.png` | Outer-fold ROC-AUC for all 15 model configurations |
| `05b_algorithm_feature_heatmap.png` | Mean nested-CV ROC-AUC for elastic-net, random forest, XGBoost and LightGBM across clinical, CpG and combined predictor sets |
| `06_panel_sizes.png` | Comparison of 5-, 10- and 20-CpG panels with the CpG elastic-net model |
| `07_marker_stability.png` | CpG selection/stability analysis across outer training folds |
| `08_marker_distributions.png` | Development-set distributions for selected CpGs |
| `09_test_performance.png` | Held-out ROC curves, precision-recall curves and compact-panel confusion matrix |
| `10_test_intervals.png` | Held-out ROC-AUC estimates with conditional 95% bootstrap intervals |

## Generated result files

The notebook also writes:

| File | Contents |
|---|---|
| `data_audit.csv` | Dataset audit summary |
| `split_manifest.csv` | Development/test membership |
| `high_correlations_development.csv` | Highly correlated CpG pairs |
| `nested_cv_folds.csv` | Fold-level nested-CV metrics and tuned parameters |
| `nested_cv_summary.csv` | Mean model performance across outer folds |
| `marker_ranking.csv` | CpG selection/stability information |
| `final_panel.csv` | Locked compact CpGs, coefficients and annotation status |
| `test_metrics.csv` | Held-out point estimates |
| `test_intervals.csv` | Bootstrap uncertainty intervals |
| `test_predictions.csv` | Held-out predicted probabilities for each locked model |
| `analysis_manifest.json` | Seed, settings, selected models, selected CpGs, software versions and input checksum |

The final XGBoost and LightGBM results should be read from `nested_cv_summary.csv`, `test_metrics.csv`, and `test_intervals.csv` after running the complete 5 x 3 nested-CV analysis. The README does not hard-code a boosting-model winner before that full run is completed.

## Run in Google Colab

1. Upload these files to the root of the GitHub repository:

```text
README.md
Xsurv.csv
CpG_step_by_step_XGBoost_LightGBM.ipynb
```

2. Open the notebook using the Colab badge/link above.
3. Use a fresh CPU runtime.
4. Run the cells from top to bottom.
5. If XGBoost or LightGBM is unavailable, the setup cell installs the missing package automatically.
6. The notebook looks for `Xsurv.csv` locally and downloads it directly from the GitHub repository if needed.
7. Retrieve figures and tables from `Xsurv_results/` after execution.

The full nested cross-validation is computationally heavier than the original workflow because XGBoost and LightGBM are now included in all three predictor-set comparisons.

## Run locally

Python 3 and Jupyter are required.

Install the main dependencies:

```bash
python -m pip install jupyterlab numpy pandas scipy scikit-learn matplotlib seaborn xgboost lightgbm
```

Then start Jupyter:

```bash
python -m jupyterlab
```

Open:

```text
CpG_step_by_step_XGBoost_LightGBM.ipynb
```

and run all cells in order.

## Suggested repository layout

```text
CRG-groupProject/
├── README.md
├── Xsurv.csv
├── CpG_step_by_step_XGBoost_LightGBM.ipynb
├── CpG_step_by_step.ipynb
├── Member_1_Data_EDA.ipynb
├── Member_2_Models_Markers.ipynb
├── Member_3_Evaluation_Reporting.ipynb
└── project_Xsurv.ipynb
```

`CpG_step_by_step_XGBoost_LightGBM.ipynb` is the updated combined analysis containing XGBoost and LightGBM.

The existing three-member notebooks represent the earlier split workflow unless they are separately updated. Do not assume that XGBoost and LightGBM are present in `Member_2_Models_Markers.ipynb` or `Member_3_Evaluation_Reporting.ipynb` solely because they are included in the combined notebook.

## Optional CpG annotation

The dataset contains CpG probe identifiers but does not provide a verified platform manifest or gene mapping.

If a verified annotation file is available, create:

```text
CpG_annotation.csv
```

with columns:

```text
CpG,Gene,Chromosome,Position,Genome_build,Annotation_source
```

The notebook merges this file when available. Otherwise, selected CpGs remain explicitly unannotated rather than assigning unsupported gene mappings.

## Interpretation

The project performs internal validation of melanoma stage classification. It does not establish that the selected CpGs are validated biological biomarkers.

Important limitations include:

- one supplied dataset only
- 64 patients in the held-out test set
- no independent external validation cohort
- unknown upstream CpG filtering and transformation
- no supplied batch, site or patient-level dependency information
- unverified AGE and SEX metadata
- no verified CpG annotation manifest

Feature-selection frequency indicates reproducibility within overlapping development folds. It is not the probability that a CpG is biologically valid.

Similarly, a higher ROC-AUC for XGBoost, LightGBM, elastic-net or random forest should be interpreted as predictive performance in this internal analysis, not biological evidence or clinical utility.

## Scientific scope

The supplied assignment references:

Li et al. *Efficient gradient boosting for prognostic biomarker discovery*. Bioinformatics. 2022;38(6):1631-1638. DOI: 10.1093/bioinformatics/btab869.

The present project uses the supplied melanoma data for binary stage classification. It does not reproduce the original survival-analysis methodology.

Useful documentation:

- XGBoost: https://xgboost.readthedocs.io/
- LightGBM: https://lightgbm.readthedocs.io/
- scikit-learn nested cross-validation: https://scikit-learn.org/stable/auto_examples/model_selection/plot_nested_cross_validation_iris.html
- scikit-learn common pitfalls and data leakage: https://scikit-learn.org/stable/common_pitfalls.html

## Reproducibility

The notebook records:

- random seed
- development/test split
- outer and inner CV folds
- selected model roles
- selected CpGs
- tuned hyperparameters
- package versions
- SHA-256 checksum of the input dataset

Keep the untouched test set reserved for the final evaluation. Do not repeatedly change seeds, models, features, or thresholds based on held-out test performance.
