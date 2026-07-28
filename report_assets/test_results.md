# Test results (real output) — GuarantorLens

Automated test suite in `guarantorLens_mission_capstone_BE/tests/`. Run:
```
cd guarantorLens_mission_capstone_BE
venv/bin/pip install -r requirements-dev.txt
venv/bin/python -m pytest        # 28 passed
```

## Summary by testing category (ALU template §4.3)
| Category | File | Tests | Result |
|---|---|---:|---|
| Unit testing | test_scoring_unit.py | 8 | passed |
| Validation testing | test_validation.py | 5 | passed |
| Integration testing | test_api_integration.py | 6 | passed |
| Functional / system testing | test_functional_cases.py | 6 | passed |
| Acceptance testing | test_acceptance.py | 3 | passed |
| **Total** | | **28** | **28 passed** |

## Full run (verbatim)
```
tests/test_scoring_unit.py::test_band_boundaries PASSED
tests/test_scoring_unit.py::test_display_score_aligns_with_bands PASSED
tests/test_scoring_unit.py::test_display_score_is_monotonic PASSED
tests/test_scoring_unit.py::test_two_defaulter_backers_escalate_to_high PASSED
tests/test_scoring_unit.py::test_no_guarantors_never_escalates PASSED
tests/test_scoring_unit.py::test_more_savings_does_not_raise_risk PASSED
tests/test_scoring_unit.py::test_bigger_loan_does_not_lower_risk PASSED
tests/test_scoring_unit.py::test_assess_returns_expected_shape PASSED
tests/test_validation.py::test_deployed_model_beats_baseline_and_meets_spec PASSED
tests/test_validation.py::test_bands_are_ordered PASSED
tests/test_validation.py::test_display_score_never_contradicts_band PASSED
tests/test_validation.py::test_api_rejects_zero_amount PASSED
tests/test_validation.py::test_api_rejects_missing_amount PASSED
tests/test_api_integration.py::test_health PASSED
tests/test_api_integration.py::test_assess_risk_requires_auth PASSED
tests/test_api_integration.py::test_assess_risk_returns_band_and_score PASSED
tests/test_api_integration.py::test_well_covered_scores_lower_than_thin PASSED
tests/test_api_integration.py::test_officer_cannot_recommend PASSED
tests/test_api_integration.py::test_manager_can_recommend PASSED
tests/test_functional_cases.py::test_case_band_and_flag[well-covered -> Low] PASSED
tests/test_functional_cases.py::test_case_band_and_flag[big loan, higher rate -> Medium] PASSED
tests/test_functional_cases.py::test_case_band_and_flag[over-extended -> High] PASSED
tests/test_functional_cases.py::test_case_band_and_flag[over-committed backer] PASSED
tests/test_functional_cases.py::test_case_band_and_flag[two defaulter backers] PASSED
tests/test_functional_cases.py::test_interest_rate_lever_raises_risk PASSED
tests/test_acceptance.py::test_scenario_assess_escalate_recommend PASSED
tests/test_acceptance.py::test_scenario_officer_cannot_approve_own_case PASSED
tests/test_acceptance.py::test_scenario_manager_sees_escalation_queue PASSED
============================== 28 passed in 0.33s ==============================
```

## What each category proves (for Chapter 4)
- **Unit** — the scoring logic is correct: band boundaries, score-vs-band consistency and monotonicity, flag escalation, and metamorphic sanity (more savings never raises risk; a bigger loan never lowers it).
- **Validation** — the deployed model meets its spec (beats the baseline, ROC ≥ 0.85), bands are ordered, the displayed score never contradicts the band, and the API rejects invalid input.
- **Integration** — the API works end-to-end with authentication, and the officer/manager permission split is enforced (officer → 403).
- **Functional / system** — the five verified assessment scenarios produce the correct band and flag, and a higher interest rate never lowers the band.
- **Acceptance** — the real SACCO workflow runs end-to-end: officer proposes → escalates → manager recommends; officer cannot approve their own case; the manager sees the escalation queue.
