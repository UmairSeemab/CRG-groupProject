# CRG Group Project: CpG Markers for Melanoma Stage

An educational machine-learning project that investigates whether DNA methylation CpG markers can distinguish low-stage from high-stage melanoma. The aim is to identify an interpretable marker panel and evaluate its predictions on held-out patients.

## Open in Google Colab

Repository: [UmairSeemab/CRG-groupProject](https://github.com/UmairSeemab/CRG-groupProject). The buttons below target the root-level notebooks on the `main` branch, matching the supplied repository archive.

| Notebook | Purpose | Open |
|---|---|---|
| [CpG_step_by_step.ipynb](CpG_step_by_step.ipynb) | Complete analysis with explanations and saved figures | [![Open analysis in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/UmairSeemab/CRG-groupProject/blob/main/CpG_step_by_step.ipynb) |
| [project_Xsurv.ipynb](project_Xsurv.ipynb) | Original assignment and exploratory starting notebook | [![Open starter in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/UmairSeemab/CRG-groupProject/blob/main/project_Xsurv.ipynb) |

Opening a notebook in Colab does not automatically copy the repository's CSV into the runtime. Follow the data-loading steps below before running the analysis.

## Dataset

[Xsurv.csv](Xsurv.csv) contains 320 rows and 200 variables, excluding the first column used as the row identifier.

| Variable | Description |
|---|---|
| `Stage` | Target: 1 = low stage; 2 = high stage, as defined in the assignment |
| `AGE` | Supplied age variable; units and preprocessing require confirmation |
| `SEX` | Codes 1 and 2; male/female mapping is not documented in the supplied files |
| 197 `cg…` columns | CpG measurements; the original transformation and filtering require confirmation |

There are 157 low-stage and 163 high-stage observations. The analysis recodes them to 0 and 1 internally, with high stage as the positive class. It uses 256 development patients and 64 test patients, with a stratified split and seed 42.

The CSV contains no survival-time or event-status columns. This project evaluates stage classification.

## Run the complete analysis in Colab

1. Click the Colab button for `CpG_step_by_step.ipynb`.
2. Connect to a standard CPU runtime. A GPU is not required by this workflow.
3. Add a code cell at the top and run:

```python
from google.colab import files
uploaded = files.upload()  # Select Xsurv.csv from this repository.
```

4. Confirm the uploaded file is named exactly `Xsurv.csv`. Leave `DATA_PATH = None` in the setup cell. The notebook will look for the CSV in the current working directory.
5. Run the remaining cells in order. If an import fails, run this in a separate cell, then restart from the setup cell:

```python
%pip install numpy pandas scipy scikit-learn matplotlib seaborn
```

6. Read the explanations alongside the figures and tables. The nested cross-validation section takes longer than the exploratory sections.
7. Download the generated outputs before closing the runtime:

```python
import shutil
from google.colab import files

shutil.make_archive('Xsurv_results', 'zip', 'Xsurv_results')
files.download('Xsurv_results.zip')
```

To keep your edited notebook, save a copy in Google Drive or download the `.ipynb` file. Runtime data and generated output files are separate from the notebook.

### Run the original starter notebook

Upload `Xsurv.csv` using the same Colab upload cell. In `project_Xsurv.ipynb`, replace:

```python
df = pd.read_csv("../data/Xsurv.csv", index_col=0)
```

with:

```python
df = pd.read_csv("Xsurv.csv", index_col=0)
```

The original relative path does not match this repository's root-level CSV layout. Apply the same change when running the starter locally from the repository root.

## Run locally

Download and extract the repository, or clone it:

```bash
git clone https://github.com/UmairSeemab/CRG-groupProject.git
cd CRG-groupProject
```

Open a terminal in the repository directory and install the dependencies:

```bash
python -m pip install jupyterlab numpy pandas scipy scikit-learn matplotlib seaborn
python -m jupyterlab
```

Open `CpG_step_by_step.ipynb` and run all cells from the top. Keep `Xsurv.csv` beside the notebook, or set `DATA_PATH` to its location. In the starter notebook, select an available Python kernel if its original `conda_py312` kernel is unavailable.

The saved complete-analysis outputs record Python 3.12.14, NumPy 2.3.5, pandas 2.2.3, SciPy 1.17.0, scikit-learn 1.8.0, Matplotlib 3.10.8 and seaborn 0.13.2. Other environments have not been verified here. The setup cell prints your installed versions, and results may vary across versions.

## Workflow and group responsibilities

All members should understand the setup cell and use the same split and configuration.

| Section | Task | Owner |
|---|---|---|
| 1 | Load and audit the dataset | Member 1 |
| 2 | Reserve the test set before exploration | Member 1 |
| 3 | Explore development patients: distributions, correlations and PCA | Member 1 |
| 4 | Define models and run nested validation | Member 2 |
| 5 | Lock model and panel choices before testing | Member 2 |
| 6 | Assess marker stability and interpret coefficients | Member 2 |
| 7 | Evaluate locked models on held-out patients | Member 3 |
| 8 | Add verified biological annotation when available | Member 3 |
| 9 | Export results and summarize evidence | Member 3 |

Member 3 also helps review validation code. All members review the final interpretation together.

## Modeling and evaluation

- Compare clinical-only (`AGE`, `SEX`), CpG-only and combined predictor sets.
- Evaluate elastic-net logistic regression and random forest.
- Compare 5-, 10- and 20-CpG panels selected by training-only ANOVA F scores and modeled with L2 logistic regression.
- Use five outer folds and three inner folds for nested cross-validation.
- Fit scaling and feature selection inside training folds.
- Select the smallest panel within 0.02 mean cross-validation AUC of the best compact panel. This is a practical selection rule, not proof of equivalent performance.
- Use a fixed classification threshold of 0.5.
- Report ROC-AUC, balanced accuracy, sensitivity, specificity, precision–recall curves and a confusion matrix.
- Estimate conditional test uncertainty with 2,000 stratified bootstrap resamples.

PCA is exploratory and is not used as input to the classifiers. Marker-selection frequency describes consistency across five overlapping training sets; it is not a probability that a marker is biologically valid.

## Results in the saved notebook

The development-based selection rule chose a 20-CpG panel. Its held-out ROC-AUC was **0.609**, with a conditional 95% bootstrap interval of **0.467–0.743**.

The interval includes chance-level AUC of 0.5, so these results do not establish reliable stage prediction. The interval reflects test-sample uncertainty conditional on the fitted model, not all uncertainty in data preparation and model selection. These markers remain exploratory candidates.

The test results have already been inspected. Further choices motivated by these results should be reported as exploratory; fresh independent data would be needed for a new final assessment.

## Generated outputs

Running the complete notebook creates `Xsurv_results/`, including:

| Output | Contents |
|---|---|
| `01_split.png` through `10_test_intervals.png` | Ten figures covering exploration, model comparison, markers and test performance |
| `data_audit.csv` | Dataset checks |
| `split_manifest.csv` | Development/test membership |
| `high_correlations_development.csv` | Highly correlated CpG pairs in development data |
| `nested_cv_folds.csv`, `nested_cv_summary.csv` | Fold-level and summarized model results |
| `marker_ranking.csv` | CpG selection frequencies and elastic-net summaries |
| `final_panel.csv` | Selected CpGs, coefficients and annotation status |
| `test_metrics.csv`, `test_intervals.csv` | Held-out estimates and uncertainty intervals |
| `test_predictions.csv` | High-stage probabilities for test patients |
| `analysis_manifest.json` | Settings, selected markers, model parameters, software versions and input checksum |

These files are generated on execution and are not included in the supplied repository archive. Subsequent runs overwrite files with the same names in this output directory.

## Optional CpG annotation

Gene mapping is unresolved until the assay platform and matching annotation manifest are verified. To enable the merge, place `CpG_annotation.csv` in the notebook's working directory, with these columns:

```text
CpG,Gene,Chromosome,Position,Genome_build,Annotation_source
```

Use one row per CpG. Preserve multiple gene mappings within the `Gene` field. In Colab, upload this file before running Section 8. The notebook merges supplied annotations but does not independently verify their correctness. Without this file, it exports unresolved annotations and continues.

## Scientific background

The original assignment cites Li et al., *Efficient gradient boosting for prognostic biomarker discovery*, Bioinformatics 38(6), 1631–1638 (2022): [article and DOI](https://doi.org/10.1093/bioinformatics/btab869).

That study introduces Xsurv for survival analysis and uses a melanoma methylation example. This educational project addresses the assignment's stage-classification question. It does not reproduce the article's survival analysis or directly compare performance with its survival models. The exact steps used to derive the supplied CSV still require verification.

Additional resources:

- [Xsurv repository cited by the assignment](https://github.com/wanglab1/Xsurv)
- [scikit-learn: avoiding data leakage](https://scikit-learn.org/stable/common_pitfalls.html)
- [scikit-learn: nested cross-validation](https://scikit-learn.org/stable/auto_examples/model_selection/plot_nested_cross_validation_iris.html)
- [Google Colab GitHub integration example](https://colab.research.google.com/github/googlecolab/colabtools/blob/main/notebooks/colab-github-demo.ipynb)
