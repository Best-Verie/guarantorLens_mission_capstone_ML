# GuarantorLens — Final Report Author's Kit (Chapters 3, 4, 5)

**How to use this file.** This is a *fact and evidence pack*, not the report text. For each section you get: what it must contain, the real numbers/tables/code/figures with their evidence, and bullet points. **Write every paragraph in your own words** (Turnitin checks AI-generated prose). Provided as-is (safe to reuse): tables, code snippets, figures, metrics, tool lists. Written by you: all connecting narrative and discussion.

**Naming:** always "Umwalimu SACCO". **Tense:** past tense.

**Evidence tags:**
- `[fig: models/outputs/NN_*.png]` — a real exported figure (drop it in with the caption given).
- `[code: file]` — a real source file to quote a snippet from.
- `[cite: Author, Year]` — external literature (verify final APA).
- `[SOURCE]` — a Rwanda/institution fact you must source (reuse your proposal's citation).

---

# CHAPTER THREE: SYSTEM ANALYSIS AND DESIGN

## 3.1 Introduction
Purpose bullets (your words): this chapter presents the research design, the dataset, the requirements, and the system/model design.

## 3.2 Research Design (SDLC / methodology)
- **Software process:** iterative, incremental development (frontend, backend, model built and refined in cycles).
- **ML process:** followed a CRISP-DM-style flow — business understanding → data understanding → data preparation → modelling → evaluation → deployment. `[cite: Wirth & Hipp, 2000 (CRISP-DM)]`
- **Why appropriate:** requirements evolved as data understanding grew (e.g., the write-off label and leakage rules were clarified during the project), which suits an iterative process. `[evidence: your own project history]`

### 3.2.1 Dataset and Dataset Description (ML)
- **Source & authorisation:** anonymized member/loan records from Umwalimu SACCO, provided under institutional authorisation. Opaque client IDs; no names or national IDs. `[SOURCE — authorisation]`
- **Sampling:** a **sample of 11 branches** out of ~30 the SACCO operates (not the full network). `[SOURCE — the ~30 figure]` Branches in the sample: Rusizi, Nyagatare, Gasabo, Rubavu, Nyamagabe, Nyanza, Ngororero, Kicukiro, Rutsiro, Nyarugenge, HQ. `[fig: models/outputs/07_data_overview.png]`
- **Files & label construction:** two workbooks — `Normal.xlsx` (label 0) and `Written Off.xlsx` (label 1). Branch is a column; yearly sheets. Where a loan appears in both, **written-off wins** (deduplication). The **label = written-off** (a severe, late-stage outcome); the dataset's *Repayment Status* (Repaid / Active) is the post-write-off recovery state, **not** the label. `[evidence: dataset construction + supervisor clarification]`
- **Terminology (the marker flagged default/NPL/write-off being conflated):** the project keeps three outcomes distinct — **written-off** (the model's label and a member's historical severe loss / "prior write-off"), **NPL / non-performing** (a loan *currently* 90+ days in arrears, monitored separately), and **default risk** (the model's forward-looking prediction of write-off). The interface uses each term only in its own context.
- **Storage:** the anonymized members, loans and guarantees are persisted in **PostgreSQL** (tables `members`, `loans`, `guarantees`), seeded once from the anonymized files and loaded read-only into memory at serve time for the network computations. So the reference data is a real database, not flat files. `[code: app/models.py, app/data_store.py]`
- **Cohort size:** **11,015 loans**, **226 written off (2.1%)** — a highly imbalanced target. Disbursed **2022–2023** (a matured cohort, so loans had time to reach write-off). `[fig: 07_data_overview.png]`
- **Members/network:** 22,485 members in the sample; ~2.5 guarantors per loan. Outcomes: Repaid 10,667 (97%), Written off 226 (2%), In arrears 90+ 122 (1%). `[evidence: Insights page]`
- **Class imbalance:** the 2.1% positive rate makes accuracy misleading; PR-AUC is the headline metric. `[cite: Saito & Rehmsmeier, 2015]` `[fig: 02b_imbalance.png]`

**Feature set (18 features) — Table 3.x:**

| Group | Features |
|---|---|
| Borrower / loan (11) | log_amount, savings, salary, interest_rate, loan_to_savings, loan_to_salary, account_age, b_prior_loans, b_prior_writeoff, b_prior_arrears_rate, b_recent_loans |
| Guarantor network (7) | n_guarantors, g_mean_savings, g_mean_salary, g_prior_default_rate, g_prior_arrears_rate, g_sav_ratio, community_prior_default_rate |

`[code: notebooks/train.ipynb; app/scoring.py _feat_value]`

- **Leakage-safe engineering (important — the marker flagged temporal leakage):** every feature is computed **as of the disbursement date**; post-outcome fields (days in arrears, repayment status, last payment) are blocked; `time_since_last_loan` was **dropped** by the leakage screen. `[fig: 01_leakage_evidence.png]` `[cite: on time-aware validation — verify a temporal-leakage reference]`
- **Community feature:** `community_prior_default_rate` is built with a **union-find** over borrower↔guarantor edges (connected components), taking the as-of default rate of the borrower's cluster. `[code: app/scoring.py _load_communities / _community_rate]`
- **Missing values (guide requires this):** borrower financial fields (savings, salary) are **complete for all 11,015 loans (0% missing)** in this cohort. Missingness sits on the **guarantor side** — guarantors who only ever backed loans have no savings/salary record (**51% of the 22,485 members**), so guarantor-mean features are averaged over the guarantors that have data, and any remaining gaps are **median-imputed** (`SimpleImputer`). Ratio features (`loan_to_savings`, `loan_to_salary`) fall back to the training median when a denominator is missing. `[code: app/scoring.py _feat_value; SimpleImputer in the pipeline]`
- **Data preparation & split / validation design (marker flagged "no validation design"):** rows were de-duplicated to one per loan and features built as-of disbursement. Evaluation used **5-fold borrower-grouped cross-validation** — every loan is scored by a model that never saw that borrower — with metrics reported on the **out-of-fold** predictions across all 11,015 loans (the confusion matrix sums to the full cohort). This prevents any borrower leaking between train and test. `[cite: on grouped CV / time-aware validation — verify a reference]`

## 3.3 Functional and Non-functional Requirements

**Functional requirements (Table 3.x):**

| ID | Requirement |
|---|---|
| FR1 | Score a proposed loan's default risk (0–100 + Low/Medium/High band) |
| FR2 | Explain each score: top drivers (SHAP) + plain-language reasons |
| FR3 | Flag guarantor-network red flags (over-committed backer; defaulter backers) |
| FR4 | What-if simulation (change amount, savings, guarantors) and re-score |
| FR5 | Fix-it advisor: suggest stronger same-branch guarantors with new score |
| FR6 | Member profile, guarantor network graph, and contagion view |
| FR7 | Portfolio insights (overview, weak links, monitoring/early warning) |
| FR8 | Applications workflow: create → escalate → recommend |
| FR9 | Role-based access (loan officer / credit manager / admin) |
| FR10 | Export a printable PDF report of an assessment |
| FR11 | Admin: hot-swap the model and **re-seed the reference dataset** (delete old + re-seed, with a seed timestamp + row counts shown as proof) |

**Non-functional requirements (Table 3.x):**

| ID | Requirement |
|---|---|
| NFR1 | Explainability — no unexplained "black-box" score |
| NFR2 | Calibrated probabilities usable at an operating threshold |
| NFR3 | Leakage-free — features only use information available at decision time |
| NFR4 | Security — JWT auth; anonymized data; no secrets in the repo |
| NFR5 | Availability — transparent rule-based fallback if the model can't load |
| NFR6 | Usability/responsiveness — works on desktop, tablet, mobile |
| NFR7 | Performance — sub-second scoring after cold start |
| NFR8 | Maintainability/deployability — CI-friendly, one-command run |

### 3.3.1 Proposed Model Diagram
`[fig: docs/diagrams/01_ml_pipeline.png]` — the ML pipeline, read **left to right** (landscape, fits a slide): **(1) prepare & merge** the two source files, label, and remove leakage → **(2) feature engineering** (borrower details + guarantor-network signals) → **(3) supervised training & evaluation** of five model families (Logistic Regression, Decision Tree, Random Forest, XGBoost, **Feed-forward NN**) with imbalance handling; an **unsupervised baseline** (IsolationForest / LOF) is run as a **dashed side-branch** that under-performs and loses → **(4) best-model selection** (leaderboard → XGBoost) → **(5) best model + guarantor-network** features → **(6) finalise the model**: monotone constraints → isotonic calibration → Low/Medium/High risk bands (70th/90th score percentiles) → SHAP explainability → export the serving bundle → **(7) deploy & explain**. A parallel **"additional models & analyses"** branch ships alongside the classifier: network communities + contagion, survival, clustering, and the anomaly flag. The unsupervised comparison in step (3) answers "did we need labels?" — the anomaly detectors under-perform (Ch5), which justifies the supervised path; the *same* detector is later reused as the complementary "unusual application" flag (§3.7.4).

### 3.3.2 Why we calibrate and constrain the model (plain terms)
Two finishing steps turn a good ranking model into one a loan officer can trust and defend. Explain both in your own words in the report.

**Calibration - making the score mean what it says.**
- A raw model score is only a ranking signal. A "0.30" says this loan is riskier than a "0.10", but it does not promise that 30 out of 100 such loans really get written off.
- Calibration stretches or squeezes the scores so the number matches reality: of all loans scored around 30%, about 30% actually end up written off.
- Everyday analogy: a weather forecast of "70% chance of rain" is only useful if it really rains on about 70% of such days. Calibration makes the model's percentages that trustworthy.
- Why it matters for GuarantorLens: the officer reads the score as a probability of default, the Low/Medium/High bands sit on top of it, and it has to be comparable to the 2.1% portfolio base rate, so the numbers must be honest. `[method: isotonic calibration via CalibratedClassifierCV]` `[cite: Niculescu-Mizil & Caruana, 2005]`
- Evidence to cite: the reliability curve `[fig: 05_calibration.png]` (predicted vs observed default rate should sit on the diagonal).

**Monotone constraints - keeping the direction sensible.**
- Left free, a model can learn strange wiggles from noise, for example "a bit more savings slightly raises risk" over some range. That is nonsense to a human and destroys trust in the tool.
- A monotone constraint forces a feature to move risk in one agreed direction, always: more savings never raises risk; a bigger loan (relative to savings/salary) never lowers it; a worse guarantor or community repayment record never lowers it.
- Everyday analogy: like wiring a dial so that turning it up can only ever increase the setting, never randomly drop it.
- Why it matters for GuarantorLens: this is decision-support a person must justify to a client, so the explanation can never contradict common sense. We constrain only the six features an officer scrutinises and leave noisy ones free, because over-constraining collapsed accuracy. `[code: MONO set in notebooks/train.ipynb]` `[cite: Chen & Guestrin, 2016]`
- Cost to state honestly: a small accuracy trade (PR-AUC 0.653 -> 0.603, about 1.8x more false alarms at the operating point — 1,005 vs 544, see Table 5.4), accepted in exchange for guaranteed believable behaviour.

**Tie them together (one line):** monotone constraints fix *which way* each feature pushes the score; calibration fixes *how much* the final number actually means. Together they make the model both sensible and honest.

## 3.4 System Architecture
`[fig: docs/diagrams/02_architecture.png]` — a **layered (n-tier)** architecture. The **React web app** (Vite / TypeScript / Tailwind) on **Vercel** talks over **HTTPS with a JWT in the Authorization header** to a **single FastAPI service** (the REST API) on **Render**. That one service is organised in **layers, not separate services**:
- **Routes + JWT auth** (the API layer): the REST routers — `auth`, `assess`, `applications`, `insights`, `admin` — plus the JWT dependency that gates the protected routes (role-based access). Auth is therefore *part of* the REST API, not a separate component: `/login`, `/register`, `/token` are themselves REST endpoints.
- **Service** (the scoring engine): feature build, the calibrated model call, risk bands, flag rules, and SHAP. It loads **two exported model bundles** at startup — `serving.joblib` (the calibrated, monotone XGBoost) and `extra.joblib` (KMeans + Isolation Forest). The models are trained *artifacts* the service loads, so a retrain swaps the file without changing code.
- **ORM** (SQLAlchemy, the data-access layer): the only path to storage.

**All tabular data is in PostgreSQL via the ORM** — users, applications, recommendations, and the reference `members`, `loans`, `guarantees`. The anonymized JSON files are only the DB seed (production runs on PostgreSQL). Key point for the viva: the backend is **one FastAPI application** whose layers (routes + auth, service, ORM) run in the same process; the only resources *outside* the service are the database and the two model files.
- **ERD:** `[fig: docs/diagrams/03_erd.png]` — the **six real DB tables**: User, Application, Recommendation (transactional) and Member, Loan, Guarantee (reference data, now persisted in PostgreSQL). Built from the ORM models, so every field matches the text — fixes the marker's ERD comment.
- **Class diagram:** `[fig: docs/diagrams/04_class_diagram.png]` — the backend services and domain classes (ScoringService, NetworkData, InsightsService, AuthService, plus User/Application/Recommendation) — includes the central domain classes the marker said were missing.

## 3.5 Ethical Considerations
Cover each of these — the guide requires them and the **proposal lost marks here (0.65/1)**. Write in your own words; the points are real.
- **Privacy & pseudonymisation:** the dataset uses **opaque client IDs only** — no names or national IDs. Use "pseudonymised/anonymised" accurately (identifiers were replaced, not merely hidden). Data used under **institutional authorisation**. `[SOURCE — authorisation]`
- **Sensitive financial data:** savings, salary, and default history are sensitive; they are stored anonymised, with no secrets in the repository and access behind JWT authentication. `[code: app/security.py]`
- **Bias, fairness & guilt-by-association (the marker flagged this specifically):** network features can penalise a borrower for *who backs them*. This is disclosed as a risk; the model reports **association, not causation**; and a **branch-parity / fairness audit** is recommended before any operational use.
- **Model misuse:** the tool must not be used to auto-decline or auto-approve loans, or to profile members.
- **Human-in-the-loop:** the loan officer proposes and the credit manager decides — the model never makes the lending decision. `[code: role gating in app/applications.py]`
- **No live lending decisions:** the prototype is decision-support for research; it did **not** make real lending decisions during the project.
- **Retention, access & breach:** state how long the data is kept, who may access it, and the breach procedure (aligned with the SACCO's policy). `[SOURCE — SACCO policy]`
- **Ethical approval / authorisation:** state the authorisation under which Umwalimu SACCO shared the data. `[SOURCE]`

## 3.6 Development Tools (Table 3.x)

| Layer | Tools |
|---|---|
| Modelling | Python, pandas, scikit-learn 1.6.1, XGBoost 3.2.0, imbalanced-learn 0.14.2, SHAP, NetworkX (guarantee graph), NumPy, Matplotlib; Google Colab |
| Backend | FastAPI, Uvicorn/Gunicorn, SQLAlchemy, PyJWT, joblib |
| Frontend | React 19, Vite, TypeScript, Tailwind CSS |
| Database | PostgreSQL (prod), SQLite (local) |
| Hosting/CI | Render (API), Vercel (frontend), GitHub |
| Testing | pytest, httpx |

---

## 3.7 Additional models and analyses (beyond the classifier)

The default-risk classifier (section 3.3) is the main **model**. A credit problem needs more than one lens, so GuarantorLens adds four more techniques. Be precise about what they are: **two are trained models** (like the classifier - they learn from data and score a new input) and **two are analyses** (they summarise the data or its network structure; they do not predict a new loan). Reproduced in `notebooks/pipeline.ipynb`; figures and tables under `models/guarantorlens_new/` (`10_*`, `11_*`, `12_*`). All are **leakage-safe** (as-of / application-time features only).

| Technique | Type | Question it answers | Method (library) | Where it appears in the tool |
|---|---|---|---|---|
| Guarantee network | analysis | Which guarantee groups are risky, and how far does a default spread? | Louvain communities + contagion cascade (networkx) | Insights (communities, weak-links) + member page (contagion, graph) |
| Survival | analysis | How long does a loan last before write-off? | Kaplan-Meier estimator (numpy/pandas) | Monitoring (plain-language summary) |
| Segmentation | unsupervised **model** | What borrower types exist? | KMeans, k = 4 (scikit-learn) | Notebook / report only |
| Anomaly | unsupervised **model** | How unusual is this application? | Isolation Forest (scikit-learn) | Assess result ("Unusual application") |

**Are they all "models", and what "metrics" does each have?** Only the supervised classifier is *tuned against a performance metric* and carries *accuracy-style* numbers, because only it is checked against known outcomes. The two unsupervised models still **train** and have **hyperparameters**, but they are **configured** (not tuned against labels) and judged by a *quality* score or *external validation*. The two analyses report *descriptive* statistics, not performance.

**Table 3.5 — What each technique is, and the metrics it has.**

| Technique | Trains? | Uses the label? | Tuned vs a metric? | Metrics / key findings |
|---|---|---|---|---|
| **XGBoost** (the main model, for reference) | yes | yes | yes (PR-AUC via CV) | PR-AUC 0.603, ROC 0.932, precision/recall/F1, confusion matrix, calibration |
| **KMeans** (segmentation) | yes | no | no - `k = 4` set for interpretability | silhouette **0.33**; 4 segments, write-off **1.8-2.8%** (Table 3.7) |
| **Isolation Forest** (anomaly) | yes | no | no - contamination = 2.1% (the bad rate) | validation: flagged 10% default **6.9% vs 2.1% (3.4x)**; as a standalone predictor 0.889 / 0.232 |
| **Kaplan-Meier** (survival) | computed over data | - | - | **~98% still performing at 24 months**; large loans 97.1% (fail soonest) |
| **Louvain + contagion** (network) | graph algorithm | - | - | **1,366 communities, modularity 0.99**; riskiest **27.5% (~13x)**; cascade **217->377**; 173 loans / RWF 364M exposed |

**Technologies used.**
- Serving bundle `guarantorlens_serving.joblib`: **XGBoost** + scikit-learn **isotonic calibration**.
- Extra bundle `guarantorlens_extra.joblib`: scikit-learn **KMeans** + **Isolation Forest** with their imputer / winsor / scaler.
- Computed live in the backend (no bundle): **networkx** Louvain communities + a contagion traversal; a hand-written **Kaplan-Meier** estimator (numpy / pandas).

**Key things to know.**
- Each answers a *different* question; none replaces the risk score.
- The two trained extras are **unsupervised** - they never see the default label, so they cannot be "tuned for accuracy"; we can only validate them afterward because we happen to have outcomes.
- Segmentation is **context, not risk** (segments barely differ in risk) - notebook / report only.
- The anomaly flag is a **prompt, not a probability**; survival and the network views are **descriptive analyses**, not per-loan predictions.

### 3.7.1 Guarantee-network model (communities and contagion)
- **What it is:** the guarantees form a graph (each loan links a borrower to its guarantors). Community detection groups tightly connected members; contagion follows how one member's failure exposes others.
- **Why we added it:** the tool is network-aware, so we model the network itself, not just borrower features. It answers "which groups are risky" and "how far can one default spread".
- **Settings and why:** Louvain community detection (`networkx.community.louvain_communities`, seed 42 for reproducibility) because it needs no preset number of groups and maximises **modularity** (0 to 1, how cleanly the network splits). Contagion is a simple cascade: seed with members already written off, mark a member "exposed" if a guarantor is compromised, repeat for a few rounds.
- **What it adds / where:** powers the Contagion, Network and Weak-links views; the "Community write-off rate" stat on the member page is this model's output.
- **Metrics (this data):** 22,485 members (nodes), 25,624 guarantee links (edges); **1,366 communities, modularity 0.99**; the riskiest community has a **27.5% write-off rate vs the 2.1% portfolio (about 13x)**. One-hop exposure: **173 loans (~RWF 364 million)** are guaranteed by a written-off member; a default cascades from **217 to 377 members over four rounds**. `[fig: 10_community_default.png, 10_contagion.png, 10_network_sample.png; table: 10_communities.csv]`
- **Honest note:** the very high modularity means the network is many small tight clusters, not one giant graph, which is realistic for a SACCO.

### 3.7.2 Survival analysis (time to default)
- **What it is:** instead of "will it default", it asks "how long does a loan last before write-off". The Kaplan-Meier curve shows the share still performing as months pass.
- **Why we added it:** timing matters for monitoring and provisioning; it is a standard, distinct credit-risk technique.
- **Settings and why:** Kaplan-Meier estimator (implemented directly, no extra library). Duration = months on book from disbursement to a fixed observation date (2024-12-31); event = written off; repaid or still-active loans are **censored**. Split by loan-size tier (tertiles) to compare groups. `[cite: Kaplan & Meier, 1958]`
- **Data caveat:** the dataset records the disbursement date and the final outcome, not the exact write-off date, so months-on-book is a standard proxy for the event time.
- **What it adds / where:** the Monitoring page shows a **plain-language summary** of the finding (about 98% of loans still healthy at 2 years; large loans slip soonest). The full Kaplan-Meier curve is kept in the notebook/report, not the UI (the step-curve is too technical for an officer).
- **Metrics:** about **99.97% of loans still performing at 12 months, 97.97% at 24 months**; **large loans fail soonest (97.13% at 24mo)** vs medium (98.50%) and small (98.27%). `[fig: 11_survival_km.png; table: 11_survival_table.csv]`

### 3.7.3 Borrower segmentation (clustering)
- **What it is:** groups borrowers into a few "types" from loan and profile features, without using the default label (unsupervised).
- **Why we added it:** a portfolio view of borrower "types" and an extra unsupervised technique.
- **Settings and why:** KMeans, **k = 4** (simple, interpretable segments). Features: loan amount, savings, salary, loan-to-savings, number of guarantors. Features are **median-imputed**, **winsorised to the 1st-99th percentile** (so one extreme value cannot own a cluster), then **standardised** (the features are on very different scales); seed 42. Quality: **silhouette 0.33** (moderate separation).
- **What it adds / where:** kept in the **notebook and report only** — a valid technique, but the segments barely differ in risk for a single loan, so it is **not shown in the UI** (see §3.7.5).
- **Metrics:** four segments with write-off rates **1.8% to 2.8%**; they differ mainly by loan size and wealth and only mildly by risk (honest: default is rare and driven by other factors). `[fig: 12_cluster_scatter.png; table: 12_clusters.csv]`

**Table 3.7 — Borrower segments (KMeans, k = 4).** `[table: 12_clusters.csv]`

| Segment | Loans | Write-off rate | Avg loan | Median savings | Avg guarantors | Profile |
|---|---:|---:|---:|---:|---:|---|
| 0 | 3,534 | 2.0% | RWF 1.64M | RWF 180k | 2.0 | smaller loans, lower savings, few guarantors |
| 1 | 1,345 | 2.8% | RWF 3.02M | RWF 1.08M | 2.4 | larger loans, higher savings, few guarantors |
| 2 | 3,631 | 1.8% | RWF 1.59M | RWF 150k | 3.0 | smaller loans, lower savings, more guarantors |
| 3 | 2,505 | 2.2% | RWF 3.13M | RWF 258k | 2.6 | larger loans, higher savings, few guarantors |

The write-off rates sit in a narrow **1.8-2.8%** band — the segments separate borrowers by size and wealth far more than by risk, which is why segmentation is context, not a risk score.

### 3.7.4 Anomaly detection (unusual-application flag)
- **What it is:** scores how unusual an application looks compared with the rest of the book.
- **Why we added it:** a cheap early screen that flags atypical cases for a closer look.
- **Settings and why:** Isolation Forest, **200 trees**, **contamination set to the portfolio bad rate (~2.1%)** so the expected share of anomalies matches the real write-off rate; we flag the **most unusual 10%** (a practical review budget); seed 42.
- **What it adds / where:** an "Unusual application" badge on the assessment result and saved application.
- **Metrics:** the flagged 10% default **6.9% of the time vs 2.1% overall (about 3.4x)**. `[fig: 08_unsupervised_vs_supervised.png]`
- **Honest note:** unusual means atypical, not automatically risky; it is a prompt to look closer, not a decision.
- **Relation to Chapter 5:** this is the *same* Isolation Forest that Ch5 (§5.1) reports as under-performing the classifier. There is no contradiction — Ch5 rejects it as a **standalone risk model** (it cannot rank defaults as well as the supervised model), while here it is kept in a **different role**: a complementary flag that never scores risk, only marks atypical applications for a second look. Both statements are true of the same tool used two different ways.

### 3.7.5 How the extra models are served, and where they appear in the tool
- The two **per-application** models (segment, anomaly) are exported to `guarantorlens_extra.joblib` and loaded by the backend (`app/extra.py`); they are added to every `/assess-risk` response and stored on the saved application. Pickled with **scikit-learn 1.6.1** to match the serving environment (same version rule as the main model). If the bundle is missing, the fields are simply omitted and nothing breaks.
- The **network** and **survival** models are computed in the backend from the loan and member tables (no extra model file) and served via endpoints (`/insights/survival`, and the community rate on the member detail).

| Model | Where it appears in the tool |
|---|---|
| **Anomaly (Isolation Forest)** | an **"Unusual application"** badge on the assessment result and the saved application, shown only when the profile is atypical |
| **Survival (Kaplan-Meier)** | a **plain-language summary** on the **Monitoring** page (the full step-curve stays in the report) |
| **Network (Louvain + contagion)** | the **Insights** page (communities ranked by write-off rate + weak-links) and the **member page** (contagion cascade + guarantee graph) |
| **Segmentation (KMeans)** | kept in the **notebook/report only** — it is a valid unsupervised technique but adds little for a single loan (the segments barely differ in risk and the label can look odd on one application), so it is **not shown in the UI** |

*Design note (worth writing up):* the **anomaly** flag is a book-wide validation, not a per-loan probability - the most-unusual 10% of applications were historically written off ~3x more often (6.9% vs 2.1%), so the badge is a "look closer" prompt, not a second score.

## 3.8 How each output is produced (model vs rule vs analysis vs data)

For transparency and for the viva, every user-facing output falls into one of four kinds:
- **MODEL** - a trained machine-learning model (learns from data, scores a new input).
- **ANALYSIS** - a statistical method or graph algorithm the backend computes from the data (not trained, not a fixed rule).
- **RULE** - a hard-coded threshold or heuristic.
- **DATA** - a plain read or aggregation of the stored records.

| What the user sees | How it is produced | Kind |
|---|---|---|
| Risk score and probability | XGBoost (calibrated, monotone) | **MODEL** |
| "Why this score" drivers and plain reasons | SHAP on the XGBoost model | **MODEL** |
| "Unusual application" flag | Isolation Forest | **MODEL** |
| Fix-it advisor (alternative guarantors) | re-scored by XGBoost | **MODEL** |
| Monitoring "predicted risk" ranking | XGBoost | **MODEL** |
| Borrower segment (report only) | KMeans | **MODEL** |
| "How loans age" survival figures | Kaplan-Meier estimator | **ANALYSIS** |
| Guarantee communities ranking (Insights) | Louvain community detection | **ANALYSIS** |
| Contagion - "if this member fails, what is exposed" | graph-spread simulation over the guarantee network | **ANALYSIS** |
| Guarantor-network flags (guarantor written off before; over-committed guarantor; band raised to High) | hard-coded thresholds | **RULE** |
| Low / Medium / High bands | 70th / 90th percentile cut-offs on the calibrated score | **RULE** |
| Watchlist "why at risk" reason | strongest-known-signal heuristic | **RULE** |
| Weak-links / single points of failure | guarantee counts + exposure sums (uses the model's bands for the "high-risk" count) | **DATA + MODEL** |
| Member profile, loans taken, backed-by, guarantees given | read from the data | **DATA** |
| The "Guarantor network" graph (who backs whom) | the raw guarantee links, drawn | **DATA** |
| Community write-off rate shown on a member | stored lookup | **DATA** |
| Portfolio / by-branch / outcome statistics | counts and sums | **DATA** |

**So "is network contagion a model?"** No - it is an **analysis**: the backend walks the guarantee graph to see how far one default's exposure spreads. The communities are a graph algorithm and survival is a statistical estimator. Only the risk score, the anomaly flag, and the segments come from **trained models**; the flags and bands are **rules** on top of the score; the rest is the **data**.

**Where Louvain runs, and what is saved in the model bundles.** Two community computations must not be conflated: **Louvain** runs in the **notebook** (offline) and each member's `community_id` is exported into `members.json` - the backend only reads it for the Insights communities ranking (Louvain is **not** in the backend); separately the backend computes **union-find connected components** at load, only for the `community_prior_default_rate` model feature.

| Community notion | Algorithm | Runs in | Used for |
|---|---|---|---|
| `community_id` (Insights ranking) | Louvain | notebook -> `members.json` | the communities view |
| scoring's community cluster | union-find | backend, at load | the `community_prior_default_rate` feature |

The two saved bundles (joblibs) hold **only the trained models plus their config**: `guarantorlens_serving.joblib` = the calibrated, monotone XGBoost + `features`, `bands`, `medians`, `metrics`; `guarantorlens_extra.joblib` = KMeans + Isolation Forest with their imputer / winsor / scaler + cluster descriptions/rates + the anomaly threshold. **Not** in the joblibs: the member/loan data (separate JSON), the Louvain `community_id` (in `members.json`), survival + contagion + weak-links (computed live in the backend), and the flag thresholds (hard-coded in `scoring.py`) - though the band cut-offs *are* inside the serving bundle.

---

# CHAPTER FOUR: SYSTEM IMPLEMENTATION AND TESTING

## 4.1 Implementation and coding
### 4.1.1 Introduction
Bullets (your words): the system has three parts — the trained model (Colab notebook), the serving/API backend (FastAPI), and the officer-facing frontend (React). The model is exported as a bundle and loaded by the API.

### 4.1.2 Implementation tools and technology (how each was used)
- **XGBoost + CalibratedClassifierCV (isotonic) + monotone constraints** — the deployed estimator. `[cite: Chen & Guestrin, 2016; Niculescu-Mizil & Caruana, 2005]` `[code: notebooks/train.ipynb]`
- **imbalanced-learn** — required to unpickle the pipeline; class weighting handled imbalance (beat SMOTE, see Ch5). `[cite: Chawla et al., 2002; Elor & Averbuch-Elor, 2022]`
- **SHAP** — per-feature contributions for explanations. `[cite: Lundberg & Lee, 2017]` `[code: app/scoring.py _shap]`
- **FastAPI** — REST API, auto Swagger docs; JWT auth. `[code: app/main.py, app/auth.py]`
- **React + TypeScript + Tailwind** — officer UI. `[code: src/pages/*]`
- **Version pin note (worth a sentence):** the serving bundle is pickled with scikit-learn 1.6.1; a mismatched version drops the API to the rule-based fallback — so the version is pinned. `[code: requirements.txt]`
- **Data storage & seeding (good defense point):** members/loans/guarantees are stored in **PostgreSQL** and loaded into memory at startup; the anonymized files are only the seed. The network computations (communities, contagion, weak-links) traverse the whole graph, so keeping it in memory is far faster than per-row SQL — the DB is the source of truth, the in-memory graph is the read path. Admin uploads **delete and re-seed** the tables (so uploaded data survives redeploys) and record a timestamped seed row as proof. `[code: app/data_store.py, app/models.py, app/admin.py]`

**Key code snippets to include (quote these):**
- Leakage-safe as-of feature building — `[code: app/scoring.py _feat_value / _build_features]`
- Union-find community feature — `[code: app/scoring.py _load_communities]`
- Guarantor-network flag overlay (leak-free band escalation) — `[code: app/scoring.py adjust_band]`
- Band-consistent display score — `[code: app/scoring.py _display_for_band]`

## 4.2 Graphical view of the project
### 4.2.1 Screenshots with description
Use your captioned screenshot deck (`GuarantorLens screenshots captioned.docx`). Suggested figures: empty assess form; Low result; High result with flags; plain-language drivers; fix-it advisor; insights; monitoring; member profile + contagion + network graph; mobile/tablet views. Each already has a caption.

## 4.3 Testing
### 4.3.1 Introduction (your words): a mix of automated and manual strategies was used.
### 4.3.2 Objective of testing
- Confirm correct functionality, sane model behaviour, access control, and cross-environment usability.

### 4.3.3 Unit testing outputs `[code: tests/test_scoring_unit.py]`
Automated unit tests of the scoring logic:
- band boundaries; display-score aligns with band and is monotonic in probability;
- two defaulter guarantors escalate the band to High;
- **metamorphic tests**: more savings never raises risk; a bigger loan never lowers it;
- `assess()` returns the full expected shape.

### 4.3.4 Validation testing outputs `[code: tests/test_validation.py]` (5 tests)
- The **deployed model beats the baseline and meets spec** (PR-AUC far above base rate; ROC ≥ 0.85).
- **Bands are ordered** (Low < Medium < High thresholds).
- The **displayed score never contradicts the band** (swept over 0–1).
- The API **rejects a zero amount (400)** and **missing amount (422)**.
- Model-level validation evidence: leakage screen `[fig: 01_leakage_evidence.png]`, calibration curve `[fig: 05_calibration.png]`, borrower-grouped CV, bootstrap CIs on the network lift (honest: incremental PR lift CI crosses zero).

### 4.3.5 Integration testing outputs `[code: tests/test_api_integration.py]` (6 tests)
Real HTTP with auth: health check; auth required; assess-risk returns a valid band/score; a well-covered loan scores no higher than a thin one; **an officer is blocked (403) from recording a recommendation** while a manager succeeds.

### 4.3.6 Functional and system testing results `[code: tests/test_functional_cases.py]` (6 tests)
Each of the five verified scenarios is an **automated** functional test asserting the band (and flag). Plus a rate-lever sanity test (higher rate never lowers the band).

| # | Inputs (amount/savings/salary/rate/guarantors) | Asserted result |
|---|---|---|
| 1 | 300k / 2.5M / 400k / 13 / 2 clean | Low |
| 2 | 5M / 90k / 220k / 14 / 2 clean | Medium |
| 3 | 12M / 30k / 150k / 14 / 2 clean | High |
| 4 | 2M / 300k / 300k / 14 / over-committed + clean | Medium + over-committed flag |
| 5 | 1.5M / 200k / 250k / 14 / 2 defaulters | High + defaulter flag |

Cross-environment: verified on desktop, tablet, and mobile widths, in more than one browser (screenshots in `screenshots/`).

### 4.3.7 Acceptance testing report `[code: tests/test_acceptance.py]` (3 tests)
End-to-end user scenarios through the API:
- **Officer proposes → escalates → manager recommends** (statuses transition assessed → escalated → recommendation recorded).
- **Officer cannot approve their own case** (403).
- **Manager sees the escalation queue.**
Also demonstrated live with the supervisor.

### 4.3.8 Test summary (real results — `venv/bin/python -m pytest`)
| Testing category | File | Tests | Result |
|---|---|---:|---|
| Unit | test_scoring_unit.py | 8 | passed |
| Validation | test_validation.py | 5 | passed |
| Integration | test_api_integration.py | 6 | passed |
| Functional / system | test_functional_cases.py | 6 | passed |
| Acceptance | test_acceptance.py | 3 | passed |
| **Total** | | **28** | **28 passed** |

Run with:
```
cd guarantorLens_mission_capstone_BE
venv/bin/pip install -r requirements-dev.txt
venv/bin/python -m pytest        # 28 passed
```
Screenshot the `28 passed` line as testing evidence.

---

# CHAPTER FIVE: DESCRIPTION OF THE RESULTS

## 5.1 Model results

**Table 5.1 — Feature-set comparison (held-out):** `[fig: models/outputs/00_final_metrics.csv / 05_roc_pr_curves.png]`

| Model set | PR-AUC | ROC-AUC |
|---|---:|---:|
| Baseline (bad rate) | 0.021 | 0.500 |
| Borrower-only | 0.641 | 0.942 |
| Network-only | 0.237 | 0.827 |
| Borrower + network (unconstrained) | 0.653 | 0.952 |
| Borrower + network (monotone → **deployed**) | **0.603** | **0.932** |

**Table 5.2 — Best per model family (CV PR-AUC):** `[fig: 02_leaderboard.png; table: 05b_xgb_featureset_metrics.csv]`

| Model | Feature set | CV PR-AUC |
|---|---|---:|
| XGBoost | borrower+network | 0.659 |
| Random Forest | borrower-only | 0.572 |
| Decision Tree | borrower-only | 0.374 |
| Neural network (feed-forward, MLP) | borrower+network | 0.348 |
| Logistic Regression | borrower+network | 0.346 |

**Table 5.3 — Imbalance strategy (PR-AUC):** class weighting matched/beat every SMOTE variant, so it was chosen. `[fig: 02b_imbalance.png]` `[cite: Elor & Averbuch-Elor, 2022]`

| Strategy | PR-AUC |
|---|---:|
| Class weighting | 0.593 |
| BorderlineSMOTE | 0.543 |
| SMOTETomek | 0.519 |
| SMOTE | 0.516 |
| ADASYN | 0.512 |
| Undersample | 0.434 |

**Table 5.3b — Tuning the class-weighting hyperparameter (`scale_pos_weight`).** The usual rule is `neg/pos` (~48), but that targets balanced accuracy at a 0.5 threshold, not ranking. For our headline metric (PR-AUC) heavy up-weighting distorts the probabilities and *lowers* PR-AUC; the recall we want comes from the **operating threshold**, not the weight. The notebook now **sweeps and tunes it**, and the deployed model adopts the best value (=1). `[fig: 02c_scale_pos_weight.png; table: 02c_scale_pos_weight.csv]` `[cite: Brownlee, Weighted XGBoost for Class Imbalance; Elor & Averbuch-Elor, 2022]`

| scale_pos_weight | PR-AUC (borrower+network, OOF) |
|---|---:|
| **1 (best, adopted)** | **0.653** |
| 2 | 0.650 |
| 5 | 0.634 |
| 10 | 0.623 |
| 25 | 0.606 |
| 48 = neg/pos (old default) | 0.594 |
| 100 | 0.573 |

**Effect: PR-AUC on the borrower+network model rises 0.594 → 0.653 (+0.06) as the weight drops from the neg/pos heuristic (48) to 1; best-F1 0.586 → 0.66. The deployed monotone model, at the adopted weight, is PR-AUC 0.603 / ROC 0.932** — the single biggest model improvement, with the monotone constraints and calibration unchanged.

**Table 5.4 — Confusion matrix at the recall-first operating point (≥80% recall), at three stages.** Model chosen at selection (unconstrained) → after isotonic calibration → the **deployed** monotone-constrained model, so the cost of the realism constraint is visible. `[fig: 03a_confusion_unconstrained.png (selection) / 03b_confusion_calibrated.png (calibrated) / 03_confusion.png (deployed) / 06_confusion_grid.png]`

| Metric | Unconstrained (selection) | After calibration | **Deployed (monotone)** |
|---|---|---|---|
| Threshold (≥80% recall) | 0.032 | 0.045 | 0.017 |
| True negatives / False positives | 10,245 / 544 | 10,099 / 690 | 9,784 / 1,005 |
| False negatives / True positives | 45 / 181 | 45 / 181 | 45 / 181 |
| Recall | 0.801 (181 of 226 caught) | 0.801 | 0.801 |
| Precision | 0.250 | 0.208 | 0.153 |
| F1 | 0.381 | 0.330 | 0.256 |
| Accuracy | 0.947 | 0.933 | 0.905 |

All three catch the **same 181 of 226** write-offs at this point. Calibration moves the operating point modestly (544 → 690 false alarms); the deployed monotone model gets there with **markedly more false alarms** (1,005 vs 544) — the honest price of forcing every score to move in a believable direction. `[note: canonical Colab output "guarantorlens_new (9)", scikit-learn 1.6.1, matching the shipped bundle. Calibrated held-out PR-AUC 0.746; deployed bands medium 0.008 / high 0.028.]`

- **SHAP importance** — top drivers: interest_rate, salary, loan_to_salary, account_age, g_sav_ratio, loan_to_savings, savings, community_prior_default_rate. `[fig: 04_shap_summary.png]` `[cite: Lundberg & Lee, 2017]`
- **Permutation importance** (model-agnostic cross-check) — shuffling each feature and measuring the drop in PR-AUC gives the **same top-of-list order** as SHAP (interest_rate, then guarantor/community history), so the explanation does not depend on the model's internals. `[fig: 04b_permutation_importance.png; table: 04b_permutation_importance.csv]`
- **Best model across feature sets** `[fig: 05b_xgb_confusion.png; table: 05b_xgb_featureset_metrics.csv]` — XGBoost on borrower-only vs network-only vs borrower+network, with PR-AUC/ROC-AUC and precision/recall/F1 at the ≥80%-recall point, plus the confusion matrix of the deployed borrower+network model (caught 181/226 with 539 false alarms at that operating point).
- **Per-model diagnostics** `[fig: 02d_diag_<model>.png]` — each family's confusion matrix and learning curve.
- **Training and loss curves** `[fig: 09b_learning_curves.png]` — three convergence checks: (1) XGBoost log-loss per boosting round, train vs validation, with a **small gap = the model is not over-fitting**; (2) the feed-forward network's training loss per epoch; (3) cross-validated PR-AUC as the training set grows (**still rising at full data = more data would help**).
- **Calibration** — probabilities track observed default rates. `[fig: 05_calibration.png]`
- **Unsupervised baselines** — tested as a *standalone risk model* (a replacement for the classifier), the best anomaly detector ranks defaults far below the supervised model: IsolationForest ROC 0.889 / PR 0.232 and LOF 0.598 / 0.031, versus 0.952 / 0.653. So anomaly detection is **not** used to score risk — this is what justifies the supervised path. The **same** IsolationForest is, however, kept in a *different role* — a complementary **"unusual application" flag** (§3.7.4): not a risk score, but an atypicality prompt, and in that role it earns its place (the flagged 10% default 3.4× more). Rejected as a predictor, kept as a complement. `[fig: 08_unsupervised_vs_supervised.png]`

## 5.2 System results
- A working, deployed decision-support tool (live URLs in the README) that scores, explains, flags, simulates, and advises.
- The five functional test cases above produced the expected bands and flags — the tool behaves sensibly across the risk spectrum.

## 5.3 Discussion (your analysis — bullets to expand)
- **RQ1 (prediction):** written-off loans are predicted far above the 2.1% base rate (ROC 0.92; ~80% of defaults caught at the operating threshold).
- **RQ2 (does the network help?):** honest finding — the network is **predictive on its own** (PR 0.237, ROC 0.827, far above baseline) and lifts ranking slightly when added (ROC 0.942→0.952), but its **incremental PR lift over borrower-only is within noise** (bootstrap lift +0.012, 95% CI [-0.019, +0.044]) because risky borrowers cluster with risky guarantors (**homophily**). `[cite: McPherson et al., 2001]`
- **RQ3 (calibration):** probabilities are calibrated and usable at a recall-first threshold.
- **RQ4 (branches/fairness):** not yet audited — flagged as a limitation.
- **Design trade-off** (the "why" is in §3.3.2): monotone constraints traded held-out PR (0.653 → 0.603) and roughly 1.8× more false alarms at the operating point (1,005 vs 544, see Table 5.4) for guaranteed sane behaviour in production (more savings never raises risk; bigger loan never lowers it). For a decision-support tool a loan officer must trust, believable behaviour is worth the extra review load.

## 5.4 Limitations
- 11-branch **sample** (of ~30) → generalisability to the full network is untested.
- **Write-off label delay**: mitigated by the matured 2022–2023 cohort, but some very recent loans may not yet have reached write-off.
- **Binary label** hides intermediate states (restructuring, partial recovery).
- **No fairness/branch-parity audit** yet.
- The model reports **association, not causation**.

## 5.5 Link to objectives
- Objective 1 (leakage-safe data) — achieved (`01_leakage_evidence.png`).
- Objective 2 (network features) — achieved (7 network features).
- Objective 3 (tuned, calibrated model) — achieved (PR 0.52 / ROC 0.92, calibrated).
- Objective 4 (explainable interface) — achieved (deployed tool).

---

# CHAPTER SIX: CONCLUSIONS AND RECOMMENDATIONS
*(ALU template Ch6 = the defense guide's "Chapter 5: Conclusion & Recommendations" content.)*

## 6.1 Introduction (your words)
One or two sentences: this chapter restates what was achieved, answers the research questions, and gives recommendations and future work.

## 6.2 Summary of objectives and how they were met
| Objective | Met? | Evidence |
|---|---|---|
| Leakage-safe dataset + data-quality validation | Yes | leakage screen; `figures/01_leakage_evidence.png` |
| Borrower + guarantor-network features | Yes | 18 features (11 + 7 network) |
| Trained, tuned, calibrated model | Yes | PR-AUC 0.52 / ROC 0.92; `figures/02_leaderboard.png`, `05_calibration.png` |
| Explainable decision-support interface | Yes | deployed tool; SHAP + plain-language; `screenshots/` |

## 6.3 Answers to the research questions (bullets to expand)
- **RQ1 (prediction):** achieved — ROC-AUC 0.92, ~80% of defaults caught at the operating threshold, far above the 2.1% base rate.
- **RQ2 (network value):** the guarantor network is **predictive on its own** but its **incremental** PR-AUC lift over borrower-only is within noise (homophily). Honest, and a genuine finding.
- **RQ3 (calibration):** the probabilities are calibrated and usable at a recall-first threshold.
- **RQ4 (branches/fairness):** **not** established — a limitation and future work.

## 6.4 Main findings and contribution
- A **leakage-safe, calibrated, explainable, network-aware** loan-default risk model for a SACCO, delivered as a working **decision-support** prototype (not automated lending).
- An honest empirical result on **guarantor-network features and homophily** in cooperative lending.

## 6.5 Limitations (recap)
11-branch **sample** of ~30 branches; write-off **label delay**; **binary** label hides intermediate states; **no fairness/branch-parity audit**; the model reports **association, not causation**.

## 6.6 Recommendations (to Umwalimu SACCO / the community)
- Use the tool as **decision support** — officer proposes, manager decides — never automatic approval.
- **Collect more matured defaults** and extend beyond the 11-branch sample to strengthen the network signal.
- **Monitor calibration and fairness** over time before any operational reliance.

## 6.7 Future work
- **Prospective evaluation**: deploy and measure whether write-offs actually fall over time.
- **Temporal / dynamic network features**; a **fairness / branch-parity audit**; server-side reporting and client-safe PDFs.

## 6.8 Conclusion (strong, specific — not "it worked")
State the concrete outcome, e.g. XGBoost with guarantor-network features, calibrated and monotone-constrained, ranked default risk at ROC-AUC 0.92 in a working, explainable prototype; the network adds real signal but its incremental gain is limited by homophily, and the tool validates the framework rather than proving real-world NPL reduction.

---

## Figures index (drop-in, all real, in `models/outputs/`)
| Figure | File | Suggested caption |
|---|---|---|
| Data overview | 07_data_overview.png | Cohort size, class balance, distributions |
| Feature correlation | 07_feature_correlation.png | Correlation heatmap of features |
| Leakage evidence | 01_leakage_evidence.png | Blocked post-outcome features |
| Merged dataset | 00_merged_dataset.csv | the two files joined into one labeled table |
| Model leaderboard | 02_leaderboard.png | PR-AUC by model family and feature set |
| Class-weight tuning | 02c_scale_pos_weight.png | PR-AUC vs scale_pos_weight (tuning beats the neg/pos heuristic) |
| Per-model diagnostics | 02d_diag_<model>.png | each family's confusion matrix + learning curve |
| Best model by feature set | 05b_xgb_confusion.png / 05b_xgb_featureset_metrics.csv | XGBoost borrower/network/both + confusion |
| Training / loss curves | 09b_learning_curves.png | XGBoost log-loss per round, feed-forward NN loss per epoch, PR-AUC vs data size |
| Feature by outcome | 07c_bivariate.png | Written-off vs normal, per feature |
| Imbalance strategies | 02b_imbalance.png | Class weighting vs SMOTE variants |
| Confusion matrix (deployed) | 03_confusion.png | Deployed monotone model at the ≥80% recall point |
| Confusion matrix (selection) | 03a_confusion_unconstrained.png | Chosen model before the realism constraint |
| Confusion matrix (calibrated) | 03b_confusion_calibrated.png | After isotonic calibration |
| Confusion grid | 06_confusion_grid.png | Confusion matrices for all models |
| SHAP summary | 04_shap_summary.png | Feature importance / direction |
| Permutation importance | 04b_permutation_importance.png | Model-agnostic importance (confirms SHAP) |
| Calibration | 05_calibration.png | Predicted vs observed default rate |
| ROC & PR curves | 05_roc_pr_curves.png | ROC and precision-recall curves |
| Unsupervised vs supervised | 08_unsupervised_vs_supervised.png | Anomaly detectors vs the model |
| Riskiest communities | 10_community_default.png | Write-off rate of the riskiest guarantee communities (Louvain) |
| Default contagion | 10_contagion.png | How far a default spreads through the guarantee network |
| Guarantee community | 10_network_sample.png | One community; red = written off |
| Loan survival | 11_survival_km.png | Kaplan-Meier survival by loan-size tier |
| Borrower segments | 12_cluster_scatter.png | KMeans segments (PCA view) |

*(Additional-model figures 10_*, 11_*, 12_* are in `models/guarantorlens_new/`, generated by `notebooks/extra_models.ipynb`.)*

## Diagrams (generated — in `docs/diagrams/`)
| Diagram | File | Goes in |
|---|---|---|
| ML pipeline | 01_ml_pipeline.png | Ch3 (Proposed Model Diagram) |
| System architecture | 02_architecture.png | Ch3 (System Architecture) |
| Entity-Relationship Diagram | 03_erd.png | Ch3 |
| Class diagram | 04_class_diagram.png | Ch3 |
| Gantt chart | 05_gantt.png | Ch1 (Research Timeline) — **adjust the weeks to your real dates** |
| Use-case diagram | 06_use_case.png | Ch3 |
| Data-flow diagram (level 1) | 07_data_flow.png | Ch3 |
| Sequence diagram (assess a loan) | 08_sequence.png | Ch3 |

All built from the real code (`app/models.py`, `app/scoring.py`), so fields and classes match the report text — this directly answers the marker's "Technical Diagrams" comments (ERD field consistency, missing domain classes, small Gantt).
