# GuarantorLens pipeline — study guide (understand it, defend it)

This walks through **`notebooks/pipeline.ipynb`** in order and explains, for every stage: **what** it does,
**why** you chose it, **how to read** the figure it produces, and the **keywords** in plain words. It is a
learning aid for you (and for the viva) — write the report in your own words, don't paste from here.

Numbers quoted are the canonical Colab run. If a number ever differs from your latest run, trust the run.

---

## 0. The one-paragraph summary (say this if asked "what is your project?")

GuarantorLens predicts whether a SACCO loan will be **written off** (defaulted), and explains why, so a loan
officer can decide with more confidence. The twist is that loans are **guaranteed by other members**, so the
tool also studies that **guarantee network**. It is not one model — it is a **portfolio**: a main classifier
for the risk score, plus network, survival, clustering, and anomaly models that each answer a different
question.

---

## 1. The data and the label

**What.** Two Excel files — `Normal.xlsx` (good loans) and `Written Off.xlsx` (defaults) — each with a sheet
per year and **one row per (loan, guarantor)** pair. The pipeline merges them into **one row per loan** with
a list of that loan's guarantors and a **0/1 label**.

**The target label.** `label = 1` if the loan is in the write-off file, else `0`. Result: **11,015 loans,
226 written off = a 2.1% "bad rate"**, across 11 branches, disbursed 2022–2023.

**Why label by file, not by the `Repayment Status` column?** Repayment Status is the *recovery* state
(Repaid/Active) recorded **after** the decision. A loan that was written off and later partly repaid is still
a default. Using the status would leak the future. So the **file** is the honest label.

- **Keyword — label / target:** the thing you are trying to predict (here: default yes/no).
- **Keyword — bad rate / base rate:** the share of loans that are actually bad (2.1%). Everything is judged
  against this.
- **Keyword — class imbalance:** one class (defaults) is much rarer than the other. 2.1% is heavily imbalanced.

---

## 2. The golden rule: no leakage

**What.** Every feature is computed **"as-of" the day the loan was disbursed** — using only information that
existed *before* that day. For a member's history, only their *earlier* loans are looked at, never later ones.

**Why.** If a feature secretly contains future information ("**leakage**"), the model looks brilliant in
testing and then fails in production, because at decision time that information does not exist yet. Columns
like days-in-arrears, unpaid amounts and last-payment-date only exist *after* the loan is decided, so they are
**dropped**.

- **Keyword — data leakage:** the model accidentally sees information it would not have at prediction time.
  The single most common way ML results are secretly worthless. Avoiding it is why the numbers here are honest.
- **Keyword — as-of feature:** a feature calculated using only data available up to a cut-off date.

**The date-leak screen (figure `01_leakage_evidence.png`).** Even an as-of feature can hide a clock. We
measure each feature's correlation with the disbursement date; if `|correlation| > 0.40` the feature is a
disguised time-signal and is dropped (this is what removes `time_since_last_loan`). **18 features** survive.

**How to read `01_leakage_evidence.png`:** left panel = each feature's standalone predictive power (bar length
= how well it alone separates good/bad, 0.5 = useless). Right panel = predictive power vs date-correlation;
anything in the shaded red zones (|corr| > 0.4) is a suspected leak and gets cut.

---

## 3. Feature engineering — borrower and network

**What.** For each loan we build two groups of features:

- **Borrower/loan features** (what the applicant looks like): `log_amount`, `savings`, `salary`,
  `interest_rate`, `loan_to_savings`, `loan_to_salary`, `account_age`, and as-of history (prior loans, prior
  write-off, prior arrears rate).
- **Guarantor-network features** (the distinctive part): the guarantee links form a **graph** — each loan
  connects a borrower to their guarantors.

**The community feature (why union-find).** We group everyone tied together through guarantees into
**communities** using **union-find** (a simple algorithm that merges connected people into the same group).
Then `community_prior_default_rate` = the past write-off rate of the borrower's community, as-of. This is the
strongest single network feature.

- **Keyword — feature:** an input number the model uses (e.g. loan-to-savings).
- **Keyword — graph / node / edge:** a network of dots (nodes = members) joined by lines (edges = guarantees).
- **Keyword — union-find (connected components):** an algorithm that finds "who is connected to whom" and
  puts each connected cluster in one group.
- **Keyword — `log_amount`:** the loan amount run through a logarithm, so a few huge loans don't dominate.
- **Keyword — `loan_to_savings` / `loan_to_salary`:** the loan size relative to the borrower's savings/salary
  — how stretched the borrower is.

---

## 4. Why these metrics (the most important concept to defend)

Defaults are 2.1% of loans. A model that always says "good" is **98% accurate and completely useless**. So
**accuracy is the wrong metric**. We use:

- **PR-AUC (Precision–Recall Area Under Curve)** — the **headline**. It measures how well the model ranks the
  rare bad loans above the good ones. Baseline (random) = the bad rate (0.021). Our deployed model = **0.603**
  — about **29× the baseline**.
- **ROC-AUC** — probability that a random bad loan is scored higher than a random good loan. 0.5 = coin flip,
  1.0 = perfect. Deployed = **0.932**. (ROC-AUC looks flattering under imbalance, which is why PR-AUC leads.)
- **Precision** — of the loans we flag, what share are actually bad.
- **Recall (sensitivity)** — of the actually-bad loans, what share we catch.
- **F1** — the balance (harmonic mean) of precision and recall.

**The precision–recall trade-off (defend this).** You cannot maximise both. At a 2.1% base rate, catching most
defaults (high recall) forces you to also flag many good loans (low precision). We deliberately choose a
**recall-first** operating point: missing a default costs the SACCO more than an extra review.

- **Keyword — precision:** "when I raise the alarm, how often am I right?"
- **Keyword — recall:** "of all the real problems, how many did I catch?"
- **Keyword — PR-AUC:** overall skill at ranking rare positives; our main score.
- **Keyword — ROC-AUC:** overall ranking skill; less reliable under heavy imbalance.

---

## 5. The fair split (grouped cross-validation)

**What.** A borrower can have several loans. We keep **each borrower entirely in train OR test**, never split
across both. Cross-validation uses **StratifiedGroupKFold** (5 folds, grouped by borrower, keeping the rare
defaults balanced in each fold).

**Why.** If a borrower's loan A is in training and loan B is in testing, the model can "recognise" the
borrower and cheat — inflating results. Grouping by borrower prevents this.

- **Keyword — cross-validation (CV):** train on part of the data, test on the held-out part, rotate, average.
  A more honest estimate than a single split.
- **Keyword — fold:** one of the rotations (we use 5).
- **Keyword — grouped split:** all rows of the same group (borrower) stay together.
- **Keyword — stratified:** each fold keeps the same good/bad ratio, so no fold is missing the rare class.
- **Keyword — out-of-fold (OOF) prediction:** each loan is scored by a model that did **not** train on it —
  an honest prediction for every loan.

---

## 6. Training five model families + the experiment engine

**What.** Five model types are trained: **Logistic Regression, Decision Tree, Random Forest, XGBoost, and a
Feed-forward Neural Network (MLP)**. Each is tried **twice** — borrower-only and borrower+network.

**Why five?** To justify the choice: you show XGBoost wins on merit, not by assumption. Each family is a
different way of drawing the boundary between good and bad.

**The experiment engine — RandomizedSearchCV (`N_ITER = 15`).** Instead of hand-picking settings
("hyperparameters"), each model draws settings from **ranges**, and we run **15 random experiments per model**,
keeping the best by PR-AUC. `N_ITER` is a knob — raise for a deeper search, lower for speed.

- **Keyword — model family:** a type of model (tree, linear, neural net…).
- **Keyword — hyperparameter:** a setting you choose before training (tree depth, learning rate, etc.),
  as opposed to what the model learns.
- **Keyword — grid search vs randomized search:** grid tries every combination (slow); randomized samples N
  combinations from ranges (faster, and just as good for finding a strong setting).
- **The five, one line each:**
  - **Logistic Regression** — a weighted sum passed through an S-curve; simple, linear baseline.
  - **Decision Tree** — a flowchart of yes/no splits; interpretable but unstable alone.
  - **Random Forest** — hundreds of trees voting; robust.
  - **XGBoost** — trees built one after another, each fixing the last one's mistakes ("gradient boosting");
    strongest on tabular data, supports monotone constraints. **This is what we deploy.**
  - **Feed-forward Neural Network (MLP)** — layers of weighted sums with non-linear activations; needs more
    data than we have, so it underperforms here.

**How to read the leaderboard (`02_leaderboard.png`):** one bar per model×feature-set, longer = better
PR-AUC; the dashed line is the base rate. Orange bars use the network, blue don't. XGBoost+network is longest.

**How to read a per-model hyperparameter table (`02_grid_<model>.csv`):** each row is one experiment (one
settings combination) with its scores; the best row (rank 1) is what that family contributes to the
leaderboard.

---

## 7. Reading the diagnostic graphs

**Confusion matrix** (`02d_diag_<model>.png`, `03_confusion.png`). A 2×2 count of predictions vs reality:

|  | Predicted good | Predicted bad |
|---|---|---|
| **Actually good** | TN (true negative) | FP (false positive = false alarm) |
| **Actually bad** | FN (false negative = missed default) | TP (true positive = caught default) |

Deployed model at ≥80% recall: `[[TN 9784, FP 1005], [FN 45, TP 181]]` — it **caught 181 of 226** defaults
(recall 0.80) at the cost of **1005 false alarms**. Read it as: "we catch 4 of every 5 defaults, and for each
real default we flag we also flag about 5 good loans for review."

- **Keyword — false positive (FP):** a good loan wrongly flagged (an extra review — cheap).
- **Keyword — false negative (FN):** a bad loan missed (expensive — a write-off slips through).
- **Keyword — threshold / operating point:** the score cut-off for calling a loan "bad". Lower threshold →
  more recall, less precision. We set it to reach 80% recall.

**Learning curve** (right panel of `02d_diag_*`, and `09b_learning_curves.png` panel 3). PR-AUC vs how much
training data is used. If the **CV line is still rising** at full data, **more data would help**. The gap
between the train line and the CV line shows overfitting (big gap = overfit).

**Loss curves** (`09b_learning_curves.png`, panels 1–2). "Loss" = how wrong the model is; training tries to
push it down.
- XGBoost log-loss per boosting round, **train vs validation**: both fall and stay close → **not
  overfitting** (a big gap would mean it memorised the training data).
- Neural-net loss per epoch: falls and flattens → it **converged** (finished learning).

**ROC and PR curves** (`05_roc_pr_curves.png`).
- ROC: recall (true-positive rate) vs false-positive rate. Higher/left-er = better; the diagonal = random.
- PR: precision vs recall. Higher = better; the flat dashed line = the base rate. Compare borrower-only /
  network-only / both — you can *see* network-only beat the baseline but add little on top of borrower.

- **Keyword — loss:** a number measuring prediction error; lower is better.
- **Keyword — epoch:** one full pass over the training data (neural-net term).
- **Keyword — boosting round:** one more tree added in XGBoost.
- **Keyword — overfitting:** the model memorises training quirks and fails on new data.
- **Keyword — convergence:** training has stopped improving; the model has settled.

---

## 8. Handling the imbalance

**SMOTE vs class weighting (`02b_imbalance.png`).** Two ways to stop the model ignoring the rare class:
- **Class weighting** — tell the model a default matters more.
- **SMOTE and variants** — invent synthetic defaults to balance the data.
Result: **class weighting matched or beat every SMOTE variant** (PR 0.593 vs 0.543 and below), so we use it —
oversampling mainly helps weak learners and can distort probabilities.

**Tuning `scale_pos_weight` (`02c_scale_pos_weight.png`).** This is *how much* to up-weight the rare class in
XGBoost. The textbook rule is `neg/pos` (~48 here), but that targets balanced accuracy, not ranking. Sweeping
it shows PR-AUC is **highest at 1 (no reweighting)** and falls as the weight rises (0.653 at 1 → 0.594 at 48).
So we adopt **1**; the recall we want comes from the **threshold**, not from reweighting. This was the single
biggest model improvement.

- **Keyword — SMOTE:** Synthetic Minority Over-sampling — creates fake examples of the rare class.
- **Keyword — class weight / `scale_pos_weight`:** how strongly the model is told to care about the rare class.

**How to read `02c`:** x-axis = the weight (log scale), y-axis = PR-AUC; the line slopes **down**, and the
dashed line marks the old heuristic (48). The peak is at the far left (weight = 1).

---

## 9. Does the guarantor network actually help? (the honest finding)

**What.** Three models on the same data — borrower-only, network-only, both — plus a robustness check.

**Findings.**
- **Network-only** scores PR 0.237 / ROC 0.827 — far above the 0.021 baseline, so the network **carries real
  signal on its own**.
- **Adding** it to the borrower features lifts ROC (0.942 → 0.952, better ranking) but its **incremental
  PR-AUC lift is within noise** (bootstrap lift +0.012, 95% CI [-0.019, +0.044] — the interval crosses zero).
- **Why:** **homophily** — risky borrowers tend to be guaranteed by risky people, so the network largely
  re-states what the borrower features already say. A robustness check (residualising the network against the
  borrower features, and adding stronger network aggregations) confirms it — the extra network signal is
  genuinely small, not under-engineered.
- **So why keep the network?** It still powers the **flags**, the **contagion** view, the **fix-it advisor**,
  the **community rate**, and **better ranking (ROC)** — real product value even if the incremental PR is small.

- **Keyword — homophily:** "birds of a feather" — similar (here, risky) people cluster together.
- **Keyword — bootstrap:** re-sample the data many times to get a range (confidence interval) on a number.
- **Keyword — confidence interval (CI):** the plausible range for a result. If it **crosses zero**, the effect
  is not statistically distinguishable from nothing.
- **Keyword — residualisation:** remove the part of a feature that the borrower features can already explain,
  then test only what's left over.

---

## 10. Calibration (making the score an honest probability)

**What.** The raw model output is passed through **isotonic calibration** so that a predicted "10%" really
does default about 10% of the time.

**Why.** Officers read the score as a probability. An uncalibrated score ranks correctly but the *number* can
be misleading. Calibration fixes the number without changing the ranking.

**How to read `05_calibration.png`:** x-axis = predicted probability, y-axis = observed default rate. The
dashed diagonal is perfect calibration; the closer the model's line hugs it, the more trustworthy the numbers.

- **Keyword — calibration:** aligning predicted probabilities with real-world frequencies.
- **Keyword — isotonic regression:** a flexible, monotonic mapping used to calibrate (only ever bends the
  curve up or flat, never down).
- **Keyword — reliability curve:** the calibration plot above.

---

## 11. Monotone constraints (making the model believable)

**What.** With few defaults, a free tree can learn back-to-front rules ("more savings → more risk"). We **pin
certain features to their common-sense direction** and retrain: `savings → -1` (more savings only lowers
risk); `loan_to_savings, loan_to_salary, log_amount → +1` (a bigger loan only raises risk);
`g_prior_arrears_rate, community_prior_default_rate → +1`. Everything else is free (`0`).

**Why.** A loan officer must trust the tool. A model that says a small loan against big savings is high-risk
gets ignored. The cost is honest: PR-AUC drops **0.653 → 0.603** and false alarms roughly **double** (1005 vs
544) — we accept it for guaranteed believable behaviour. We keep the set **small** because constraining the
dominant `interest_rate` collapses accuracy (PR 0.60 → 0.34).

- **Keyword — monotone constraint:** a rule forcing "more of X only ever moves risk one way".
- **Keyword — +1 / -1 / 0:** raise-only / lower-only / free.

---

## 12. Operating point and risk bands

- **Operating point (`09_operating_points.png`).** The same model at three thresholds: recall-first
  (screening), F1-optimal (balanced), precision-first. It shows precision looks low **only** at the
  recall-first cut — at an F1-optimal cut the same model reads precision ~0.8. Low precision at high recall is
  inherent to a 2.1% base rate, not a fault.
- **Risk bands.** Low / Medium / High in the app are the **70th and 90th percentiles of the calibrated score**
  across the portfolio — so "High" means "top 10% riskiest of the book", a relative, stable cut.

- **Keyword — percentile:** the value below which a given % of the data falls (90th percentile = top 10%).

---

## 13. Explaining the model (XAI)

- **SHAP (`04_shap_summary.png`).** For each loan, SHAP splits the score into per-feature contributions —
  how much each feature pushed this score up or down. Top drivers: interest_rate, salary, loan_to_salary,
  account_age, then guarantor/community history.
- **Permutation importance (`04b_permutation_importance.png`).** A model-agnostic check: shuffle a feature and
  see how much PR-AUC drops. Same top order as SHAP → the explanation doesn't depend on the model's internals.

**How to read the SHAP summary:** each dot is one loan; position left/right = pushed score down/up; colour =
the feature's value (high/low). Features are ordered by overall importance (top = most important).

- **Keyword — SHAP:** a method that fairly attributes a prediction to its features (from game theory).
- **Keyword — explainability / XAI:** making a model's decisions understandable to a human.
- **Keyword — permutation importance:** importance measured by "how much worse is the model if I scramble this
  feature?".

---

## 14. The unsupervised check — and the two roles of Isolation Forest

This is the part you said you don't understand, so here it is slowly.

**Supervised vs unsupervised.**
- **Supervised** learning uses the **labels** (we know which loans defaulted) to learn the good/bad boundary.
  Our classifier is supervised.
- **Unsupervised** learning uses **no labels** — it looks for structure/oddness in the data itself.

**Why test unsupervised at all?** Because defaults are rare, a fair question is: *do we even need labels — can
an "outlier detector" find the bad loans just by spotting weird ones?* We test two:
- **Isolation Forest** — isolates each point with random splits; points that are easy to isolate are
  **anomalies** (unusual).
- **Local Outlier Factor (LOF)** — flags points that sit in a sparser neighbourhood than their neighbours.

**Result (`08_unsupervised_vs_supervised.png`).** As **standalone risk models** they lose badly: Isolation
Forest ROC 0.889 / PR **0.232**, LOF 0.598 / **0.031**, versus the supervised **0.952 / 0.653**. So **unusual ≠
bad**; you need the labels. This **justifies the supervised path.**

**The important nuance (the reviewer's point).** The *same* Isolation Forest is then **reused in a different
role** — not to score risk, but as a **complementary "unusual application" flag**. It never replaces the
classifier; it just marks atypical loans for a second look, and those flagged loans do default **3.4× more
often** (6.9% vs 2.1%). So: **rejected as a predictor, kept as a complement.** Both statements are true of the
same tool used two ways.

- **Keyword — supervised / unsupervised:** with labels / without labels.
- **Keyword — anomaly / outlier:** a data point that doesn't fit the usual pattern.
- **Keyword — Isolation Forest:** an anomaly detector; unusual points get isolated in fewer random splits.
- **Keyword — contamination:** the expected share of anomalies you tell the detector to assume (we set it to
  the 2.1% bad rate).

**How to read `08_unsupervised_vs_supervised.png`:** bars for each method's PR-AUC and ROC-AUC; the supervised
bar towers over the two unsupervised ones — that's the whole point.

---

## 14b. Model vs analysis — and why the unsupervised ones are "configured", not "tuned"

A question that trips people up: are all these things "models", and do they all have training,
hyperparameters, tuning and accuracy like the classifier? Be precise.

**Trained models (learn a pattern, then score a new input) — three:**
- **XGBoost** (supervised) — the risk score.
- **KMeans** (unsupervised) — borrower segments.
- **Isolation Forest** (unsupervised) — the anomaly flag.

**Analyses (summarise the data or its structure; they do NOT score a new loan) — two:**
- **Kaplan-Meier survival** — a statistical estimator (a formula computed over the data).
- **Louvain communities + contagion** — graph algorithms.

So "survival is not really a model" is **correct** — it's an analysis.

**Do the unsupervised models train / have hyperparameters / metrics?**
- **Train?** Yes — KMeans learns cluster centres; Isolation Forest builds its trees. They fit to data.
- **Hyperparameters?** Yes — KMeans `k = 4`; Isolation Forest `n_estimators = 200`, `contamination = 2.1%`.
- **Tuned?** **No.** Tuning means optimising a hyperparameter against a *labelled* performance metric (PR-AUC via
  cross-validation). With no labels there is nothing to optimise, so you **configure** them — by domain sense
  (k = 4 is interpretable), an internal quality score (silhouette), or a review budget (flag the top 10%).
- **Metrics?** **No accuracy** (no ground truth to compare to). Instead a *quality* score (silhouette 0.33 for
  clustering) or *external validation* if outcomes happen to exist (our anomaly flag: the top-10% default 3.4×
  more often).

**What is special about our case:** we DO have the default labels. The unsupervised models still never use them
to *learn* (that is what makes them unsupervised), but we can **validate** them against outcomes afterward — which
is why we can quote silhouette, each segment's write-off rate, and the 3.4× anomaly lift. That is *checking*, not
*training*.

**The two saved bundles:**
- `guarantorlens_serving.joblib` = the XGBoost classifier (+ calibration, bands, medians, metrics).
- `guarantorlens_extra.joblib` = the two unsupervised models (KMeans + Isolation Forest) with their preprocessing.
- Survival and communities are **not saved** — the backend computes them live from the loan/member tables.

**One-line viva answers:**
- *"Is survival a model?"* No — a statistical estimator (Kaplan-Meier), an analysis of the portfolio, not a
  per-loan predictor.
- *"Do your unsupervised models have accuracy?"* No — accuracy needs labels; they are judged by quality
  (silhouette) and validated against outcomes (3.4× anomaly lift).
- *"Why not tune k?"* There is no labelled metric to tune against; k = 4 is chosen for interpretability and
  checked with silhouette.

## 15. The additional models (the "portfolio") — slowly, with how to read each

These ship alongside the classifier and each answer a **different question**. They are in the notebook's
"Additional models" section.

### 15a. Network communities + contagion (`10_*`)

- **What.** Study the guarantee **graph** itself. **Louvain** community detection automatically finds
  tight-knit groups who guarantee each other; **modularity** (0–1) measures how cleanly the network splits
  (we get ~1,366 communities, modularity **0.991** = very clean groups). Then we read each group's write-off
  rate — some groups are far riskier than the 2.1% average.
- **Contagion.** If a member defaults, every loan they guarantee is **exposed**. We follow that exposure round
  by round: one default's reach grows **217 → 310 → 349 → 367 → 377** members over four rounds. One-hop
  exposure: 173 loans (~RWF 364M) are backed by an already-written-off member.
- **How to read the graphs:** `10_community_default.png` = bar chart, each bar a community's write-off rate vs
  the dashed portfolio average (tall bars = risky groups). `10_contagion.png` = members reached vs
  propagation round (how far trouble spreads). `10_network_sample.png` = a picture of one risky community
  (red dots = written-off members).
- **Keywords — Louvain:** an algorithm that finds communities in a network. **Modularity:** how well-separated
  those communities are. **Contagion/cascade:** how a default's exposure spreads through guarantees.

### 15b. Survival analysis (`11_*`)

- **What.** Classification asks "will it default?"; survival asks "**how long** does a loan last before it is
  written off?". The **Kaplan-Meier** curve shows the share of loans **still performing** as months pass.
- **Censoring.** A loan that is repaid or still running has **not** failed, so it is **censored** — set aside
  rather than counted as bad. That's why the curve is a *survival share*, not a default rate.
- **Numbers.** ~**98% still performing at 24 months**; **large loans fail soonest** (97.13% vs ~98.5% for
  small/medium). We cap at 24 months because every loan has had that long to show its behaviour.
- **How to read `11_survival_km.png`:** x-axis = months on book; y-axis = share still performing; each line
  starts at 100% and steps **down** a little each time a loan in that group is written off. A **lower** line =
  riskier group. The size tiers let you compare small/medium/large.
- **Keywords — survival analysis:** modelling time-until-an-event. **Kaplan-Meier:** the standard step-curve
  estimate. **Censoring:** an observation that hasn't had the event yet, handled correctly rather than dropped.
  **Months on book:** how long since disbursement.

### 15c. Clustering / segmentation (`12_*`, Table 3.7)

- **What.** **KMeans** groups borrowers into **k = 4 "types"** from loan size, savings, salary, loan-to-savings
  and number of guarantors — **without** the default label (unsupervised).
- **Quality — silhouette 0.33.** The **silhouette** score (−1…1) measures how well-separated the clusters are;
  0.33 = moderate.
- **Honest finding.** The four segments' write-off rates sit in a tight **1.8–2.8%** band — they separate
  borrowers by **size and wealth**, not really by risk. So segmentation is **context, not a risk score**, and
  it's kept in the notebook/report, not the UI.
- **How to read `12_cluster_scatter.png`:** the 5 features are squashed to 2 dimensions with **PCA** (a
  compression that keeps the biggest differences) and plotted; colour = cluster. Well-separated colour blobs =
  distinct types.
- **Keywords — KMeans:** groups points into k clusters by nearest centre. **Silhouette:** cluster-separation
  score. **PCA (Principal Component Analysis):** squashes many features into a few for plotting. **Winsorise:**
  clip extreme values to the 1st/99th percentile so one outlier can't dominate. **Standardise:** rescale each
  feature to a common scale so no feature dominates by unit size.

### 15d. Anomaly flag (`12_*`)

- **What.** The Isolation Forest from §14, in its "unusual application" role. We flag the **most-unusual 10%**;
  they default **3.4× more** than average — a cheap first screen.
- Bundled (with its imputer/winsor/scaler) into `guarantorlens_extra.joblib` and served as the "Unusual
  application" badge.

---

## 16. Test cases (does the final model behave sensibly?)

**What.** We score worked cases through the exported model: a small loan against big savings → **Low**; a big
loan against thin savings → **High**; and **monotone sweeps** confirm raising savings only lowers risk, raising
the loan only raises it, a worse community record only raises it. These are the same checks that fixed the
deployed app.

**Why.** Good metrics aren't enough — the model must also be **directionally sane** on obvious cases, or
officers won't trust it. This is the practical proof of the monotone constraints.

---

## 17. What gets exported (the two bundles)

- **`guarantorlens_serving.joblib`** — the deployed classifier: the calibrated, monotone XGBoost, its feature
  list, the risk bands, the training medians (to fill unknown fields at serve time), and the metrics.
- **`guarantorlens_extra.joblib`** — the segmentation (KMeans) + anomaly (Isolation Forest) with their
  preprocessing, for the "Unusual application" flag.
- Plus `guarantorlens_members.json` / `guarantorlens_loans.json` — the anonymised tables the app reads.
- **Version rule:** pickled with **scikit-learn 1.6.1** to match the backend; a version mismatch makes the app
  fall back to rules.

---

## 18. Glossary (quick reference)

| Term | Plain meaning |
|---|---|
| Label / target | what we predict (default 1 / good 0) |
| Base / bad rate | share of loans that are bad (2.1%) |
| Class imbalance | one class much rarer than the other |
| Leakage | model sees future info it wouldn't have; makes results fake |
| As-of feature | computed using only data up to the loan date |
| Feature | an input number for the model |
| Graph / node / edge | network of members (nodes) joined by guarantees (edges) |
| Union-find | groups everyone connected through guarantees |
| PR-AUC | main score: ranking rare bad loans; baseline = bad rate |
| ROC-AUC | ranking skill; flattering under imbalance |
| Precision | of flagged loans, share truly bad |
| Recall / sensitivity | of bad loans, share caught |
| F1 | balance of precision and recall |
| Threshold / operating point | score cut-off for "bad"; sets recall vs precision |
| Confusion matrix | TN/FP/FN/TP counts |
| FP / FN | false alarm / missed default |
| Cross-validation | rotate train/test and average for an honest estimate |
| Stratified / grouped | keep class ratio / keep a borrower's loans together |
| Out-of-fold (OOF) | scored by a model that didn't train on it |
| Hyperparameter | a setting chosen before training |
| Grid / randomized search | try all combos / sample N combos from ranges |
| XGBoost / gradient boosting | trees built sequentially, each fixing the last's errors |
| MLP / feed-forward NN | layered neural network |
| Overfitting | memorising training quirks; fails on new data |
| Loss | prediction-error number; training lowers it |
| Epoch / boosting round | one training pass / one added tree |
| Learning curve | performance vs amount of training data |
| Calibration / isotonic | make probabilities match real frequencies |
| Monotone constraint | force "more of X moves risk one way" |
| SMOTE | synthetic minority oversampling |
| scale_pos_weight | how much to up-weight the rare class in XGBoost |
| Homophily | risky people cluster together |
| Bootstrap / CI | resample to get a range; crosses zero = not significant |
| Residualisation | test only the part of a feature not explained by others |
| SHAP / permutation importance | per-feature contribution / shuffle-and-measure importance |
| Supervised / unsupervised | with labels / without labels |
| Anomaly / Isolation Forest / LOF | odd point / two outlier detectors |
| Contamination | assumed share of anomalies |
| Louvain / modularity | community-finding / how clean the communities are |
| Contagion / cascade | how a default's exposure spreads through guarantees |
| Survival / Kaplan-Meier | time-to-default modelling / the step survival curve |
| Censoring | a loan that hasn't defaulted yet, handled correctly |
| KMeans / silhouette / PCA | clustering / separation score / compress-for-plotting |
| Winsorise / standardise | clip extremes / rescale to a common scale |

---

## 19. Defense Q&A (whole project — rehearse these)

Answers are kept short on purpose — say them in your own words, then expand if the panel probes.

### Problem & motivation

1. **What problem does GuarantorLens solve, and for whom?** Umwalimu SACCO loan officers approve loans that
   are guaranteed by other members. They lacked a data-driven, explainable way to judge default risk and to
   see the guarantee network. The tool scores risk, explains it, flags network problems, and suggests fixes —
   as **decision support**, not an auto-decision.

2. **Why is the guarantor network the interesting angle?** Because a loan's risk isn't only about the
   borrower — it's also about who backs them, and defaults can spread through shared guarantors. No existing
   tool at the SACCO looked at that structure.

3. **Who are the users and what can each do?** A **loan officer / credit staff** can assess a loan, save an
   application, and **escalate** it to a manager. A **credit manager** reviews escalations and records a
   **recommendation**. Officers cannot approve their own case — enforced in the backend (403), not just the UI.

### Data & label

4. **Describe your dataset.** 11,015 loans, 226 written off (**2.1% bad rate**), 11 branches, disbursed
   2022–2023 (a matured window). Anonymised member IDs, approved for the public repo.

5. **How is the target label defined, and why not the repayment-status column?** Label = 1 if the loan is in
   the Written-Off file. The repayment-status column is the *recovery* state recorded after the decision — a
   written-off-then-repaid loan is still a default, so using status would leak the outcome.

### Validity & leakage

6. **How do you know your results aren't inflated by leakage?** Three defences: features are computed
   **as-of** the loan date (only earlier loans), all **post-decision columns are dropped**, and a
   **date-correlation screen** removes any feature that tracks the disbursement date too closely (it dropped
   `time_since_last_loan`).

7. **Why split by borrower instead of randomly?** A borrower can have several loans; a random split could put
   one of their loans in train and another in test, letting the model recognise them and cheat. Grouping by
   borrower (StratifiedGroupKFold) prevents that and gives honest out-of-fold numbers.

### Metrics

8. **Why not accuracy?** At a 2.1% base rate, "always predict good" is 98% accurate and useless. **PR-AUC**
   (ranking the rare bad loans) is the headline; baseline = the bad rate; we reach ~0.60 (≈29× baseline).

9. **Explain precision vs recall and your choice.** Precision = of flagged loans, how many are truly bad;
   recall = of bad loans, how many we catch. We choose a **recall-first** operating point (catch ~80% of
   defaults) because a missed default costs more than an extra review — which necessarily lowers precision at
   this base rate.

### Modelling choices

10. **Why XGBoost over the other four families?** It won the leaderboard on merit (CV PR-AUC 0.659), handles
    tabular/imbalanced data and missing values, and supports the **monotone constraints** we need. The neural
    net underperforms because we don't have enough data for it.

11. **Why did you train five families and use RandomizedSearch?** To justify the choice with evidence rather
    than assumption, and to tune each fairly. RandomizedSearch samples 15 settings per model from ranges —
    as effective as a full grid but faster.

12. **How did you handle class imbalance?** Compared class weighting vs SMOTE and variants — **class weighting
    won** on PR-AUC. Then tuned `scale_pos_weight`: the textbook `neg/pos` heuristic (~48) optimises balanced
    accuracy, not ranking; PR-AUC actually **peaks at 1** (no reweighting), so we adopt 1 and get recall from
    the threshold. This was the single biggest model gain.

13. **Why calibrate, and how?** Officers read the score as a probability, so we apply **isotonic calibration**
    so a "10%" really defaults ~10% of the time. It fixes the number without changing the ranking.

14. **Why monotone constraints, and what did they cost?** A free tree on few defaults can learn back-to-front
    rules (more savings → more risk). We pin scrutinised features to their common-sense direction. Cost:
    PR-AUC 0.653 → 0.603 and roughly double the false alarms — accepted, because a believable model is one an
    officer will actually trust.

### The network

15. **Does the guarantor network actually improve prediction?** Honest answer: it's **predictive on its own**
    (ROC 0.827) and improves ranking when added (ROC 0.942 → 0.952), but its **incremental PR-AUC lift is
    within noise** (bootstrap +0.012, CI crosses zero) due to **homophily** — risky borrowers cluster with
    risky guarantors. A residualisation robustness check confirms it's genuine, not weak engineering.

16. **Then why keep the network at all?** It powers the flags, the contagion view, the community write-off
    rate, the fix-it advisor, and better ranking (ROC) — real product value beyond the incremental PR.

### Unsupervised & the model portfolio

17. **Why test unsupervised methods, and what did you find?** To check whether we even need labels. Isolation
    Forest and LOF, as **standalone risk models**, score far below the supervised model (PR 0.232 / 0.031 vs
    0.653) — so labels matter, which justifies the supervised path.

18. **Isn't it contradictory that Isolation Forest is both rejected and deployed?** No — two roles. Rejected as
    a *standalone predictor*; kept as a *complementary "unusual application" flag* that never scores risk, just
    marks atypical loans (which default 3.4× more) for a second look.

19. **What are the "additional models" and why include them?** A credit tool needs more than one lens:
    **Louvain communities + contagion** (which guarantee groups are risky, how far a default spreads),
    **Kaplan-Meier survival** (how long loans last — large loans fail soonest), and **KMeans clustering +
    anomaly flag**. Each answers a different question and lives in a different part of the app.

### System, architecture & deployment

20. **Describe the architecture.** React + Vite + TypeScript + Tailwind frontend on Vercel; FastAPI +
    SQLAlchemy backend on Render (PostgreSQL in prod, SQLite locally); the model is a joblib bundle the backend
    loads. Auth is JWT with role-based access.

21. **How does the trained model get from the notebook into the product?** The notebook exports
    `guarantorlens_serving.joblib` (+ an extras bundle and member/loan JSON). The backend loads it and scores
    live requests. **Critical rule:** it's pickled with scikit-learn **1.6.1** to match the backend, or the app
    falls back to rule-based scoring.

22. **How do you make the model trustworthy to a non-technical officer?** Plain-language reasons and an officer
    brief, a clear verdict + one main reason + next step, SHAP driver bars gated to managers, a what-if
    simulator, and a fix-it advisor — the score is always framed as guidance, never an approval.

### Ethics & privacy

23. **What are the privacy/ethics risks and how do you handle them?** The tool exposes other members'
    guarantee/default history, which is sensitive. The **client-facing report is redacted** (no guarantor IDs,
    no third-party default status) so it's safe to circulate; the client email excludes guarantor details; the
    dataset is anonymised. Officer-only screens keep the IDs because the officer needs them to act.

24. **Is the tool making lending decisions? What are its limits?** No — it's decision support; the SACCO
    decides. Limitations to state honestly: a single-SACCO 2022–2023 dataset, only 226 defaults (wide
    confidence intervals), no fairness/branch audit yet, and months-on-book as a proxy for exact write-off
    dates in the survival model. Future work: more data, fairness audit, and monitoring for drift.

### Testing

25. **How did you test it?** 28 automated pytest tests across unit (scoring logic, metamorphic checks like
    "more savings never raises risk"), validation (model beats baseline, bands ordered), integration (real HTTP
    + auth + role gating), functional (the five worked scenarios), and acceptance (officer → escalate → manager
    recommends). Plus the notebook's monotone sweep test cases.

---

## 20. Defense delivery — the panel's advice, applied to your project

### Time budget (what the panel stressed)
- **Problem statement: ≤ 5 minutes.** A few statistics, your objective, existing solutions and their gap — then move on. Do not linger.
- **System architecture: keep it very short** (one slide, 1–2 min). You are an **ML student** — the panel cares about your **data, features, models, and evaluation metrics**, not your infrastructure. Do **not** spend time on FastAPI/React/Render.
- **Spend your time on:** feature engineering, the baselines/models you compared, what you improved, and your metrics.

### The checklist — each mapped to your project
1. **Say where the dataset is from.** Anonymised loan records from **Umwalimu SACCO** — 11 branches, disbursed 2022–2023: **11,015 loans, 226 written off (2.1%)**. `[SOURCE: confirm the exact "how it was provided" sentence with your supervisor before the defense.]`
2. **Compare baselines and other models.** You have a clean ladder: base rate **0.021** → borrower-only **0.641** → network-only **0.237** → borrower+network **0.653**; and five families (Logistic Regression, Decision Tree, Random Forest, XGBoost, feed-forward NN) on the leaderboard. **Show the leaderboard figure.**
3. **How you split the data, and why** (a very likely question). You did **not** use a plain 50/50 or 75/25 *random* split. You used a **borrower-grouped** split: a **~75/25 grouped hold-out** (train 8,265 / test 2,750) plus **5-fold StratifiedGroupKFold** for cross-validation. **Why:** a borrower can have several loans; a random split would put the same borrower in train and test and let the model "recognise" them — that's leakage. Grouping keeps every borrower wholly in one side; stratifying keeps the rare defaults balanced across folds. (So the honest one-liner: "≈75/25, but grouped by borrower and stratified, not random.")
4. **Know everything on every slide.** Don't display a number or term you can't define on the spot. Lean on the glossary (§18) and the metrics tables (§4, Table 3.5).
5. **Explain your feature engineering.** Borrower features + **as-of** behavioural history + the **guarantor-network** features (union-find communities, `community_prior_default_rate`) — all computed as-of the loan date so nothing leaks (§3).
6. **Explain what you improved over the baseline.** Concrete wins: **tuned `scale_pos_weight` (0.594 → 0.653 — your single biggest gain)**, added network features, **isotonic calibration** (honest probabilities), **monotone constraints** (believable behaviour), and a **recall-first threshold**. Lead with the `scale_pos_weight` story.
7. **Demo.** Show one **High-risk** assessment end to end: score + plain reasons + a guarantor-network flag + the "unusual" flag → the fix-it advisor → Monitoring. Keep it tight and rehearsed; don't debug live.
8. **Conclusion + future work.** What you delivered (an explainable, network-aware decision-support tool) + honest limitations (single SACCO, only 226 defaults so wide confidence intervals, no fairness/branch audit, months-on-book as a survival proxy) + next steps (more data, fairness audit, drift monitoring).

### Concepts to be ready to explain (the panel's examples, mapped to you)
- **Class imbalance + SMOTE.** You **tested** SMOTE and three variants, so give the *stronger* answer: defaults are only 2.1%, so you handle imbalance; you **compared SMOTE vs class weighting and class weighting won** on PR-AUC, so you **did not** use SMOTE. Be ready to define SMOTE (invents synthetic minority examples to balance the classes) and why you rejected it (it can distort probabilities; your model is a strong, calibrated tree). Then mention you went further and **tuned the class weight** (`scale_pos_weight`).
- **Give statistics on the problem.** Have 2–3 ready: the **2.1%** write-off rate; risk-based pricing (**14%-rate loans default ~13% vs ~0.5% at 13%**); the riskiest guarantee community **27.5% vs 2.1% (~13×)**.
- **State objectives + existing solutions + the gap.** Objective: an **explainable, network-aware** default-risk decision-support tool for SACCO officers. Existing solutions: score the **borrower alone** and are often **black-box**. Your gap-filler: the **guarantor-network lens** (communities, contagion, flags) **plus explainability** (SHAP + plain reasons).
- **Markov chain / transition matrix — you do NOT use these,** so keep them off your slides. If a panellist asks, know the plain definitions so you're not caught out: a **Markov chain** models moving between *states* where the next state depends only on the current one; a **transition matrix** is *a table showing how something moves from one state to another*. Then clarify what you actually have: the **contagion cascade** is a spread simulation over the guarantee graph (not a Markov chain), and **survival** is Kaplan-Meier / time-to-event (not a transition matrix). Knowing the difference is the point.

**Golden rule the panel repeated:** as an ML student, your marks are in the **evaluation metrics and the modelling decisions**, not the plumbing. Rehearse the metric story until it's second nature.

---

## 21. Model vs rule vs analysis vs data — how every output in the tool is produced

A likely viva question ("is the contagion a model?") and a common confusion. Four different kinds of thing
produce what you see on screen:
- **MODEL** — a trained machine-learning model (learns from data, scores a new input).
- **ANALYSIS** — a statistical method or graph algorithm the backend computes from the data (not trained, not a fixed rule).
- **RULE** — a hard-coded threshold or heuristic.
- **DATA** — a plain read or aggregation of the stored records.

| What you see in the tool | How it is produced | Kind |
|---|---|---|
| Risk score and probability | XGBoost (calibrated, monotone) | **MODEL** |
| "Why this score" drivers / plain reasons | SHAP on the XGBoost model | **MODEL** |
| "Unusual application" flag | Isolation Forest | **MODEL** |
| Fix-it advisor (alternative guarantors) | re-scored by XGBoost | **MODEL** |
| Monitoring "predicted risk" ranking | XGBoost | **MODEL** |
| Borrower segment (report only) | KMeans | **MODEL** |
| "How loans age" survival figures | Kaplan-Meier estimator | **ANALYSIS** |
| Guarantee communities ranking (Insights) | Louvain community detection | **ANALYSIS** |
| Contagion — "if this member fails, what is exposed" | graph-spread simulation over the guarantee network | **ANALYSIS** |
| Guarantor-network flags (guarantor written off; over-committed; band raised to High) | hard-coded thresholds | **RULE** |
| Low / Medium / High bands | 70th / 90th percentile cut-offs on the calibrated score | **RULE** |
| Watchlist "why at risk" reason | strongest-known-signal heuristic | **RULE** |
| Weak-links / single points of failure | guarantee counts + exposure sums (uses model bands for "high-risk") | **DATA + MODEL** |
| Member profile, loans taken, backed-by, guarantees given | read from the data | **DATA** |
| The "Guarantor network" graph (who backs whom) | the raw guarantee links, drawn | **DATA** |
| Community write-off rate on a member | stored lookup | **DATA** |
| Portfolio / by-branch / outcome stats | counts and sums | **DATA** |

**So "is network contagion a model?"** No — it is an **analysis**: the backend walks the guarantee graph to see
how far one default's exposure spreads. The communities are a graph algorithm; survival is a statistical
estimator. Only the risk score, the anomaly flag, and the segments come from **trained models**.

**Two lines for the viva:** "The tool has **three trained models** — XGBoost (risk score), Isolation Forest
(unusual-application flag), KMeans (segments). Everything network (communities, contagion, weak-links) and the
survival figures are **analyses** the backend computes; the flags and bands are **rules** on top of the score;
the rest is just the **data**."

### Where Louvain runs, and what is saved in the joblibs

**Two community computations — don't conflate them:**
- **Louvain** runs in the **notebook** (offline). Each member's `community_id` is exported into `members.json`;
  the backend only *reads* it for the Insights communities ranking. **Louvain is not in the backend.**
- The backend separately computes **union-find connected components** at load — a different, simpler grouping —
  only for the `community_prior_default_rate` *model feature*.

| Community notion | Algorithm | Runs in | Used for |
|---|---|---|---|
| `community_id` (Insights ranking) | **Louvain** | notebook → `members.json` | the communities **view** |
| scoring's community cluster | **union-find** | **backend**, at load | the `community_prior_default_rate` **feature** |

**What the two joblibs actually contain (only trained models + config):**
- `guarantorlens_serving.joblib` → the XGBoost classifier (calibrated + monotone) + `features`, `bands`,
  `medians`, `metrics`, `network_features`.
- `guarantorlens_extra.joblib` → the KMeans segmenter + Isolation Forest anomaly model + their
  imputer / winsor / scaler + cluster descriptions + rates + the anomaly threshold.

**Not in the joblibs:** the member/loan data (separate JSON), the Louvain `community_id` (in `members.json`),
survival + contagion + weak-links (computed live in the backend), and the flag thresholds (hard-coded in
`scoring.py`). Note the Low/Med/High **band cut-offs ARE** saved inside the serving joblib.

**One-liner:** the joblibs hold only the **three trained models** plus their preprocessing and config —
everything else is JSON data, a hard-coded rule, or a live backend calculation.
