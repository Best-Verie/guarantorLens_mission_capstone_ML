# GuarantorLens — 7-minute demo script & test cases

## Part 1 — 7-minute demo script (spoken word-for-word)

Paced with time markers and `[DEMO]` cues for the live app. Short problem, minimal architecture, heavy on
metrics + live demo + the honest finding.

**0:00 – 0:50 · Open + the problem** *(slide: title)*
> "Good morning. I'm Best Verie Iradukunda, and this is **GuarantorLens** — an explainable, network-aware
> loan-risk tool for **Umwalimu SACCO** in Rwanda. At a SACCO, loans aren't backed by a house — they're backed
> by **other members who guarantee them**, so the collateral is a *network of people*, and that network was
> invisible to the loan officer. Appraisal was manual, individual, and opaque, against a backdrop of high
> write-offs. My research question was: **does that guarantee network actually help predict which loans go
> bad — and can we make the decision explainable?**"

**0:50 – 2:00 · Data & method** *(slide: data / pipeline)*
> "The data: **11,015 real, anonymised loans**, **226 written off — a 2.1% failure rate**, across 11 of about
> 30 branches. The label is the SACCO's own write-off record. I built **18 features** — 11 about the borrower,
> 7 about the backers — using **only what's known on the day the loan is given**, and I screened out anything
> that leaks the future. I split **by borrower**, so a person's loans never sit in both training and testing.
> Because failures are rare, **I don't judge by accuracy** — approving everyone would be 98% accurate and
> useless. I use **PR-AUC**, which measures how well the model ranks the rare bad loans. I compared **five
> model families**, tuned the same way; **XGBoost won**. I then **calibrated** the score into an honest
> probability and added **monotone constraints** so it always behaves sensibly."

**2:00 – 3:00 · The main finding** *(slide: the finding)*
> "The honest headline: the network **is** predictive on its own — but added to the borrower's own details it
> adds **almost nothing**: a lift of **+0.012, with a confidence interval that crosses zero**. Statistically
> it's a tie. That's a **finding, not a failure** — a *boundary condition*: the big gains reported for
> corporate guarantee networks **don't transfer** to a teachers' cooperative, because people back others like
> themselves. The deployed model lands at **PR-AUC 0.60, ROC 0.93, catching 80% of write-offs**. So we keep the
> network **not for accuracy, but for what it explains**. Let me show you."

**3:00 – 5:30 · Live demo** *(switch to the app)*
- `[DEMO — Assess]` "A large loan, thin savings, and a backer who guarantees many loans." `[submit]` "Instantly
  a **High-risk** score, and in plain language *why*: the loan is large against savings, and this backer is
  **over-committed**. There's also an **'unusual application'** flag — atypical loans fail about three times
  more often."
- `[DEMO — monotone]` "It stays believable: I raise the savings — the score **can't go up**; I raise the
  loan — it **can't go down**."
- `[DEMO — fix-it advisor]` "It suggests a **real, stronger backer** from the same branch that would lower the
  risk — a concrete next step."
- `[DEMO — member / network]` "On a member's page: the **guarantee network** and the **contagion view** — if
  this member fails, exactly which loans are exposed."
- `[DEMO — monitoring]` "Monitoring shows how loans age — **98% still healthy at two years**; big loans slip
  earliest."
- `[DEMO — role gating]` "And it's governed: as an officer, if I try to record a manager's recommendation —
  **403, blocked**. The officer proposes; the manager decides."

**5:30 – 6:15 · Ethics + architecture** *(slide: architecture)*
> "Under the hood it's a **single FastAPI service** — the REST API with JWT auth, a scoring engine that loads
> the exported models, and a PostgreSQL database, with a React app on top. Ethically, the tool **never
> decides** — a human always does; every score has a plain-language explanation; exported reports **carry no
> client or guarantor identities**; and it follows Rwanda's **data-protection Law 058/2021**."

**6:15 – 7:00 · Conclusion + future work** *(slide: conclusion)*
> "The contribution is twofold. As **research**, an honest answer the SACCO didn't have: the guarantee network
> carries signal but adds no reliable accuracy — reported with the confidence interval that proves it. As a
> **system**, a deployed, explainable, network-aware tool that turns a slow, opaque appraisal into a fast,
> auditable one. Honest limitations: only 226 failures, a fairness check on the interest-rate signal still to
> run, and a forward-in-time test across all thirty branches — that's where I'd go next. Thank you — happy to
> take questions."

**Delivery tips**
- Rehearse the demo clicks blind; if the app lags, keep talking — don't debug live.
- If over time, drop the monitoring/survival bit first.
- Numbers to have memorised: **11,015 loans · 2.1% · deployed PR-AUC 0.60 / ROC 0.93 · recall 80% · lift +0.012 (CI crosses zero)**.
- Do **not** demo the extreme-savings slider (see the scope note at the end of Part 2).

---

## Part 2 — Score-assessment test cases

Worked test cases run through the **deployed model** (calibrated, monotone XGBoost, PR-AUC 0.60 / ROC 0.93).
Bands: **Low / Medium / High**; score is the 0–100 display value. All outputs below are **verified**, not
illustrative — they are what the live tool returns.

Guarantor keys used: **clean** = backers with no prior write-off and few guarantees; **over-committed** =
`Client120931` (backs 7 loans); **written-off** = `Client107745`, `Client113217` (prior write-offs).

---

## Table 1 — Risk spectrum and network flags

| # | Scenario | Amount | Savings | Salary | Rate | Guarantors | Band | Score | Flag |
|---|---|---:|---:|---:|---:|---|---|---:|---|
| 1 | Well-covered small loan | 300,000 | 2,500,000 | 400,000 | 13% | 2 clean | **Low** | 12 | — |
| 2 | Moderate loan | 3,000,000 | 600,000 | 350,000 | 14% | 2 clean | **Medium** | 47 | — |
| 3 | Big loan, thin savings | 12,000,000 | 30,000 | 150,000 | 14% | 2 clean | **High** | 74 | — |
| 4 | Over-committed backer | 2,000,000 | 300,000 | 300,000 | 14% | 1 over-committed + 1 clean | **Medium** | 59 | *Over-committed guarantor: Client120931 backs 7 loans* |
| 5 | Two written-off backers | 1,500,000 | 200,000 | 250,000 | 14% | 2 written-off | **High** | 72 | *Backed by 2 guarantors written off before* |

Cases 1–3 show the model spanning the full risk range on borrower finances alone; cases 4–5 show the
**guarantor-network flags** and the rule that escalates the band when the backing is weak.

## Table 2 — Behaviour (metamorphic) checks — the score must move the sensible way

| Check | What changes (all else fixed) | Result | Correct? |
|---|---|---|---|
| More savings lowers risk | 2M loan · savings 50,000 → 500,000 | Medium **56 → 51** | ✓ falls |
| Bigger loan raises risk | savings 500k · loan 1,000,000 → 5,000,000 | Medium 63 → **High 70** | ✓ rises |
| Higher interest rate raises risk | 4M loan · rate 13% → 14% | **Low 14 → Medium 65** | ✓ rises |

These are the **monotone-constraint** guarantees an officer can trust: more savings never raises the score, a
bigger loan never lowers it, and a higher (riskier) rate never lowers it.

## Table 3 — The "unusual application" flag (anomaly model)

| Scenario | Band | Unusual flag | Reading |
|---|---|---|---|
| Well-covered small loan (300k vs 2.5M savings) | Low | **yes** | *Unusual ≠ risky* — this profile is atypical (a very small loan against large savings), so it is flagged for a look, not marked risky. |
| Big loan, thin savings (case 3) | High | **yes** | Both High-risk **and** unusual — the strongest "look closer" combination. |

The anomaly flag is a **prompt, not a score**: applications flagged as unusual historically failed about
**3× more often** (6.9% vs 2.1%), so it is a cheap first screen.

---

## Automated test suite (`venv/bin/python -m pytest`)

| Category | File | Tests | What it checks |
|---|---|---:|---|
| Unit | `test_scoring_unit.py` | 8 | band boundaries; display score aligns with band; **metamorphic** checks (more savings never raises risk; a bigger loan never lowers it); two defaulter guarantors escalate to High |
| Validation | `test_validation.py` | 5 | deployed model beats the baseline (PR-AUC ≫ base rate; ROC ≥ 0.85); bands ordered; score never contradicts the band; API rejects a zero/missing amount |
| Integration | `test_api_integration.py` | 6 | real HTTP + auth; an officer is blocked (403) from recording a recommendation; a manager succeeds |
| Functional | `test_functional_cases.py` | 6 | the five worked scenarios above, asserted automatically, plus a rate-lever sanity test |
| Acceptance | `test_acceptance.py` | 3 | officer proposes → escalates → manager recommends; officer cannot approve own case (403) |
| **Total** | | **28** | **28 passed** |

---

## Note on scope (for the viva)

The monotone guarantee holds for the **constrained** features (savings, loan-to-savings, loan-to-salary, loan
amount, guarantor/community history) across **normal input ranges**. At an *extreme* borrower-savings value the
displayed score can nudge upward, because raising the borrower's own savings also lowers the *guarantor-to-
borrower savings ratio* (an unconstrained feature) — i.e. very rich borrowers make their backers look
proportionally weaker. It's an edge case, not the operating range; constraining that ratio too is a small,
known future fix.
