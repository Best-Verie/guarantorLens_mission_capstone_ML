# Defense answers — GuarantorLens

Draft answers grounded in the codebase and in real numbers computed on the actual cohort. **[YOU]** marks a
point only you can supply (something you observed, decided, or were told) — I've written what to say, but the
facts are yours. Numbers without a mark are ones I verified.

**Keep these in front of you:** 11,015 loans · 226 write-offs (2.1%) · 22,485 members · 11 of ~30 branches ·
18 features (11 borrower + 7 network) · deployed PR-AUC 0.603 / ROC 0.932 · recall 0.801 · precision 0.153 ·
threshold 0.017 · network lift +0.012 [−0.019, +0.044]. New: drop interest_rate → PR 0.660→0.613 ·
complete-records network lift −0.043 · 122 arrears-but-0 (0.54× positives) · contagion 377 vs random 774 ·
guarantee cap max 7 · g_mean_savings 35% fully imputed · riskiest community 27.5%, 95% CI [16%, 43%].

---

## A. The problem and why it is a problem

**1.** A house is a fixed asset the SACCO can value and seize; two teachers are two *people* whose ability to
cover is itself uncertain and *correlated* (same employer, same shocks). So the collateral's value depends on a
network of human risk, which is exactly what a house-backed loan doesn't have — and what no tool at the SACCO
currently makes visible.

**2. [sharp]** Write-off is the SACCO's own definitive record of a loan it decided it will not recover — it is
the outcome the institution acts on and provisions for, and it is unambiguous in the data. Default/NPL states
(arrears) are *transient* and reversible; a loan can be 90+ days late and recover. I chose the outcome the
business actually cares about. **The honest caveat** (own it): write-off is partly an accounting act, so the
model predicts "loans the SACCO ends up writing off", which folds in some institutional behaviour.
- **[follow-up]** Yes — if branches differ in write-off practice, the model partly learns branch policy. I
  mitigate by never using branch as a feature and by borrower-grouped CV, but I did **not** test branch-level
  write-off-rate differences. That's a stated limitation and a fair future check.

**3. [sharp]** Approximate answer from the data: of the 226 write-offs, **182 (≈80%) had no guarantor with a
prior write-off and were within the 8-guarantor cap** — i.e. they looked *clean* on the guarantor rules at
disbursement. So the guarantor rules are **largely not the failure**; write-offs happen despite clean backers,
which means my contribution is on the **borrower-financial** side and the network's role is limited (consistent
with my null result). I could not check "sufficient savings cover" and "performing status" without the exact
thresholds **[YOU: confirm the two thresholds and I'll compute the exact all-four pass rate]**.

**4.** The cap of 8 is **decorative, not binding**: the maximum guarantees any member gives in the data is
**7**, only 5 members reach 7, and 100% are at ≤5 (mean 1.58, median 1). So over-commitment via the cap is not a
live constraint in this cohort — worth stating plainly rather than implying the cap bites.

**5.** Without the decade-apart comparison: SACCOs serve lower-income, thinner-file borrowers with *social*
collateral rather than physical collateral, and Umwalimu SACCO had no data-driven, network-aware, explainable
appraisal tool. The case stands on the **structural gap** (no tool makes the guarantee network visible), not on
a headline NPL number. **[YOU/SOURCE: ideally add one current SACCO NPL figure from BNR/MINECOFIN.]**

**6.** Recovery order **[YOU: confirm against SACCO policy]** — typically: the borrower's own savings are seized
first, then guarantors are called on (their savings, then salary deduction), and the loan is written off only
when recovery from all of these fails. Walk it in that order; it explains why guarantor quality matters.

**7. [sharp] [YOU]** Be honest about your access. If you did not observe a live appraisal, say the process
description comes from the literature plus an informal conversation with IT/operations staff, and that direct
observation of a credit officer is a limitation. Do not claim fieldwork you didn't do — the panel will credit
the honesty.

---

## B. Data, labels, and cohort

**8. [sharp]** **122 loans (1.1%) are 90+ days in arrears but labelled 0** — 0.54× the size of the 226
positives. Some will become write-offs, so the negative class is contaminated with future positives. This
biases every metric **downward** (my reported scores are a conservative lower bound), since the model is
penalised for ranking these "not yet written off but troubled" loans as risky.
- **[follow-up]** I ran the sensitivity: relabelling the 122 arrears-90+ loans as write-offs moves PR-AUC
  **0.653 → 0.520**. Note the direction — it *drops*, because the model does **not** rank these loans as
  high-risk (they looked normal at origination). So they are **not** inflating the headline; the result instead
  shows that loans deteriorating after a clean start are genuinely hard to predict from application-time
  features — an honest limitation, not a hidden bias in my favour.

**9. [YOU]** The merge rule: the two source files (Normal / Written Off) are joined into one row per loan, and
**a loan appearing in both is treated as written-off** (write-off wins on overlap). Confirm the 2025-vs-2026
extract handling — the current pipeline dedupes by Loan ID with write-off precedence.

**10. [sharp]** Real: **35% of loans have `g_mean_savings` fully median-imputed** (no guarantor has any savings
record), and on average only **~36% of a loan's guarantors have a record**. So these features are heavily
imputed — the reviewer is right. **But** (the key point) the network's lift stays ~0 even when restricted to the
**1,049 loans where every guarantor has complete records** (lift −0.043), so imputation is *not* what killed the
signal — homophily is. The imputation weakens the features but is not the reason for the null result.

**11. [YOU]** Say who selected the 11 branches and why (data availability vs deliberate sampling). Then state
honestly whether they're representative on size/members/write-off rate or were the easiest to extract — if you
didn't test representativeness, name it as a limitation.

**12.** ~2 members per loan on average, but a member can borrow and/or guarantee, so 22,485 distinct members
appear across 11,015 loans — most members are in the graph as **guarantors, not borrowers** (that's why 51%
have no loan of their own). The graph is dominated by people whose only footprint is backing others.

**13.** They measure different things: **17,501 = distinct guarantors (people)**; **~25,624/27,559 = guarantee
links (edges)**. Not a contradiction — a labelling slip. Fix the captions to "distinct guarantors" vs
"guarantee links".

---

## C. Features and leakage

**14.** The screen drops any field whose correlation with the disbursement date exceeds 0.40 (a disguised
clock). The hardest call was `time_since_last_loan` (date-correlation 0.42) — genuinely informative but it
tracks calendar time, so keeping it would leak the vintage; I dropped it. 18 features survive.

**15. [sharp]** It is a credit-risk model that *includes* a pricing signal. `interest_rate` is the top feature
and encodes risk-based pricing (13% → 0.5% default, 14% → 13.3%), which correlates with employment type. **But
dropping it entirely costs only 0.047 PR-AUC (0.660 → 0.613)** — the model keeps most of its skill from savings,
loan-to-savings, account age and guarantor/community history. So it is *not merely* an employment classifier.
- **[follow-up]** Drop interest_rate: PR-AUC 0.660 → 0.613 (borrower+network); 0.663 → 0.599 (borrower-only).
- **[follow-up]** No — "your rate is 14%" is not an acceptable member-facing reason. The member-facing
  explanation uses the plain drivers (savings, loan size, guarantor history), and this is exactly why a fairness
  audit on the public/private split is a stated precondition for deployment.

**16. [sharp]** Honest: they are two different community definitions. The **model feature**
`community_prior_default_rate` uses **union-find connected components** (the borrower's whole reachable guarantee
cluster) computed live in the backend. The **Insights view** ranks **Louvain** communities (tight groups),
computed offline in the notebook. So the community *shown* to the officer is **not** the same grouping the
*score* was built from. That's an inconsistency I'd align in future work (use one definition end to end); for
now the feature is a broad "cluster history" signal and the view is a browsing aid.

**17.** The `+1` avoids divide-by-zero for a borrower with (near-)zero savings. For such a borrower the ratio
becomes essentially the guarantors' savings themselves — i.e. "how much backing stands behind someone with
almost no savings of their own", which is exactly the high-risk case we want the feature to capture.

**18.** At serve time the application's date is "now"; all as-of histories are computed up to that point. Since
serving happens after all recorded loans, the as-of window is effectively the full history — consistent with
training, where each loan's window ends at its own disbursement. There's no train-serve skew in *definition*;
the only difference is the reference date, which is intended.

**19. [sharp]** A first-time borrower (b_prior_loans 0, no prior write-off) with two never-borrowed guarantors:
the borrower-history features are 0/empty, the guarantor savings/salary features are missing → median-imputed,
and the community rate is ~0/imputed. The model leans on the loan-vs-savings ratios and interest rate, and
returns a score near the portfolio baseline unless the amount is large relative to savings. It degrades
gracefully to "not much signal → near base rate", which is the honest output.

---

## D. Methodology and evaluation

**20. [sharp]** Borrower-grouped CV controls **identity** leakage (the real risk here, since a borrower recurs).
I did **not** do a strict 2022-train/2023-test **temporal** split, and you're right that deployment is forward
in time. My mitigations were the date-correlation leakage screen (dropping vintage proxies) and a matured
cohort. I accept that grouped folds don't control temporal drift; a time-based split is the correct additional
robustness check and is honest future work. **[RE-RUN: I can run a 2022→2023 split for you.]**

**21.** The bootstrap resamples **over loans** (2,000 draws), recomputing PR-AUC on each resample of the
out-of-fold predictions. It does **not** explicitly model fold-to-fold correlation — a limitation; a
grouped/block bootstrap would be more conservative. The interval crossing zero is the headline regardless.

**22.** The `neg/pos ≈ 48` rule optimises balanced accuracy at a 0.5 threshold, not ranking. Heavy up-weighting
distorts the predicted probabilities and flattens the score distribution, which *lowers* PR-AUC. The recall we
want comes from the operating **threshold**, not from re-weighting — so `scale_pos_weight = 1` ranks best (PR
0.594 → 0.653 across the sweep).

**23. [sharp]** Honest: 80% recall was chosen as a **screening-first** stance (a missed default costs more than
a review), **not** derived from a measured cost ratio or the SACCO's review capacity. That's a limitation — the
threshold should be set with the SACCO's actual capacity.
- **[follow-up]** At that threshold, ~1,100/11,015 loans are flagged to catch 181, precision 0.153 → ~6 false
  alarms per catch. That is a heavy review load and a real adoption risk; officers would lose trust. The right
  fix is to set the threshold from branch capacity, or present a *ranked queue* (top-N by score) rather than a
  binary flag — I'd propose the ranked queue.

**24. [sharp] [YOU]** Say honestly who you consulted before trading 0.653 → 0.603 for monotonicity. If it was
your own judgement (that a believable model an officer trusts beats a slightly sharper black box), say that — it
is a defensible product decision — but don't invent a stakeholder sign-off.

**25.** The still-rising learning curve says the model is **data-starved at 226 positives**, not at its ceiling —
more defaults would help. Roughly, going from 11 to all ~30 branches (≈2.7× the data, ~600 positives) is the
natural next step; I can't promise a specific PR-AUC gain, only that the curve predicts headroom.

**26. [sharp]** I did **not** encode the SACCO's eligibility rules as a classifier — baselines are base rate and
logistic regression, which is a gap. The rules are mostly hard *filters* applied pre-disbursement (so they don't
produce a probability to ROC-compare directly), but I could approximate a rule-score and compare. Honest future
work. Note the related finding from Q3: 182/226 write-offs already *passed* the guarantor rules, so the rules
have limited discriminating power on this cohort.

---

## E. The network finding

**27.** "Adding the guarantor-network features to the borrower features does not improve write-off ranking
beyond noise (lift +0.012, 95% CI [−0.019, +0.044])." It is a **finding**, not a failure, because it is a
*boundary condition* on the corporate-guarantee-network literature: what works for Chinese SME networks does not
transfer to a Rwandan teachers' cooperative, and I show *why* (homophily + a near-tree graph).

**28. [sharp]** The two explanations are homophily vs imputation destroying the signal. The clean test is lift
on **complete-record loans only** — I ran it: on the 1,049 loans where every guarantor has complete savings and
salary (44 defaults), the network lift is **−0.043** (still none). So imputation is **not** the cause;
homophily holds. Caveat: small subset, so noisy — but the direction is clear.

**29. [sharp]** Average degree ≈ **2.3** (2 × ~25,600 edges / 22,485 nodes) — a very sparse, near-tree graph.
At that density Louvain is close to **recovering connected components**, so modularity 0.99 is near-trivial and
I present it as such, not as strong community structure.

**30.** The riskiest community has **40 loans (61 members)** at 27.5% write-off. The 95% Wilson CI is
**[16.1%, 42.8%]** — wide because n is small, but the lower bound (16%) is still ~8× the 2.1% portfolio rate, so
it's a genuinely elevated group, not noise. (The 14 communities at "100%" are 1-loan/4-member artefacts — I
filter to ≥30 loans.)

**31.** Real cascade: 217 defaulters → **377 reached**. A **random seed of 217 members reaches 774 on average
(95% CI 718–833)** — so the real cascade is *below* random, because defaulters sit in less-central positions.
Honest conclusion: the contagion figure illustrates **exposure**, not above-chance systemic spread; I present it
that way rather than as evidence the network amplifies risk.

**32.** Structural differences: corporate SME guarantee networks have **directed, high-value, deliberately
chosen** guarantees among firms with rich financials, dense interlocking structure, and strategic default
behaviour. A teachers' cooperative has **thin-file individuals**, near-tree low-degree structure, guarantees
chosen by social proximity (homophily), and salary-deduction recovery — so the network carries far less
*independent* signal. That's the theoretical spine (joint-liability lending literature: Ghatak & Guinnane;
Besley & Coate).

**33. [sharp]** The value is **not** incremental accuracy — it's **explainability and visibility**: the network
flags (a written-off or over-committed guarantor), the contagion/exposure view a single officer could never
assemble by hand, the fix-it advisor, and a calibrated, monotone, defensible score. In commercial terms: faster,
consistent, auditable appraisals and earlier sight of concentrated exposure — process value, not a higher AUC.

---

## F. The system, deployment, and ethics

**34. [sharp]** It re-scores **real alternative guarantors** from the book, so it's genuine risk reduction when
the swapped-in backer is actually stronger — but you're right it can shade into "make the loan look better to
the model". The monotone constraints and the use of real member data limit gaming, and it's framed as a
suggestion, not an action.
- **[follow-up]** Yes — systematically swapping in strong-savings members concentrates *their* exposure and, at
  the next retraining, the model would learn those members as "safe backers", a feedback loop. That's a real
  risk I'd monitor (cap suggestions per member; watch exposure concentration).

**35.** Yes — that's a genuine fairness concern: penalising members connected to prior write-offs can make it
harder for them to find backers, concentrating rather than reducing risk. It's part of why the fairness audit is
a deployment precondition, and why the tool is decision-support (the officer can override), not an auto-decline.

**36. [sharp]** Corrected facts: the demo runs on the **real, pseudonymised** records, **not synthetic** — the
seeding was a known, deliberate step. It was **explicitly authorised**, and the data was **deleted immediately
after the demonstration**. Member IDs are opaque pseudonyms with no directly identifying fields. I'm fixing the
report to say this in one place and removing the wrong "synthetic only" wording. **[YOU: fill the date and who
authorised.]**
- **[follow-up]** Pseudonymised guarantee-graph data can still be personal data under Law 058/2021 (re-
  identification risk via the graph), which is exactly why the cloud copy was authorised, minimal, and deleted
  immediately — a controlled, time-boxed exception, not indefinite storage.

**37.** Reconcile by fixing §6.4: the approach is *methodologically* sound and the score is *defensible in
design*, but it **should not be deployed until the fairness audit (and the interest-rate proxy check) are done**.
I'm softening "can be trusted" to match §5.3 — the honesty is deliberate.

**38.** The audit: **groups** = public vs private teachers (the interest-rate proxy) and branch; **metric** =
equal-opportunity / TPR gap and FPR gap at the operating threshold (plus calibration-by-group); **threshold** =
refuse deployment if the FPR or TPR gap between groups exceeds a pre-agreed bound (e.g. > ~10 points) or if
calibration diverges materially by group.

**39. [YOU]** Show the exact member-facing wording from the client report/email path (it uses plain drivers —
savings, loan size, guarantor history — and **excludes** guarantor identities and the internal breakdown). State
who delivers it (the loan officer, as guidance, not an automated decline).

**40. [YOU/design]** Retraining: on each new SACCO data extract (e.g. quarterly/annually) by whoever owns the
model (you/IT), via the admin Model tab (upload the new bundle). Drift detection: monitor the calibration curve
and the score distribution / flag rate over time; a shift in observed-vs-predicted default rate signals
recalibration. (This is designed, not yet automated — say so.)

**41.** Honest: the score source is shown (the result carries "source: model" vs "rule-based fallback"), so an
officer *can* see a fallback — but confirm the fallback wording is prominent enough. If it's subtle, that's a
one-line UI fix worth making.

**42. [sharp] [YOU]** Best honest guess: a real credit manager would demand the tool respect the **actual
eligibility rules and branch review capacity** — i.e. a ranked queue sized to what they can review, not a
binary flag that lights up ~1 in 10 loans. Frame it as "I'd expect them to want the flag volume tied to their
capacity."

---

## G. Live demo requests (rehearse; these are actions, not written answers)

**43–50.** These are live actions — rehearse each: (43) assess a fresh loan and read every number; (44) a
Medium/High borderline and what you'd advise; (45) use what-if to push High→Low by swapping guarantors, then say
honestly it changes the *guarantee*, not the borrower's finances, so it's only safer if the new backer is really
stronger; (46) monotone demo — raise savings (score can't rise), raise amount (score can't fall); (47) open a
max-out-degree guarantor (out-degree 7 in your data) and show aggregate exposure; (48) try to record a
recommendation as an officer → 403; (49) remove/rename the bundle and show the rule-based fallback + "source:
rule-based fallback"; (50) open the exported report and show guarantor IDs are redacted to blanks. Practise 45
and 47 most — they're the sharp ones.

---

## H. Closing questions (your reflection — [YOU], but here's a strong frame)

**51.** Weakest part: the **network null result / the interest-rate proxy** — the headline contribution (the
network) adds little, and the strongest feature is partly a pricing proxy. I left it because reporting it
honestly is more valuable than hiding it, and it's a genuine boundary-condition finding.

**52.** With four weeks + all 30 branches: run the **time-based (2022→2023) validation** and the **fairness
audit on public/private**, and add the **incumbent-rules baseline** — the three things that would move this from
"careful prototype" to "deployment-ready evidence".

**53.** If write-offs didn't fall: first check **whether officers actually used the flags** (adoption/logs), then
**whether the threshold matched their capacity** (were they ignoring a 1-in-10 flag), then **calibration drift**.
A model that isn't acted on can't move outcomes — I'd rule out non-use before blaming the model.

**54. [YOU]** Personal reflection — e.g. that reporting a *null* result well is harder and more valuable than
chasing a positive one, or how much of "ML" is actually leakage control and data honesty rather than modelling.
