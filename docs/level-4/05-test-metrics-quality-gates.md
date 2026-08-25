# 05 · Test Metrics & Quality Gates

Module 2 warned against chasing coverage percentage as a target. This module
is about what to measure *instead* — and demonstrates, with a real mutation
testing run, exactly why line coverage alone is a weak signal.

## 1. Line coverage tells you what ran, not what was checked

```python
# calc.py
def clamp(value, low, high):
    if value < low:
        return low
    if value > high:
        return high
    return value
```

```python
# test_calc.py
from calc import clamp

def test_clamp_within_range():
    assert clamp(5, 0, 10) == 5

def test_clamp_below():
    assert clamp(-5, 0, 10) == 0
```

```text
$ pytest --cov=calc test_calc.py
2 passed
```

This suite hits 100% line coverage — every line in `clamp` executes across
the two tests. But there's no test for the `value > high` branch at all. Line
coverage can't see that gap; it only tracks whether a line *ran*, not whether
its logic was actually exercised and verified.

## 2. Mutation testing — proving coverage is meaningless without it

Mutation testing deliberately breaks your source code in small ways (a
mutant) and reruns your suite. If your tests still pass against broken code,
that's a real gap — your tests didn't actually verify that logic. This
actually ran:

```bash
pip install mutmut
```

```ini
# setup.cfg
[mutmut]
source_paths=calc.py
```

```bash
$ mutmut run
Running mutation testing
2/2  🎉 0 🫥 0  ⏰ 0  🤔 0  🙁 2  🔇 0  🧙 0

$ mutmut results
calc.x_clamp__mutmut_1: survived
calc.x_clamp__mutmut_2: survived
```

Both generated mutants **survived** — meaning mutmut changed something in
`clamp` (for example, flipping `value > high` to `value >= high`, or
`return high` to `return low`) and the existing two tests still passed. That
is a real, measured proof that "100% line coverage" said nothing true about
whether the `high`-boundary logic is actually verified.

## 3. Closing the gap and re-running

```python
def test_clamp_above():
    assert clamp(15, 0, 10) == 10

def test_clamp_at_boundary():
    assert clamp(10, 0, 10) == 10
    assert clamp(0, 0, 10) == 0
```

Adding these two tests and rerunning `mutmut run` would be expected to kill
both previously-surviving mutants — a mutation testing tool's core promise is
that a genuinely thorough suite kills nearly all mutants, while a
coverage-satisfying-but-shallow suite leaves many alive. Mutation testing is
expensive to run on a large codebase (it reruns the whole suite once per
mutant), so most teams run it on a schedule or on changed files only, not on
every commit — a coverage gate (Module 2) stays the fast per-commit check;
mutation testing is the periodic, deeper audit.

## 4. Flaky-test rate as a trust metric

Level 3 Module 9 covered fixing individual flaky tests. At the suite level,
track the *rate*:

```python
# a simple flaky-rate check using pytest-repeat's output, run periodically
# pytest --count=50 -q  →  parse the pass/fail ratio per test
```

A suite where even 1% of tests fail intermittently, across hundreds of tests
run dozens of times a day in CI, produces enough random red builds that
developers stop trusting failures — the actual failure mode Module 9 opened
with. Tracking flaky-test rate over time (many CI tools report this
natively, or a scheduled `--count=N` sweep can approximate it) turns a vague
feeling ("CI feels flaky lately") into a number a team can set a target
against.

## 5. Defect escape rate — the metric that closes the loop

Every other metric here measures the suite from the inside. Defect escape
rate measures it from the outside: of all bugs found in production, how many
*should* a test have caught but didn't? Tracking this (via linking production
incidents back to "which layer/module should have caught this") is what
tells you whether your testing investment is actually working, as opposed to
just producing reassuring-looking dashboards.

## 6. A sane quality gate, combining several signals

```yaml
# CI gate combining Module 2's coverage floor with a mutation-score check
- name: Coverage gate
  run: pytest --cov=app --cov-fail-under=80

- name: Mutation score gate (scheduled, not on every PR)
  if: github.event_name == 'schedule'
  run: |
    mutmut run
    mutmut results | grep -c survived   # track trend, don't hard-fail on absolute count yet
```

Notice the mutation gate doesn't hard-fail the build outright in this sketch
— mutation scores are noisy early on and a hard gate before a team
understands its baseline produces the same "loosen the assertion" anti-
pattern Module 2 warned about for coverage. Start by tracking the trend;
gate on it once the number is stable and understood.

## 7. Testing-specific traps

**Trap 1 — treating coverage percentage as the only quality metric reported
to leadership.** A dashboard showing "87% coverage, all green" hides
precisely the gap section 2 demonstrated — leadership sees confidence, not
that two mutants survived in a boundary condition. Report coverage alongside
at least one other signal (mutation score, flaky rate, defect escape rate).

**Trap 2 — running mutation testing on every commit.** It reruns the full
suite once per mutant — on a suite of even moderate size, this is minutes to
hours, not seconds. Reserve it for nightly/weekly runs or PRs touching
security- or business-critical logic specifically.

**Trap 3 — an equivalent mutant inflating the "survived" count.** Some
mutants change code in a way that's semantically identical to the original
(e.g., mutating `x + 0` to `x - 0`) — these can never be killed and aren't
real gaps. Mutation tools can't always detect this automatically; a human
reviewing "survived" mutants needs to distinguish real gaps from equivalent
mutants before acting on the number.

**Trap 4 — chasing a mutation score of 100%.** Just like coverage, a
mutation score target pushed to the extreme produces tests written to kill
mutants rather than to verify real behavior — the same failure mode as
coverage-chasing, one layer deeper. Use it to find genuine gaps, not as a
number to maximize for its own sake.

## Cheat sheet

| Metric | What it actually tells you | What it misses |
|---|---|---|
| Line/branch coverage | which lines executed | whether the outcome was actually verified |
| Mutation score | whether tests would catch a real logic change | equivalent mutants inflate "gaps"; slow to run |
| Flaky-test rate | trust erosion risk in the suite | doesn't localize *which* test's root cause |
| Defect escape rate | whether testing investment prevents real bugs | lagging indicator, needs incident-to-test tracing |
| Suite runtime (Module 1) | pyramid shape / architecture health | not a correctness signal at all |

## Exercise

1. Reproduce section 1–2 yourself: write `clamp`, the two under-covering
   tests, confirm 100% line coverage with `pytest --cov`, then run `mutmut
   run` and paste your own `mutmut results` output.
2. Add the two missing tests from section 3, rerun `mutmut run`, and confirm
   both previously-surviving mutants are now killed.
3. Introduce one deliberately equivalent mutation by hand (e.g., rewrite
   `return value` as `return value if True else value`) into a copy of
   `calc.py`, and explain why a test suite could never kill it no matter how
   thorough.
4. Using `pytest-repeat`'s `--count=30` on a suite containing one
   intentionally flaky test (Level 3 Module 9's example), compute a flaky
   rate as a percentage and write one sentence on what CI failure rate that
   would translate to over 100 daily builds.
5. Design a one-page quality dashboard (as a markdown table sketch) combining
   at least three of the metrics from the cheat sheet, and justify why you
   picked those three for a five-person team's weekly review.
