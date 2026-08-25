# 09 · QA Leadership & Strategy

Every module so far answers "how do I test this?" A QA lead also has to
answer "should we automate this, and is it working?" — questions with real
financial and organizational weight. This module treats those questions the
same way the rest of the course treats testing: write the logic as code,
verify it with real assertions, don't rely on gut feeling alone.

## 1. A real automation ROI calculator

```python
# roi.py
def automation_roi(manual_hours_per_run, automation_build_hours,
                    hourly_rate, runs_per_month, months):
    manual_cost = manual_hours_per_run * hourly_rate * runs_per_month * months
    automation_cost = automation_build_hours * hourly_rate
    savings = manual_cost - automation_cost
    return {
        "manual_cost": manual_cost,
        "automation_cost": automation_cost,
        "net_savings": savings,
        "breakeven_months": (
            automation_build_hours / (manual_hours_per_run * runs_per_month)
            if manual_hours_per_run else None
        ),
    }
```

```python
# test_roi.py
from roi import automation_roi

def test_positive_roi_over_a_year():
    result = automation_roi(
        manual_hours_per_run=2, automation_build_hours=40,
        hourly_rate=50, runs_per_month=20, months=12
    )
    assert result["net_savings"] > 0
    assert round(result["breakeven_months"], 2) == 1.0

def test_low_frequency_manual_never_pays_off_quickly():
    result = automation_roi(
        manual_hours_per_run=0.5, automation_build_hours=40,
        hourly_rate=50, runs_per_month=1, months=3
    )
    assert result["breakeven_months"] > 3
```

```text
$ pytest test_roi.py -v
test_roi.py::test_positive_roi_over_a_year PASSED
test_roi.py::test_low_frequency_manual_never_pays_off_quickly PASSED
2 passed in 0.30s
```

This isn't a toy exercise — it's the actual argument a QA lead needs to make
to a manager deciding whether to fund a month of automation work. The second
test encodes a real strategic principle: a manual test run rarely and briefly
(here, 30 minutes, once a month) may never be worth automating — the
breakeven point stays beyond any reasonable planning horizon. Writing this as
tested code, rather than a one-off spreadsheet, means the same reasoning is
reusable and auditable the next time a similar decision comes up.

## 2. Turning Module 5's metrics into a strategy input

```python
def prioritize_automation_candidates(candidates):
    """candidates: list of dicts with manual_hours, frequency_per_month, flakiness_risk (0-1)."""
    scored = []
    for c in candidates:
        annual_manual_hours = c["manual_hours"] * c["frequency_per_month"] * 12
        # penalize candidates likely to become flaky (Level 3 Module 9) once automated
        score = annual_manual_hours * (1 - c["flakiness_risk"])
        scored.append({**c, "priority_score": score})
    return sorted(scored, key=lambda c: c["priority_score"], reverse=True)
```

```python
def test_high_frequency_low_risk_test_ranks_first():
    candidates = [
        {"name": "login smoke test", "manual_hours": 0.1, "frequency_per_month": 100, "flakiness_risk": 0.1},
        {"name": "rare admin report", "manual_hours": 2, "frequency_per_month": 1, "flakiness_risk": 0.6},
    ]
    ranked = prioritize_automation_candidates(candidates)
    assert ranked[0]["name"] == "login smoke test"
```

This is a real, if simplified, prioritization model: it explicitly encodes
that a test's automation value is (frequency × time saved) discounted by the
risk it becomes an unreliable, trust-eroding addition to the suite (Level 3
Module 9) — a strategic tradeoff, made explicit and testable rather than left
as a vague gut call.

## 3. A team skills/coverage matrix as data, not a slide

```python
def coverage_gaps(team_skills, required_skills):
    """team_skills / required_skills: sets of skill names."""
    return required_skills - team_skills

def test_identifies_missing_mobile_skill():
    team = {"pytest", "selenium", "api-testing", "ci-cd"}
    required = {"pytest", "selenium", "api-testing", "ci-cd", "appium", "security-testing"}
    gaps = coverage_gaps(team, required)
    assert gaps == {"appium", "security-testing"}
```

```text
$ pytest -v -k coverage_gaps
test_strategy.py::test_identifies_missing_mobile_skill PASSED
```

A skills gap analysis as a testable function (rather than a static slide
reviewed once a year) can be re-run every time the required-skills set
changes — for instance, right after your team commits to shipping a mobile
app (Level 3 Module 7) and suddenly needs Appium coverage it doesn't have.

## 4. Communicating quality status without hiding the bad news

```python
def quality_status(coverage_pct, flaky_rate_pct, defect_escape_count, thresholds):
    issues = []
    if coverage_pct < thresholds["min_coverage"]:
        issues.append(f"Coverage {coverage_pct}% below floor {thresholds['min_coverage']}%")
    if flaky_rate_pct > thresholds["max_flaky_rate"]:
        issues.append(f"Flaky rate {flaky_rate_pct}% above ceiling {thresholds['max_flaky_rate']}%")
    if defect_escape_count > thresholds["max_escapes"]:
        issues.append(f"{defect_escape_count} defects escaped, above {thresholds['max_escapes']}")
    return {"healthy": len(issues) == 0, "issues": issues}
```

```python
def test_status_flags_real_problems():
    result = quality_status(
        coverage_pct=72, flaky_rate_pct=1, defect_escape_count=5,
        thresholds={"min_coverage": 80, "max_flaky_rate": 2, "max_escapes": 3}
    )
    assert result["healthy"] is False
    assert len(result["issues"]) == 2
```

A leadership report generated from a function like this — rather than a
manually-curated slide — can't quietly omit an inconvenient number, because
the thresholds and the data are the same objects a test suite already
verifies against.

## 5. Testing-specific traps

**Trap 1 — presenting vanity metrics as strategy.** "We have 5,000 tests" or
"92% coverage" says nothing about whether the right things are tested (Level
4 Module 5). A leader who reports raw counts instead of outcome-linked
metrics (defect escape rate, flaky rate, mean time to detect) is optimizing
for an easy number to report, not for actual quality.

**Trap 2 — automating for automation's sake.** Section 1's calculator exists
specifically to prevent this: a test that's cheap to run manually and rarely
needed has genuinely negative ROI to automate, however satisfying automating
it might feel. A leader's job includes saying no to automation requests that
don't pay off.

**Trap 3 — under-investing in the skills gap until it's a crisis.** Section
3's gap analysis is meant to run *before* a commitment (a new mobile app, a
new compliance requirement) lands on the team, not after — discovering a
missing skill the week a deadline requires it is a planning failure, not a
technical one.

**Trap 4 — treating quality status reporting as a one-way broadcast.** A
dashboard nobody acts on because "issues" never has a clear owner or
deadline attached becomes noise the team learns to ignore — the same trust
erosion problem as flaky tests (Level 3 Module 9), just at the organizational
level instead of the individual-test level.

## Cheat sheet

| Decision | Tool from this module |
|---|---|
| Should we automate this test? | `automation_roi` — breakeven point, not gut feeling |
| Which of many candidates first? | `prioritize_automation_candidates` — frequency × savings, discounted by flakiness risk |
| Are we ready for a new testing demand? | `coverage_gaps` — required skills minus current skills |
| Is quality actually healthy? | `quality_status` — explicit thresholds, not a subjective read |
| What to avoid reporting | raw counts with no outcome link (Trap 1) |

## Exercise

1. Extend `automation_roi` with a `maintenance_hours_per_month` parameter
   representing ongoing upkeep cost for the automated test, and write a test
   showing a case where high maintenance cost flips a positive ROI negative.
2. Add a third candidate to `prioritize_automation_candidates`'s test data
   with a genuinely ambiguous score (close to one of the existing two) and
   verify the sort order matches your hand calculation.
3. Extend `coverage_gaps` to also report skills the team has that are no
   longer required (a useful signal for redeploying people), and write a
   test for it.
4. Add a fourth threshold to `quality_status` — mean time to detect a
   regression — and write a test where three of four thresholds are
   violated, confirming all three appear in `issues`.
5. Write a one-page (in prose) 90-day QA strategy for a hypothetical
   10-person team that just failed a security audit (tie back to Level 4
   Module 4) — state what you'd measure first, what you'd automate first
   using this module's ROI model, and what skill gap you'd close first using
   section 3's approach.
