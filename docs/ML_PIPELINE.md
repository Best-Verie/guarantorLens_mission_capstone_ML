# GuarantorLens - ML pipeline documentation

A stage-by-stage guide to the machine-learning pipeline in `notebooks/pipeline.ipynb`. It is written so
you can understand and defend every step. It is reference material for you, not text to paste into the
dissertation - write the report in your own words.

- **Notebook:** `notebooks/pipeline.ipynb` (the single, explained pipeline; `train.ipynb` is kept as a backup).
- **How to run:** on Colab, upload `Normal.xlsx` and `Written Off.xlsx`, then Run all. Model pickles are made
  with scikit-learn 1.6.1 so the backend (same version) can load them; a version mismatch makes the app fall
  back to rules.
- **Outputs:** everything is written to `models/guarantorlens_new/` and zipped at the end. Two model bundles
  come out: `guarantorlens_serving.joblib` (the risk classifier) and `guarantorlens_extra.joblib`
  (segmentation + anomaly).
- **Numbers below** are from a representative run; the exact figures come from the Colab run. When you re-run,
  refresh any quoted metric. The **search depth** is `N_ITER = 15` experiments per model (a knob near the top
  of the training cell).

---

## 1. Merge the two files into one labeled dataset

Two source files: `Normal.xlsx` (good loans) and `Written Off.xlsx` (defaults). Each has a sheet per year
(2025, 2026) and one row per **(loan, guarantor)** pair. The pipeline:

1. reads every sheet of both files and stacks them,
2. tags the **label by file** - a row from `Written Off.xlsx` is `1`, from `Normal.xlsx` is `0`,
3. if a loan appears in both files, the **write-off wins** (a recovered write-off is still a default),
4. collapses to **one row per loan** with its list of guarantors.

Result: **11,015 loans, 226 written off (2.1% "bad rate"), 11 branches, disbursed 2022-2023.** Saved as
`00_merged_dataset.csv`. This single table is the input to everything else.

Why file-based labels and not the `Repayment Status` column: that column is the *recovery* state after the
decision (Repaid / Active). A loan that was written off and later partly repaid is still a default, so the
file is the honest label and the status column is never used as the target.

## 2. The target label, and the leakage rule

- **Target:** `label` = 1 if written off, else 0. This is a **binary classification** problem.
- **Leakage rule:** some columns only exist *after* a loan is decided - days in arrears, unpaid amounts,
  last payment date, repayment status. Using them would be cheating (the model would "know the future"), so
  they are dropped. We keep only what an officer knows **on the day the loan is granted**.

## 3. Feature engineering

Two groups of features, all computed **as-of** the loan's disbursement date (only earlier loans are ever
looked at, never later ones, so nothing leaks):

**Borrower / loan features** (what the applicant looks like at origination): `log_amount`, `savings`,
`salary`, `interest_rate`, `loan_to_savings`, `loan_to_salary`, `account_age`, and as-of behavioural
history (`b_prior_loans`, `b_prior_writeoff`, `b_prior_arrears_rate`, `b_recent_loans`).

**Guarantor-network features** (the project's distinctive part). The guarantee relationships form a **graph**:
every loan links a borrower to the members who guarantee it.

- We build **communities** with a **union-find** (connected components): everyone tied together through
  guarantees ends up in one group.
- `community_prior_default_rate` = the past write-off rate of the borrower's community, computed as-of (only
  loans dated before this one count). This is the strongest single network feature (single-feature AUC ~0.73).
- Other network features: `n_guarantors`, guarantors' mean savings/salary, the share of guarantors who had a
  prior default (`g_prior_default_rate`), their prior-arrears rate (`g_prior_arrears_rate`), and the
  guarantor-to-borrower savings ratio.

"As-of" is the key idea: for each loan we rebuild each member's history using only earlier loans, so a
feature can never peek at what happened after the loan was granted.

## 4. Cohort and leakage screen

- **Cohort:** all loans are 2022-2023 (the matured window - old enough to have shown whether they went bad).
- **Leakage screen:** every feature is checked for a "hidden clock". If a feature correlates too strongly with
  the disbursement date (|correlation| > 0.40), it is really a disguised time-signal and is dropped.
  `time_since_last_loan` fails this test and is removed. **18 features** are kept.

The screen is shown two ways (figure `01_leakage_evidence.png`): each feature's single-feature ROC-AUC, and a
scatter of AUC against date-correlation with the leaking zone shaded.

## 5. Exploratory data analysis (EDA)

- **Class imbalance:** defaults are only 2.1% of loans. This is why the headline metric is **PR-AUC** (which
  focuses on ranking the rare bad loans), not accuracy - a model that predicts "never defaults" would be 98%
  accurate and useless.
- **Separation:** loan-to-savings and interest rate visibly separate good from bad loans
  (`07_data_overview.png`).
- **Correlation heatmap** of the features (`07_feature_correlation.png`).
- **Extended EDA** (`07b_univariate.png`, `07c_bivariate.png`): each feature's distribution, and how it differs
  between written-off and normal loans.

## 6. Splitting the data fairly

A borrower can have several loans. If some of a borrower's loans were in train and others in test, the model
could "recognise" the borrower and cheat. So we split **by borrower** (a group split): each borrower is wholly
in train or wholly in test. Cross-validation uses `StratifiedGroupKFold` (5 folds, grouped by borrower,
keeping the rare defaults balanced across folds). A 25% borrower-grouped hold-out is set aside for SHAP.

## 7. The experiment engine (five model families, RandomizedSearch)

Five model families are trained: **logistic regression, decision tree, random forest, XGBoost, and a
feed-forward neural network** (a multi-layer perceptron - input layer, one or two hidden layers, output
layer; labelled `feed_forward_nn`).

- Instead of a fixed grid, each model draws its settings from **ranges** (distributions), and
  **`RandomizedSearchCV`** samples **`N_ITER` = 15** experiments per model. The number of experiments is a
  knob, not a hard-coded list - raise it for a deeper search, lower it for speed.
- Every model is run **twice**: borrower-only, and borrower + network. This makes the leaderboard show
  directly whether the network helps.
- Selection metric is **PR-AUC** (refit on it); every experiment is also scored on ROC-AUC, F1, precision,
  recall and accuracy.

Outputs: a **leaderboard** across all model/feature combinations (`02_leaderboard.png`,
`02_tuning_leaderboard.csv`) and, per model, the **full table of experiments** with the best row highlighted
(`02_grid_<model>.csv`). XGBoost + network wins (CV PR-AUC ~0.66).

## 8. Each model up close: hyperparameter table + confusion matrix + curves

- **Hyperparameter tables** (section "Choosing the best model"): for each family, every experiment with its
  metrics, best-first, best row highlighted.
- **Per-model diagnostics** (`02d_diag_<model>.png`): each family's **confusion matrix** at the >=80%-recall
  operating point, and its **learning curve** (CV PR-AUC as the training set grows).
- **Loss curves** (`09b_learning_curves.png`): XGBoost log-loss per boosting round (train vs validation - a
  small gap means it is not over-fitting), the MLP's training loss per epoch (the classic loss curve), and a
  learning curve vs data size (still rising means more data would help).
- **Best-per-family** table and the overall winner (`02_best_per_model.csv`).

## 9. Imbalance handling and the class-weight tuning

- **SMOTE vs class weighting** (`02b_imbalance.png`): we generate synthetic defaults with SMOTE and three
  variants, plus undersampling, and compare to plain class weighting on PR-AUC. Class weighting matches or
  beats every oversampler, so we keep it (oversampling mainly helps weak learners and hurts calibration; our
  model is a strong, calibrated tree).
- **Tuning `scale_pos_weight`** (`02c_scale_pos_weight.png`): the usual rule is `neg/pos` (~48 here), but that
  targets balanced accuracy at a 0.5 threshold, not ranking. Sweeping it shows PR-AUC is highest at
  **`scale_pos_weight = 1`** (no reweighting); the recall we want comes from the **operating threshold**, not
  from re-weighting. The tuned value is adopted for the deployed model (this raised the deployed PR-AUC by
  about +0.08 over the heuristic).

## 10. Does the guarantor network add signal? (and how it tunes XGBoost)

- **Contribution** (`05_roc_pr_curves.png`, `05_network_contribution.csv`): three models on the same data -
  borrower-only, network-only, and both. Network-only beats the base rate (PR ~0.24, ROC ~0.81), so the
  network carries **real signal on its own**. Adding it to the borrower features improves ROC (better
  ranking) but its *incremental* PR-AUC lift is small, and a bootstrap confidence interval on that lift
  crosses zero.
- **Robustness check:** to prove the small lift is genuine and not weak engineering, we (1) residualise each
  network feature against the borrower features and re-test only the leftover, and (2) add stronger network
  aggregations (worst guarantor, prior-defaulter count, dispersion, community size). Neither lifts PR-AUC
  beyond noise. The conclusion is **homophily** - risky borrowers cluster with risky guarantors, so the
  network largely re-states what the borrower features already say. The network is kept because it powers the
  flags, the contagion view, the fix-it advisor, and better ranking (ROC).
- **How the network tunes XGBoost:** the network features join the borrower features in the same XGBoost
  input; the search tunes the tree over the combined set, and two network features get monotone constraints
  (see section 12).

## 11. Calibration

Officers read the score as a probability, so the raw model output is **calibrated** with isotonic regression
(`CalibratedClassifierCV`, `05_calibration.png`). After calibration a predicted "10%" really does default
about 10% of the time. Calibration preserves the ranking, so it does not change which loans are flagged, only
what the number means.

## 12. Monotone constraints (making the model believable)

An unconstrained tree trained on few defaults can learn back-to-front relationships (e.g. "more savings ->
more risk", or a small loan against big savings scoring High), which no officer would trust. We **pin the
scrutinised features to their common-sense direction** and retrain:

- `savings` -> `-1` (more savings can only lower risk),
- `loan_to_savings`, `loan_to_salary`, `log_amount` -> `+1` (a bigger loan can only raise risk),
- `g_prior_arrears_rate`, `community_prior_default_rate` -> `+1` (a worse guarantor/community record can only
  raise risk),
- everything else `0` (let the data decide).

The set is deliberately small: constraining the dominant `interest_rate` (or the noisy prior-history
features) collapses accuracy on this data (PR ~0.60 -> ~0.34). The constrained model trades a little PR-AUC
(~0.66 -> ~0.62) for a model that always moves in a believable direction. **This is the model that is
deployed.**

## 13. Operating point, confusion matrices, and bands

- **Confusion matrices** are shown at a **>=80%-recall** operating point (catching a default matters more than
  an extra review): the unconstrained model (`03a`), after calibration (`03b`), the deployed monotone model
  (`03_confusion.png`), and a grid of all models (`06_confusion_grid.png`).
- **Why precision looks low:** at a 2.1% base rate, catching 80% of defaults necessarily means many false
  alarms. The operating-points table (`09_operating_points.png`) shows the same model reads precision ~0.6-0.7
  at an F1-optimal threshold - low precision is a property of the rare-event threshold, not a fault.
- **Bands:** the app's Low / Medium / High cut-offs are the **70th and 90th percentiles of the calibrated
  score** across the portfolio - so bands are relative to the book, separate from the evaluation threshold.

## 14. Explaining the model

- **SHAP** (`04_shap_summary.png`, `04_shap_importance.csv`): the per-feature contributions to each score.
  Interest rate leads, then the guarantor/community history.
- **Permutation importance** (`04b_permutation_importance.png`): a model-agnostic cross-check - shuffle each
  feature and measure the PR-AUC drop. It gives the same top-of-list order as SHAP, so the explanation does
  not depend on the model's internals.
- **Unsupervised check** (`08_unsupervised_vs_supervised.png`): anomaly detectors (Isolation Forest, LOF) with
  no labels score far below the trained model, confirming supervised learning is justified here.

## 15. The served model bundle

`guarantorlens_serving.joblib` contains: the calibrated, monotone XGBoost `model`; the `features` list; the
`bands` (medium/high percentiles); the training `medians` (to fill unknown fields at serve time); the
`network_features` list; and the headline `metrics`. The backend loads this and scores live applications.

Also exported for the app: `guarantorlens_members.json` and `guarantorlens_loans.json` (anonymised member and
loan tables the app reads).

## 16. The other models (a portfolio, not one classifier)

The classifier answers "will this loan default?". Three more model families answer different questions; all
are simple and leakage-safe, and they read the member/loan tables the pipeline just exported.

- **Network communities + contagion** (`10_*`): **Louvain** community detection on the guarantee graph
  (~1,366 communities, modularity ~0.99), the write-off rate inside each community (the riskiest is far above
  the 2.1% average), and a **contagion** trace - if a member defaults, every loan they guarantee is exposed,
  followed round by round. Powers the member/network view.
- **Survival analysis** (`11_*`): a **Kaplan-Meier** curve of the share of loans still performing as the
  months pass, split by loan size. Loans that are repaid or still running are **censored** (set aside, not
  counted as failures). About 98% still performing at 24 months; **large loans fail soonest**. Powers the
  Monitoring page.
- **Clustering + anomaly** (`12_*`, `guarantorlens_extra.joblib`): **KMeans** segments borrowers into a few
  types (silhouette ~0.33) - context, not risk. **Isolation Forest** scores how unusual an application is;
  the most-unusual 10% default ~3.4x more often, so it is a "look closer" flag. Both are bundled with their
  imputer / winsor bounds / scaler so the backend reproduces the exact preprocessing; the bundle powers the
  "Unusual application" flag on the assess result.

## 17. Worked test cases

The exported model is scored on worked cases to prove it behaves sensibly: a small loan against big savings is
Low; a big loan against thin savings is High; and monotone sweeps confirm that raising savings only lowers
risk, raising the loan only raises it, and a worse community record only raises it. These are the same checks
that fixed the deployed app.

## 18. Artifacts produced (in `models/guarantorlens_new/`, zipped at the end)

| File | What it is |
|---|---|
| `00_merged_dataset.csv` | the merged, labeled one-row-per-loan dataset |
| `01_leakage_evidence.png` | leakage screen (AUC + date-correlation) |
| `07_*`, `07b_*`, `07c_*` | EDA: overview, correlation, distributions, feature-by-outcome |
| `02_leaderboard.png`, `02_tuning_leaderboard.csv` | model-selection leaderboard |
| `02_grid_<model>.csv` | every experiment per model family |
| `02d_diag_<model>.png` | per-model confusion matrix + learning curve |
| `02b_imbalance.*`, `02c_scale_pos_weight.*` | imbalance strategy + class-weight tuning |
| `09b_learning_curves.png` | XGBoost/MLP loss curves + learning-vs-data-size |
| `05_*` | network contribution, ROC/PR curves, calibration |
| `03a/03b/03_confusion.png`, `06_*`, `09_operating_points.*` | confusion matrices and operating points |
| `04_shap_*`, `04b_permutation_*` | model explanation |
| `08_unsupervised_*` | unsupervised vs supervised check |
| `guarantorlens_serving.joblib` | the deployed risk classifier bundle |
| `guarantorlens_extra.joblib` | segmentation + anomaly bundle |
| `guarantorlens_members.json`, `guarantorlens_loans.json` | tables the app reads |
| `10_*`, `11_*`, `12_*` | network, survival, clustering/anomaly outputs |
| `run_log.txt` | every printed line from the run |

## Numbers to refresh after a Colab run

Re-running produces the canonical figures. Update any quoted number from the fresh
`models/guarantorlens_new/` outputs and `run_log.txt`. **Canonical values (Colab run "guarantorlens_new (9)",
scikit-learn 1.6.1):** XGBoost + network CV PR-AUC **0.659**; feature-set PR-AUC borrower-only **0.641** /
network-only **0.237** / both **0.653**; **deployed monotone PR-AUC 0.603 / ROC 0.932** at
`scale_pos_weight = 1` (heuristic 48 gives 0.594); deployed confusion at ≥80% recall `[[9784, 1005], [45,
181]]` (caught 181/226, 1005 false alarms); calibrated held-out PR-AUC 0.746; bands medium 0.008 / high
0.028; survival 97.97% at 24 months (large loans 97.13%); anomaly 3.4x; 1,366 communities at modularity
0.991; KMeans silhouette 0.330.
