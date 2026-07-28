# GuarantorLens — 5-minute demo cheat sheet

One borrower, driven entirely from the **what-if** panel. All outputs verified against the live model.

## Copy-paste values

| Field | Value |
|---|---|
| Borrower ID | Client97051 |
| Savings | 367800 |
| Salary | 299065 |
| Interest rate | 14 |
| Clean guarantors | Client32045, Client41963 |
| Over-committed guarantor | Client120931 |
| Written-off guarantors | Client107745, Client113217 |

## Step 0 — run ONE assessment, then Save
Borrower `Client97051` · amount `1500000` · savings `367800` · salary `299065` · rate `14` · guarantors `Client32045, Client41963` → **Low · 37**. Open the saved application → scroll to **Simulate a change (what-if)**.

## What-if demo (~3 min) — change one field, press Recalculate

| # | Change in what-if | Guarantors field (paste) | Result | Say |
|---|---|---|---|---|
| 1 | Amount → 9000000 | Client32045, Client41963 | Medium · 56 | Bigger loan, same savings — risk climbs |
| 2 | Amount → 15000000 | Client32045, Client41963 | High · 70 | Push further and it's High; it can never fall as the loan grows (monotone) |
| 3 | Amount → 1500000, Savings → 150000 | Client32045, Client41963 | Medium · 49 | Thinner savings, higher risk |
| 4 | Savings → 367800; swap backers | Client120931, Client32045 | Medium · 48 + over-committed flag | Same borrower and loan, but one backer is over-committed — flagged |
| 5 | Swap backers again | Client107745, Client113217 | High · 70 + "backed by 2 written-off" flag | Two backers who defaulted before push the same 1.5M loan to High — this is the guarantor-network contribution, made visible |

Rows 4–5 are the convincer: **Low → High on the backing alone**, same person, same amount.

## Page tour (~2 min)
| Page | Show | Say |
|---|---|---|
| Member (Client97051 or Client120931) | Guarantee network graph + contagion | If this backer fails, exactly which loans are exposed |
| Insights | Weak-links + communities ranked by write-off rate | 1,366 groups; the riskiest ~27.5% vs 2.1% |
| Monitoring | "How loans age" card | 98% still healthy at 2 years; big loans slip first |
| Role gating | Try to record a recommendation as officer → 403 | Officer proposes, manager decides |

## Guardrails
- Don't push savings to an extreme (e.g. 3,000,000) — a known edge makes risk tick up. Stay in the ranges above.
- Small loans floor around Low 37, so start at 1,500,000 for a clean climb to Medium/High.
- If the app lags, keep talking — don't debug live.
