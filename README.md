# guarantorLens_mission_capstone_ML
ML repo of my ALU mission capstone project

An explainable, **network-aware decision-support tool** for SACCO loan officers. It scores a loan's
default risk using both the borrower's own details **and** the structure of their guarantor network,
and explains every score in plain language. Capstone, BSc Software Engineering (ALU).

> **Track:** ML. **This repo is the primary submission**; the Frontend and Backend repos are
> cross-linked below.

---

##  Submission links 

| Item | Link |
|---|---|
| **Video demo (YouTube, 5-10 min)** | https://share.vidyard.com/watch/84BXUMBe5Ynbbfnr1bBDf4 |
| **Frontend - live app** | https://guarantor-lens-mission-capstone-fe.vercel.app/login |
| **Backend - Swagger / API docs** | https://guarantorlens-mission-capstone-be.onrender.com/docs |

### Repositories

| Component | Repo | Hosted on |
|---|---|---|
| **Frontend** (React) | https://github.com/Best-Verie/guarantorLens_mission_capstone_FE| Vercel |
| **Backend** (FastAPI + PostgreSQL) | https://github.com/Best-Verie/guarantorLens_mission_capstone_BE | Render  |
| **ML / Model** (this repo) | https://github.com/Best-Verie/guarantorLens_mission_capstone_ML| Google Colab + model artifact |

---

## What to look at first 

This is an ML project with a full-stack demo around it. The quickest path:
1. **Watch the video demo** (link above) - it walks the whole flow.
2. **Open the live app** and use the navigation: **Sign in → Dashboard → Assess a loan → Result → Download report**, and **Member**.
3. **Open Swagger** (`/docs`) and try `GET /health` and `POST /assess-risk`.
4. **Open the notebook** for the data visualizations, model architecture, and metrics.

---

## Tech stack & environment

| Layer | Tools |
|---|---|
| Frontend | React (Vite), Tailwind css, responsive layout |
| Backend | Python, FastAPI (auto Swagger at `/docs`), Uvicorn |
| Database | PostgreSQL  |
| ML | Python, pandas, NetworkX, scikit-learn, imbalanced-learn (SMOTE), XGBoost, SHAP, Matplotlib |
| Notebook | Google Colab |
| CI/CD | GitHub + platform auto-deploy (Vercel / Render); GitHub Actions for build checks |
| Secrets | `.env` files + platform env vars (no secrets in git) |

---



##  ML Track - the model notebook

**Notebook:** [`notebooks/train.ipynb`](notebooks/train.ipynb) - runs end to end on
Google Colab. It contains the three required parts:

1. **Data visualization & data engineering** - class balance, amount/savings/rate distributions,
   salary-missingness by class, guarantors-per-loan, a feature-correlation heatmap, the guarantor
   network graph, and leakage-safe feature engineering (every feature computed **as of the
   disbursement date**).
2. **Model architecture** - preprocessing pipeline (`impute → scale → SMOTE → classifier`),
   Logistic Regression, Random Forest, XGBoost, and a feed-forward neural net
   (Dense 32 ReLU → Dropout 0.3 → Dense 16 ReLU → Dense 1 Sigmoid; Adam; binary cross-entropy).
3. **Initial performance metrics** - recall (primary), precision, F1, ROC-AUC, PR-AUC, confusion
   matrix, and SHAP importance, comparing **baseline (individual features)** vs
   **augmented (+ guarantor-network features)**.

**Deployment option (ML):** the trained artifact `models/guarantorlens_xgb.joblib` is served by the
backend's `POST /assess-risk` endpoint and exercised through **Swagger UI** (see Backend below).


### Run the notebook (Colab)
1. Open `notebooks/train.ipynb` in Google Colab.
2. Upload the six branch workbooks when prompted, or mount Drive and point `DATASET_DIR` to the folder.
3. Run all cells. The notebook saves and downloads a zipped output folder with the model artifact,
   leakage audit tables, leaderboard, network-lift table, and JSON files for the backend.

### Re-train with 2025/2026 branch workbooks in Colab

The training pipeline lives in [`notebooks/train.ipynb`](notebooks/train.ipynb), not a separate
`.py` script. It reads all branch Excel files, collapses guarantor-level rows to one row per loan,
keeps the latest yearly snapshot when a loan appears in both 2025 and 2026, and computes borrower
plus guarantor-network features as of the loan disbursement date.

The notebook has an executable leakage policy. It blocks outcome/post-outcome fields
(`Days in Arrears`, `Repayment Status`, `Last Payment Date`, unpaid balances), mutable yearly
snapshot values (`Savings`, `Salary`, ratios derived from them), and calendar/vintage shortcuts
from entering model training. If any blocked feature is added to the model allowlist, the notebook
raises an error before fitting.

Current main run, using Gasabo, Kicukiro, and Nyarugenge 2025/2026 files:

| Item | Result |
|---|---:|
| Raw rows | 54,775 |
| Unique loans | 10,772 |
| Matured cohort | 6,330 loans, 4.6% defaults |
| Test split | 1,584 loans, 66 defaults |
| Best model | XGBoost + SMOTE, borrower + network features |
| Leak-free deployed features | interest rate, amount, prior borrower history, prior guarantor-network history |
| Test ROC-AUC / PR-AUC | 0.824 / 0.196 |
| Threshold policy | Recall-first, target recall 0.80 |
| Defaults caught at threshold | 53 of 66 (80.3% recall) |
| Precision at threshold | 13.8% |
| Supervised network PR lift vs borrower-only | +0.0381 |

The notebook now exports tuning tables for every model family and feature set, plus visual evidence:
`03_visual_summary.png`, `04_model_leaderboard.png`, `05_network_contribution.png`, and CSV tuning
tables such as `tuning_borrower_plus_network_xgb_smote.csv`. It also reports anomaly-detection
baselines (`IsolationForest`, `OneClassSVM`, `LocalOutlierFactor`, and kNN distance). These are
useful for the defense because defaults are rare, but the notebook does not let them use blocked
leakage features either.

---

##  Frontend (see frontend repo)

- React (Vite) dashboard for loan officers. Pages: sign-in, sign-up, dashboard, assess-a-loan form,
  result (risk gauge + SHAP reasons + guarantor network), member view, reports list, printable report.
- Responsive layout; calls the backend via `VITE_API_URL` (env var).
- **Setup:** `npm install` → `npm run dev` (local) / `npm run build` (prod). Full steps in the frontend repo README.
- **Deployed:** https://guarantor-lens-mission-capstone-fe.vercel.app/login.

##  Backend (see backend repo)

- FastAPI service. Endpoints: `GET /health`, `POST /assess-risk` (main), 
- Loads `guarantorlens_xgb.joblib`, rebuilds the same as-of features, returns risk score + SHAP reasons + network.
- PostgreSQL for loans/members/guarantees; config via env vars (`DATABASE_URL`, `MODEL_PATH`).
- **Setup:** `pip install -r requirements.txt` → `uvicorn app.main:app --reload`. Full steps in the backend repo README.
- **Deployed:** https://guarantorlens-mission-capstone-be.onrender.com  •  Swagger: https://guarantorlens-mission-capstone-be.onrender.com/docs.

### Database schema (overview)
- **members** (`member_id`, `opening_date`, `branch`)
- **loans** (`loan_id`, `member_id`, `amount`, `disbursement_date`, `rate`, `outcome`)
- **guarantees** (`loan_id`, `guarantor_member_id`, `date_guaranteed`)

(One member can guarantee many loans; one loan can have many guarantors.)

---

##  Deployment plan

| Component | Host (free tier) | How it deploys | URL |
|---|---|---|---|
| Frontend | Vercel | Auto-deploy on push to `main` | https://guarantor-lens-mission-capstone-fe.vercel.app/login |
| Backend (API + Swagger) | Render  | Auto-deploy on push; `uvicorn` web service | https://guarantorlens-mission-capstone-be.onrender.com/docs |
| Database | Neon / Supabase (Postgres) | Managed instance, `DATABASE_URL` env var | n/a |
| Model | Colab notebook + `joblib` artifact served by the API | Re-train in Colab, commit/upload artifact | via `/assess-risk` |

- **CI/CD:** GitHub Actions runs build/lint on each push; Vercel and Render redeploy automatically from `main`.
- **Config & secrets:** all environment-specific values (`VITE_API_URL`, `DATABASE_URL`, `MODEL_PATH`)
  come from env vars / platform settings. No secrets or real data are committed.

---

## Repo structure (this repo)

```
guarantorlens-ml/
├── notebooks/train.ipynb   # viz + engineering + architecture + metrics (Colab)
├── src/                                # feature code reused by the backend
├── models/                             # trained artifact (git-ignored, regenerated by the notebook)
├── reports/figures/                    # exported charts / screenshots
├── data/                               # real data LOCAL ONLY (git-ignored, never pushed)
├── requirements.txt
└── README.md
```

## Data & privacy

Real Umwalimu SACCO records are used under authorization, fully **pseudonymised**, and **never
committed** (`data/` is git-ignored). The tool is **decision support for loan officers, not automatic
approval**, and reports association, not causation.
