# Autism Classification from Resting-State fMRI (ABIDE) — LR and SVM

UE24CS352A Machine Learning — Mini-Project

**Team:** [Jayanth K, PES1UG24AM123] · [Pradeep R, PES1UG24AM809]

## What this project does

We follow the pipeline of *"Classifying Adolescent Excessive Alcohol Drinkers from fMRI Data"* (Kim, Liu, Noh, CS229). That paper's NCANDA data is access-restricted, so, with our instructor's approval, we apply the same pipeline to the public **ABIDE** dataset and classify **autism vs typical control** from resting-state fMRI.

We compare two kinds of features built from each subject's brain-region time series:

| Feature set | What it is | Size (cc200 / HO) |
|---|---|---|
| **A: max − min** (from the paper) | Range of each region's z-scored signal | 200 / 111 |
| **B: correlation** (our modification) | Fisher-z correlation between every pair of regions | 19,900 / 6,105 |

and five models: Logistic Regression and SVM with linear, polynomial (degree 2), RBF and sigmoid kernels. Each is run with and without demographics (age, sex, site).

## Dataset

- ABIDE I, preprocessed (CPAC pipeline, no filtering, no global signal regression), downloaded with `nilearn.datasets.fetch_abide_pcp`.
- 871 subjects: 468 typical controls, 403 autism (majority-class baseline = 0.537).
- 20 scanning sites; time points per subject range from 78 to 316.
- Two parcellations: **Craddock 200** (`rois_cc200`, 200 regions) and **Harvard-Oxford** (`rois_ho`, 111 regions).
- Nothing needs to be downloaded by hand. The notebook fetches it and caches it in `~/nilearn_data`.

## Repository structure

```
.
├── README.md
└── main.ipynb      # the whole project: data loading, features, LR, SVMs, results
```

## Setup

Python 3.9 or newer is recommended.

```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

## How to run

Open the notebook and run all cells from top to bottom:

```bash
jupyter notebook main.ipynb
```

(or open it in VS Code / JupyterLab and choose *Run All*). The notebook follows four stages, in this order:

1. **Load data**: downloads all 871 subjects for both atlases. **The first run can take over an hour** (one small file per subject per atlas). Re-runs reuse the cache in `~/nilearn_data`. It prints the subject count, class counts, region counts, time-point range and site table.
2. **Build features**: builds the four feature matrices (`cc200_range`, `cc200_corr`, `ho_range`, `ho_corr`) and the demographics table (age, sex as 0/1, 20 one-hot sites).
3. **Logistic regression**: sets aside 15% of subjects as a dev set (to choose `C`), then runs 10-fold stratified cross-validation on the other 85%. Prints a results table. This is the slowest stage on the correlation features.
4. **SVMs**: computes each kernel matrix once (871 × 871) and evaluates the SVMs on it, which is far faster than refitting on 19,900 raw features.

## Method notes

- **Evaluation:** stratified 15% dev set (131 subjects) for choosing `C`; 10-fold stratified CV on the remaining 740 for the reported accuracy. `random_state=0` throughout.
- **Leakage control:** in the LR pipeline, `StandardScaler` sits inside the pipeline, so it is fitted on training folds only. The SVM kernels are built with fixed settings and no fitted scaler.
- **Demographics settings:** brain only, + demographics, + demographics without age, and demographics only.

## Results (10-fold CV accuracy, brain features only; baseline = 0.537)

| Features | LR | SVM linear | SVM poly2 | SVM RBF | SVM sigmoid |
|---|---|---|---|---|---|
| cc200 max − min | 0.541 | 0.515 | 0.541 | 0.516 | 0.538 |
| **cc200 correlation** | 0.666 | 0.659 | **0.703** | 0.677 | 0.538 |
| HO max − min | 0.485 | 0.473 | 0.469 | 0.535 | – |
| **HO correlation** | 0.650 | **0.680** | 0.658 | 0.642 | 0.538 |

Demographics only (LR): 0.530. Correlation features beat the paper's max − min features by roughly 12 to 16 points; max − min and demographics stay near chance. No single kernel wins consistently, and the sigmoid kernel fails to learn (it predicts the majority class).

## Limitations

- The dataset differs from the paper's, so our numbers are not directly comparable to theirs.
- The paper's ICA parcellation is not available as a ready-made file, so Harvard-Oxford was used as the second atlas.
- `C` was chosen on a small dev set, so differences of a few points between kernels are within noise.
- Scanner (site) effects were not tested with leave-one-site-out validation.

## Reference

Y. Kim, C. Liu, J. Noh, "Classifying Adolescent Excessive Alcohol Drinkers from fMRI Data", Stanford CS229 project report.
