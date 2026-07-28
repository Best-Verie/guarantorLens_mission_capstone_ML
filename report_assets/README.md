# GuarantorLens — Report Assets (code-related chapters)

Everything you need to write the code chapters of the final report, in one folder. **Write the prose in your own words** (Turnitin); these are facts, figures, tables, and bullet points to write *from*.

## Folder map
| Folder / file | What it is |
|---|---|
| `author_kit_ch3-5.md` | The main kit — now **Ch3, 4, 5 and 6** skeletons + real facts + figures/tables/code + citations |
| `test_results.md` | Real test output (28 passed) mapped to the template's testing categories |
| `diagrams/` | **8** diagrams from the real code (pipeline, architecture, ERD, class, use-case, data-flow, sequence, Gantt) |
| `figures/` | 11 result figures exported by the notebook (leaderboard, confusion, SHAP, calibration, ROC/PR, etc.) |
| `result_tables/` | 13 CSVs behind the tables (metrics, per-model grids, imbalance, confusion, SHAP, network lift) |
| `screenshots/` | The captioned app screenshot deck |
| `defense_guide/` | The lecturer's defense guide (source for the checklist and Q&A below) |

## Which asset goes where (report map)
| Report section | Use |
|---|---|
| Ch1 Research timeline | `diagrams/05_gantt.png` (set your real weeks) |
| Ch3 Proposed model diagram | `diagrams/01_ml_pipeline.png` |
| Ch3 System architecture | `diagrams/02_architecture.png` |
| Ch3 ERD | `diagrams/03_erd.png` |
| Ch3 Class diagram | `diagrams/04_class_diagram.png` |
| Ch3 Dataset description | `figures/07_data_overview.png`, `07_feature_correlation.png` |
| Ch4 Screenshots | `screenshots/…captioned.docx` |
| Ch4 Testing | `author_kit` §4.3 (unit/integration/functional/acceptance) + the `14 passed` evidence |
| Ch5 Results | all of `figures/` + `result_tables/` (tables 5.1–5.4 in the kit) |

> **Chapter-numbering note.** The defense guide uses a 5-chapter ML layout (Ch4 = Implementation+Results+Discussion, Ch5 = Conclusion). Your **ALU template** uses Ch4 = Implementation & Testing, **Ch5 = Results & Discussion, Ch6 = Conclusions**. Follow the **ALU template numbering**; the guide's *content requirements* still apply, just spread across your Ch4–Ch6.

---

## Guide checklist → our evidence
The guide says a strong ML capstone must prove these. We already have each — cite the evidence.

| Guide requirement | Our evidence |
|---|---|
| **Real problem** | Loan write-offs at Umwalimu SACCO; officers lacked a view of aggregate guarantor exposure |
| **Clear gap** (soften wording) | "the reviewed workflow did not provide aggregate guarantor-exposure risk" |
| **Appropriate data** (source, rows, features, target, missing, class dist, leakage) | 11-branch sample, 11,015 loans, 18 features, target = written-off (2.1%), leakage screen — `figures/07_data_overview.png`, `01_leakage_evidence.png` |
| **Baseline (compared to what?)** | Base rate PR 0.021; borrower-only vs +network; unsupervised anomaly detectors — `result_tables/00_final_metrics.csv`, `08_unsupervised_comparison.csv` |
| **Right metrics for imbalance** (not accuracy) | PR-AUC (headline), ROC-AUC, recall-first threshold, precision, F1, confusion matrix, calibration — `figures/03_confusion.png`, `05_calibration.png`, `05_roc_pr_curves.png` |
| **Model evaluated & compared** | 5 model families leaderboard + SMOTE-vs-class-weighting — `figures/02_leaderboard.png`, `02b_imbalance.png` |
| **Required diagrams** | pipeline, architecture, ERD, class — all in `diagrams/` (use-case + data-flow optional, can add) |
| **Ethics** | anonymized IDs; decision-support **not** live lending; association not causation; human-in-the-loop (officer proposes, manager decides) |
| **Honest limits** | 11-branch sample of ~30; write-off label delay; no fairness audit; binary label hides intermediate states |
| **Prototype driven by the ML model** | deployed tool; every score shows "source: model"; screenshots in `screenshots/` |

---

## Defense Q&A prep (from the guide's defense questions)
Bullet answers to expand in your own words on the day.

- **What is your target variable?** A binary label: **written-off (1) vs normal (0)**. Written-off is a severe, late-stage outcome; it is *not* the same as non-performing or restructured. `Repayment Status` (Repaid/Active) is the post-write-off recovery state, not the label.
- **Why is this an ML project (not just an app)?** A rare binary outcome (2.1%) predicted from 18 interacting borrower + guarantor-network features across 11,015 loans; simple rules cannot weigh those interactions or rank risk. The app is the delivery layer.
- **How did you avoid data leakage?** Every feature is computed **as of the disbursement date**; post-outcome fields are blocked; `time_since_last_loan` was dropped; evaluation used **borrower-grouped cross-validation**. `figures/01_leakage_evidence.png`
- **Why XGBoost?** Tabular, imbalanced data; handles missing values and interactions; supports monotone constraints; it topped the model leaderboard (CV PR-AUC 0.659). It is calibrated (isotonic) so the probabilities are usable.
- **What baseline did you use?** The base rate (PR 0.021), plus borrower-only vs +network, plus unsupervised anomaly detectors — the model beats all of them.
- **What metrics and why?** PR-AUC as the headline (rare positives make accuracy misleading), ROC-AUC, and a recall-first operating threshold: deployed recall 0.80 (181/226 defaults caught), precision 0.153, F1 0.256 (deployed PR-AUC 0.603 / ROC 0.932).
- **Does the network help?** Honest: predictive alone (PR 0.237, ROC 0.827 vs 0.021 baseline) and lifts ranking (ROC 0.942→0.952), but the incremental PR lift is within noise because risky borrowers cluster with risky guarantors (homophily).
- **What are your limitations?** 11-branch sample of ~30 branches; write-off label delay; binary label; no fairness/branch-parity audit; the model reports association, not causation.
- **Defense one-liner:** "This is a proof of concept: it validates the framework — a leakage-safe, calibrated, explainable, network-aware risk model in a working decision-support prototype — not full real-world deployment."

---

## Done in this round
- **Chapter 6** kit (Conclusions & Recommendations) added to `author_kit_ch3-5.md`.
- **Use-case, data-flow, and sequence** diagrams generated — the full diagram set the guide lists (8 total in `diagrams/`).
- **All 5 testing categories coded, run, and passing** (28 tests) — see `test_results.md`.
- **Gap-audit fixes** folded into Ch3: a dedicated **3.5 Ethical Considerations** (privacy, fairness/guilt-by-association, human-in-the-loop, no live decisions, retention/breach), a **missing-values** analysis, and an explicit **validation split** (5-fold borrower-grouped, out-of-fold CV).

## Still optional (say the word and I'll build)
- **Ch1 + Ch2 author's kits** (Introduction + Literature Review) in the same format, with the marker-fixes woven in.
- Convert the kit to a **Word (.docx)**.
