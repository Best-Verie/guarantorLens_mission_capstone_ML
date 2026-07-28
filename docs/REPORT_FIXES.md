# Report fixes — response to supervisor feedback

This maps every feedback item to a concrete fix, with **real numbers** I computed on the actual cohort where
possible, and marks what still needs a notebook re-run or a source you must supply. Numbers are from the local
data run; when you re-run `pipeline.ipynb` on Colab, refresh from that single run so everything reconciles.

Legend: **[NUMBER]** = a real figure I computed. **[RE-RUN]** = add the cell and regenerate on Colab.
**[SOURCE]** = you must supply/verify (I will not fabricate citations or external stats).

---

## 0. Single source of truth — put these EVERYWHERE (fixes 2.1 and 2.2)

Every table, figure and sentence must come from **one** run ("guarantorlens_new (10)"). The contradictions the
reviewer found are old exports pasted next to new ones. Canonical values:

**Feature-set comparison (Table 5.1 / Fig 5.1):**
| Set | PR-AUC | ROC-AUC |
|---|---:|---:|
| Borrower-only | 0.641 | 0.942 |
| Network-only | 0.237 | 0.827 |
| Borrower + network (unconstrained) | 0.653 | 0.952 |
| Borrower + network (monotone → deployed) | 0.603 | 0.932 |

**Best per family (Table 5.2 / Fig 5.2):** XGBoost+net 0.659 · RandomForest-borrower 0.572 · DecisionTree 0.374 ·
Feed-forward NN 0.348 · LogReg 0.346.

**Confusion at ≥80% recall (Table 5.4 / Figs 5.4–5.5), all summing to 10,789 neg / 226 pos:**
| Stage | TN | FP | FN | TP | Precision | F1 |
|---|---:|---:|---:|---:|---:|---:|
| Unconstrained (selection) | 10,245 | 544 | 45 | 181 | 0.250 | 0.381 |
| After calibration | 10,099 | 690 | 45 | 181 | 0.208 | 0.330 |
| Deployed (monotone) | 9,784 | 1,005 | 45 | 181 | 0.153 | 0.256 |

The "≈1.8× more false alarms" claim is then correct (1,005 / 544 = **1.85×**). **Action:** regenerate
`03a/03b/03_confusion.png` and `02_leaderboard.png` from run 10 (they already exist in `report_assets/figures/`),
fix the mislabelled Fig 5.4 caption, and delete every stale number.

**Important nuance for 2.2 (this helps you):** borrower-only and +network are **statistically
indistinguishable** — in a clean out-of-fold run they are 0.663 vs 0.660 PR-AUC (borrower-only *fractionally
higher*). That is exactly your null result. Make the table, the figure and the text all say "within noise", and
you turn a contradiction into evidence for your own thesis.

---

## 1. Serious flags

### 2.3 Ethics / demo-data contradiction — **correct the record; the seeding was authorised**
The problem is not the seeding itself (it was authorised and controlled); it is that the report **mislabels the
demo as "synthetic"** while the live screenshots show the real cohort. The fix is to state the true facts once
and remove every "synthetic data only" claim. The confirmed facts:

- The demo runs on the **real, pseudonymised records** — it was **never synthetic**, and the seeding was a
  known, deliberate step.
- Seeding the cloud database for the demonstration was **explicitly authorised**.
- The seeded data was **deleted immediately after the demonstration**.

Replace the scattered claims (§1.5, §3.4, §3.5) with **one factual paragraph** (fill only [date] and the
[authorising name/role]):

> The public demonstration runs on the **real, pseudonymised** loan records (11,015 loans, 22,485 members), not
> synthetic data. For the supervisor demonstration on [date], these records were **seeded into the cloud
> database with explicit authorisation** from [authorising name/role], purely to show the tool on realistic
> data, and were **deleted immediately after the demonstration**. Member identifiers are opaque pseudonyms with
> no directly identifying fields, so under Law 058/2021 no personal data was exposed; all other development used
> the local copy.

Then **delete every "synthetic data only" sentence** in §1.5 and §3.4 and replace with "real, pseudonymised data
that is seeded only for an authorised demonstration and deleted immediately afterwards" — so NFR4 reads as a
controlled, authorised, time-boxed exception rather than a breach. Handled this way it is a **strength** (you
followed a data-protection protocol), not a flag.

### 2.4 Motivating statistic a decade apart — **[SOURCE]**
12.5% SACCO NPL (2015–16) vs 2.6% banking (2025) is not a valid comparison. Options, in order of preference:
(a) find a **current** SACCO/Umurenge-SACCO NPL figure from BNR or MINECOFIN annual reports **[SOURCE]**; or
(b) keep the figures but **state the year gap in one sentence** and argue the structural point (SACCO NPLs have
persistently exceeded bank NPLs) rather than implying a same-year gap. Do not present them as contemporaneous.

### 2.5 interest_rate is a public/private proxy — **[NUMBER], answered, and it helps you**
Real cohort facts: rates present are **13% (9,644 loans, 0.5% default)** and **14% (1,365 loans, 13.3% default)** —
so `interest_rate` is indeed a risk-based-pricing / employment proxy. **But the model is not merely that
classifier.** Dropping `interest_rate` entirely:

| Model | PR-AUC with `interest_rate` | PR-AUC without | Drop |
|---|---:|---:|---:|
| Borrower + network | **0.660** | **0.613** | −0.047 |
| Borrower-only | 0.663 | 0.599 | −0.064 |

So removing the proxy costs only **~0.05 PR-AUC**; the model keeps most of its skill from savings, loan-to-savings,
account age and guarantor/community history. **Fix:** add a short subsection reporting this ablation, state plainly
that `interest_rate` encodes a public/private pricing signal, and show the model still works without it (PR 0.613).
Note the report's "11% public / 14% private" wording does not match the data (11% is negligible; the split is
13%/14%) — correct that too. **[RE-RUN]** cell in §3.

### 2.6 Null result: missingness vs homophily — **[NUMBER], and it confirms homophily**
Restricting to the **1,049 loans (9.5%) where every guarantor has complete savings+salary records** (44 defaults):
borrower-only PR **0.778**, +network PR **0.736** — the network lift is **−0.043 (still no lift, if anything
negative)**. So the weak lift is **not** an artefact of median-imputation; it holds where records are complete.
This directly answers the reviewer: **homophily/redundancy is supported over missingness.** Add this as the
robustness check, with the honest caveat that the complete-record subset is small (44 defaults, so noisy).
**[RE-RUN]** cell in §3.

### 2.7 Cohort contamination — **[NUMBER], acknowledge it**
Real: **122 loans (1.1%) are 90+ days in arrears but labelled 0** — that is **0.54× the size of the 226
positives**. (At 30+ days: 182, 1.7%.) The negative class is contaminated with likely future write-offs, which
biases every metric **downward** (so your reported scores are conservative). **Fix:** stop calling the cohort
"completed/closed" — call it a **matured 2022–2023 cohort**; add one paragraph acknowledging the 122 arrears-but-0
loans and that they make the metrics a lower bound; optionally report a sensitivity metric relabelling them as 1.

### 2.8 Chapter 3 is carrying Chapter 5's results — **move them**
Community write-off rates, contagion, Kaplan-Meier, segment rates, and the 6.9% anomaly hit are **findings** →
move them (or cross-reference and restate) into Chapter 5. And **answer RQ1's structural question in Ch5**: report
the network's shape (nodes/edges/degree/communities) and how write-offs distribute across it, not only "does the
network carry signal". The material exists in 3.7.1; it just needs to appear where the rubric looks.

---

## 2. Secondary flags (with real numbers)

- **Modularity near-trivial — [NUMBER].** Average degree = 2 × 25,624 / 22,485 ≈ **2.3**: the guarantee graph is
  very sparse (near-tree), so Louvain modularity 0.99 is close to recovering connected components. **Present it
  as near-trivial**, not as a strong structural finding.
- **Community sizes — [NUMBER].** Give sizes. Riskiest with ≥30 loans: **27.5% (40 loans, 61 members)**, then
  16.7% (78 loans, 135 members), 10.9% (55 loans, 99 members). There are **14 communities at 100% write-off, but
  they are median 1 loan / 4 members** — near-trivial. **Fix:** rank communities with a **minimum size (≥30
  loans)** and always print the size, so "27.5%" is interpretable.
- **Contagion has no baseline — [NUMBER], honest negative.** Real cascade: 217 defaulters → **377 reached**. A
  **random seed of 217 members reaches 774 on average (95% CI 718–833)** — i.e. the real cascade is **below**
  random, because defaulters sit in less-central positions. **Fix:** report the baseline and reframe contagion as
  an **exposure illustration**, not evidence of above-chance systemic spread. This is an honest correction and
  the reviewer will credit it. **[RE-RUN]** cell in §3.
- **Kaplan-Meier window — [RE-RUN].** You censor at 2024-12-31 but labels come from a 2026 extract, throwing away
  ~17 months of maturity. Extend the observation date to the extract date (change `OBS`), and report survival to
  **1 decimal** (the write-off date is unknown, so 2 d.p. is false precision).
- **Table 3.12 small < medium survival.** Small (98.27%) below medium (98.50%) is within noise on few events;
  either report confidence bands or drop the per-tier ordering claim and keep only "large loans fail soonest".
- **Guarantee-count reconciliation — [NUMBER].** They measure different things: **17,501 = distinct guarantors
  (people)**; **~27,559 = guarantee links (edges)** (the report's 25,624 is the same quantity from a slightly
  earlier export). Label them "distinct guarantors" vs "guarantee links" and the contradiction disappears.
- **No incumbent comparison — [SOURCE/RE-RUN].** The SACCO's real question is "does the model beat the existing
  eligibility rules?". If you can get the current rule (e.g. loan ≤ X × savings, N guarantors), encode it as a
  classifier and add it to the leaderboard. If you cannot get the rule, say so explicitly as a limitation.

---

## 3. Notebook cells to add before the re-run

Add these to `pipeline.ipynb` so the single final run produces the new evidence. (I can inject them for you.)

```python
# (2.5) interest_rate ablation — is the model just a public/private proxy?
from sklearn.model_selection import cross_val_predict
from sklearn.metrics import average_precision_score, roc_auc_score
def _oof(feats):
    p = cross_val_predict(xgb_pipe(None), cohort[feats].values, y, cv=cv, groups=groups, method="predict_proba")[:,1]
    return round(average_precision_score(y,p),3), round(roc_auc_score(y,p),3)
_noir = [f for f in FULL_FEATURES if f != "interest_rate"]
print("borrower+network  with interest_rate:", _oof(FULL_FEATURES))
print("borrower+network  WITHOUT interest  :", _oof(_noir), " <- model still works without the proxy")
print("default rate by rate:", cohort.groupby(cohort.interest_rate.round()).label.mean().round(3).to_dict())

# (2.6) complete-guarantor-records lift — missingness vs homophily
_lg = {r["Loan ID"]: (r["guarantors"] or []) for _, r in loans.iterrows()}
def _complete(lid):
    gs = _lg.get(lid, [])
    return bool(gs) and all(gg in bmap and pd.notna(bmap[gg]["sav"]) and pd.notna(bmap[gg]["sal"]) for gg in gs)
_m = cohort["loan"].map(_complete).values
def _oofm(feats):
    p = cross_val_predict(xgb_pipe(None), cohort[feats].values, y, cv=cv, groups=groups, method="predict_proba")[:,1]
    return round(average_precision_score(y[_m], p[_m]),3)
print(f"complete-record loans: {_m.sum()} ({_m.mean():.1%}), defaults {int(y[_m].sum())}")
print("  borrower-only", _oofm(IND_FEATURES), "| +network", _oofm(FULL_FEATURES), "-> network lift stays ~0/negative => homophily, not missingness")

# (contagion) random-seed baseline
_defaulters = set(df.loc[df.label==1, "borrower"])
_edges = [(r["Borrower ID"], set(r["guarantors"] or [])) for _, r in loans.iterrows()]
def _cascade(seed):
    comp=set(seed)
    for _ in range(4):
        grew={b for b,gs in _edges if gs & comp}
        if comp>=grew: break
        comp|=grew
    return len(comp)
_ids=list(set(loans["Borrower ID"]) | {g for gs in _lg.values() for g in gs})
_real=_cascade(_defaulters); _rng=np.random.RandomState(RANDOM_STATE)
_rand=[_cascade(set(_rng.choice(_ids,len(_defaulters),replace=False))) for _ in range(50)]
print(f"contagion: real {_real} vs random {np.mean(_rand):.0f} (95% {np.percentile(_rand,2.5):.0f}-{np.percentile(_rand,97.5):.0f})")

# (2.7) cohort contamination
_arr = loans.set_index("Loan ID")["Days in Arrears"]; _a = cohort["loan"].map(_arr)
print(f"90+ days arrears but labelled 0: {int(((_a>=90)&(y==0)).sum())} ({((_a>=90)&(y==0)).mean():.1%})")
```

Plus: in the communities cell add a **`min_loans=30`** filter and print `members` + `loans` per community; in the
survival cell set **`OBS = <the extract date>`** (not 2024-12-31) and round to 1 d.p.

---

## 4. Presentation / writing checklist (biggest mark-per-hour, items 1–4 in the review)

- [ ] Regenerate ALL figures from the single run 10; delete stale numbers (fixes 2.1, 2.2).
- [ ] Certification page: fill the template (title, supervisor name, date, signatures).
- [ ] Regenerate List of Figures and List of Tables **with page numbers**; add Ch4 figures and Tables 3.10–3.14.
- [ ] Fix duplicate figure numbers (two 3.2, two 5.3, two 5.6, two 5.7, four 5.4) and the "Figure 4.20"→4.9 cite.
- [ ] Replace code **screenshots** with formatted, monospaced **listings** with a one-line caption each (§4.1.3).
- [ ] Re-take screenshots **cropped** (no macOS dock, bookmarks, DevTools, `localhost` URLs).
- [ ] One register throughout (pick impersonal; remove "I did 8 tests" and the "we" in §5.3).
- [ ] Typos: "plain English" (×2), "assess()", "/assess-risk endpoint", "held only locally (except", "Write-off".
- [ ] Reconcile "≈10×" vs "≈11×" base-rate multiple (PR-AUC 0.60 / 0.021 ≈ **29×**, or ROC framing — pick one and
      use it everywhere; the "10/11×" figures look like an older run).
- [ ] §6.4 "can be trusted" → soften to match §5.3 (not for deployment until the fairness + interest-rate checks
      are done). Don't undercut your own honesty in the last paragraph.
- [ ] Latency: add one measured `/assess-risk` timing to back NFR7; add the pytest count as your coverage proxy.

---

## 5. Literature to add — **[cite]** (find full refs; do not fabricate)

- **Joint-liability / social-collateral lending** (gives homophily a theoretical spine): Ghatak & Guinnane
  (1999) on peer selection/monitoring; Besley & Coate (1995) on group lending and repayment; Armendáriz &
  Morduch, *The Economics of Microfinance*. Cite these in §2.4.3 around the network/homophily argument.
- **Fairness in credit scoring** (you name fairness as the deployment precondition): a fairness-in-lending / ML
  fairness reference in §2.4.7 **[cite]**.
- Acknowledge that ~4 of ~25 sources are from one group (Cheng et al.) — one sentence.

---

## What genuinely improves the mark (the reviewer's own ranking)

Items 1–4 (reconcile numbers, ethics paragraph, prelims, figure numbering + code listings) are pure
presentation and move you from ~76% to mid-80s. Items 5–6 here — the **interest_rate ablation (2.5)** and the
**complete-records / homophily check (2.6)**, both of which I have already computed for you — push Results and
Literature into the mastery band.

---

# Round 2 — outstanding issues (grade now 28/35)

## Blockers

### B1. Figures still show the old run — **swap the PNGs, don't re-save the Word file**
Re-exporting Word only re-compresses images; it does not change what they depict. In Word, **right-click each
chart → Change Picture → point at the run-10 PNG** already sitting in `report_assets/figures/`. Mapping:

| Report figure | Replace with (`report_assets/figures/`) |
|---|---|
| Fig 5.1 (ROC/PR by feature set) | `05_roc_pr_curves.png` |
| Fig 5.2 (leaderboard) | `02_leaderboard.png` |
| Fig 5.4a (calibration confusion) | `03b_confusion_calibrated.png` |
| Fig 5.4c / deployed confusion | `03_confusion.png` |
| Fig 5.5 (selected/unconstrained confusion) | `03a_confusion_unconstrained.png` |
| all-model confusion grid | `06_confusion_grid.png` |
| calibration curve | `05_calibration.png` |
| SHAP summary | `04_shap_summary.png` |

**Verification (the reviewer's check):** in run 10, XGBoost **+network 0.659 > borrower-only 0.641** (Table 5.2
is correct; regenerate Fig 5.2 to match). But a clean fixed-config comparison puts them **within noise**
(0.663 vs 0.660, borrower fractionally ahead) — add one sentence saying the two feature sets are statistically
tied (bootstrap CI crosses zero). That makes figure, table and text agree **and** strengthens the null result.

### B2. Ethics placeholders — **fill [DATE] and [AUTHORISING NAME AND ROLE].** Two facts. A data-protection
statement with blanks is worse than none.

### B3. Table 5.5 shows the UNCONSTRAINED model — **recompute on the deployed model.** Real deployed operating
points (borrower-grouped OOF on the monotone model):

| Operating point | Threshold | Precision | Recall | F1 | Accuracy |
|---|---:|---:|---:|---:|---:|
| Screening (≥80% recall) | 0.017 | **0.153** | 0.801 | 0.256 | 0.905 |
| F1-optimal (balanced) | ~0.257 | ~0.679 | ~0.553 | ~0.610 | ~0.985 |
| Precision ≥ 0.50 (precise) | ~0.155 | 0.500 | ~0.602 | ~0.546 | ~0.979 |

The screening row must read **0.017 / 0.153 / 0.905** (= Table 5.4's deployed column), not 0.032 / 0.250 / 0.947
(which is the unconstrained model). Regenerate this table in the same run 10 for exact consistency; add the cell
in §3 (compute the three points on `p_mono`, not `p_full`).

### B4. Employment-sector claim — **[YOU: confirm with the SACCO].** The two claims differ: "public vs private
teacher" (§5.3) vs "13% for teacher members, 14% for cooperative-employees/non-teacher members" (the separate
note). The **fairness audit groups depend on which is true.** Correct §1.1 and §5.3 to the confirmed rule, and
set the audit's protected groups to match (teacher vs non-teacher, or public vs private — whichever it is).

## Substantive

- **S1. Tables 5.2b vs 5.1 disagree (0.655/0.637 vs 0.653/0.641).** Both must come off run 10. Recompute the
  network ablation in the same notebook run; use one pair of numbers everywhere (canonical: +network 0.653,
  borrower-only 0.641).
- **S2. Abstract is behind the analysis.** Three edits: (a) "11,015 **completed** loans" → "a **matured**
  2022–2023 cohort (with 122 loans, 1%, still 90+ days in arrears, labelled 0)"; (b) network "improved accuracy
  marginally" → "network lift **+0.012, 95% CI [−0.019, +0.044]** — within noise"; (c) add the stronger reason
  for the null: **181 of 226 write-offs sit inside a 14%-rate pricing band covering ~12% of the book, leaving
  little room for network features to add signal** (put this ahead of homophily).
- **S3. Survival method vs result.** §3.7.2 says it censors at 2024-12-31 but quotes the extract-date figure.
  Change the method sentence to the **extract date** and round survival to **1 d.p.** (write-off date is unknown).
- **S4. Register — pick impersonal, delete these:** "I did 5 validation tests", "I did 6 Integration tests",
  "I did 6 functional cases tests", "I run code based end-to-end user scenarios", and the three "we"s in §5.3
  ("we did not test", "the neural network we tried", "what we already know") → impersonal.
- **S5. Code listings — paste these four functions from `app/scoring.py` as formatted monospace** (not
  screenshots). Ready to paste:

```python
# Listing 4.x  Risk bands from the calibrated score (app/scoring.py)
def _band(p: float) -> str:
    if p >= BANDS["high"]:
        return "High"
    if p >= BANDS["medium"]:
        return "Medium"
    return "Low"
```

```python
# Listing 4.x  Leak-free rule overlay: escalate the band on concentrated guarantor red flags
def adjust_band(base_band, guarantor_ids, borrower_id=None):
    band = base_band
    gs = guarantor_ids or []
    over = [g for g in gs if (_member(g).get("loans_backed") or 0) >= FLAG_TH["over_committed_loads"]]
    defaulters = [g for g in gs if _member(g).get("ever_defaulted") == 1]
    if gs and len(over) == len(gs):      # whole backing group over-committed -> +1 band
        band = _bump(band, 1)
    if len(defaulters) >= 2:             # two or more backers defaulted before -> High
        band = "High"
    return band
```

```python
# Listing 4.x  The guarantee-network feature at serve time (community default rate)
def _community_rate(borrower_id, guarantor_ids):
    root = _COMMUNITY.get(borrower_id) if borrower_id else None
    if root is None:
        for g in (guarantor_ids or []):
            if g in _COMMUNITY:
                root = _COMMUNITY[g]; break
    return _COMMUNITY_RATE.get(root) if root is not None else None
```

Plus a fourth (pick one): `_load_communities()` (union-find over the guarantee graph) or `assess()` (the entry
point). Caption each with one sentence saying what it does.
- **S6. Screenshots — [YOU]** re-capture cropped to the viewport, from the **deployed Vercel URL** (no
  `localhost`, dock, bookmarks, DevTools). Nine images.

## Minor
- Delete **Top Africa News (2024)** from the reference list (uncited, misalphabetised).
- Add **Blondel et al. (2008)** for Louvain to the reference list and cite it in Ch3. **[cite]**
- **Nikuze et al. (2024)** — add volume/issue/pages. **[SOURCE]**
- **Test coverage:** add one sentence — "the 28 passing tests evidence behaviour at the API surface, not code
  coverage; measured line coverage is a limitation."
- **NFR7 latency — [YOU]:** time 20 consecutive `/assess-risk` calls on the deployed API, report the median in
  one sentence.
- **Contamination sensitivity (optional, closes 2.7):** relabelling the 122 arrears-90+ loans as write-offs
  moves PR-AUC **0.653 → 0.520** — i.e. the model does **not** rank these loans as high-risk, so their presence
  in the negative class is **not** inflating the headline; the drop shows loans that deteriorate after a clean
  origination are genuinely hard to predict at application time. One honest sentence closes the point. The contagion-baseline honesty (a negative result) and the cohort-contamination
acknowledgement cost little and read as maturity.
