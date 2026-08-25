# 02 · Shift-Left & Continuous Testing

"Shift-left" means moving testing earlier — catching a bug while a developer
is still typing the change, not two days later in a nightly regression run.
This module wires real, runnable gates into the places that actually catch
problems early: a coverage threshold, a pre-commit hook, and a fast local
subset that runs before every push.

## 1. A coverage gate that actually fails the build

```python
# app.py
def is_even(n):
    return n % 2 == 0

def is_positive(n):
    return n > 0
```

```python
# test_app.py
from app import is_even

def test_is_even():
    assert is_even(4)
    assert not is_even(3)
```

```bash
pip install pytest-cov
pytest --cov=app --cov-report=term-missing --cov-fail-under=80
```

```text
test_app.py .
ERROR: Coverage failure: total of 75 is less than fail-under=80

Name     Stmts   Miss  Cover   Missing
--------------------------------------
app.py       4      1    75%   5
--------------------------------------
TOTAL        4      1    75%
FAIL Required test coverage of 80% not reached. Total coverage: 75.00%
1 passed in 0.21s
```

That's a real, failing run: `is_positive` was never called by any test, and
`--cov-fail-under=80` turned an untested function into a nonzero exit code —
`1 passed` at the bottom is misleading in isolation; the process exit code is
what a CI pipeline actually checks, and pytest-cov made it nonzero even
though the one test that ran, passed. This is "shift-left" in its simplest
form: the missing test is caught the moment someone runs the suite, not
discovered months later when `is_positive` ships with a bug.

## 2. Pre-commit hooks — catching issues before they're even committed

```yaml
# .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: pytest-fast
        name: Run fast unit tests
        entry: pytest -m unit -x
        language: system
        pass_filenames: false
        always_run: true

      - id: ruff
        name: Lint with ruff
        entry: ruff check .
        language: system
        pass_filenames: false
```

```bash
pip install pre-commit
pre-commit install     # wires this into .git/hooks/pre-commit
```

Once installed, every `git commit` runs the fast unit layer (Level 4 Module 1
established that split) and a linter *before* the commit is created. A
developer finds out about a broken unit test in seconds, locally, instead of
minutes later when CI reports it — or worse, after a teammate has already
pulled the broken commit. `-x` (stop on first failure) keeps this fast enough
to not be annoying; the full suite still belongs in CI, not in every commit.

## 3. Fast feedback loops with `--lf` and `--ff`

```bash
pytest --lf     # run only tests that failed last time
pytest --ff     # run previously-failed tests first, then the rest
```

After the coverage-gate failure in section 1, adding a test for
`is_positive` and running `pytest --lf` re-runs only the newly-relevant
failing check rather than the whole suite — during active development this
turns a multi-second full run into a sub-second feedback loop, which is the
entire point of shifting testing left: friction near zero means developers
actually run tests constantly instead of batching them up.

## 4. Testing at PR time vs. testing continuously on `main`

A shift-left pipeline typically runs different depth at different points:

```yaml
# .github/workflows/tests.yml (extends Level 3 Module 4)
on:
  pull_request:
    # fast: unit + integration, every push to a PR branch
  push:
    branches: [main]
    # full: unit + integration + e2e + coverage gate, only on merge to main
```

```yaml
jobs:
  pr-check:
    if: github.event_name == 'pull_request'
    steps:
      - run: pytest tests/unit tests/integration -v

  main-check:
    if: github.event_name == 'push'
    steps:
      - run: pytest -v --cov=app --cov-fail-under=80
      - run: pytest -m e2e -v
```

This gives contributors fast (~seconds to low-minutes) feedback on every push
to a PR, while reserving the expensive full suite — including the coverage
gate from section 1 and any E2E/UI layer — for the point where code is about
to land on `main`.

## 5. Contract checks as an even-earlier gate

Shift-left extends past code into design: catching an API contract break
(Level 4 Module 3 covers this properly) at PR-review time, before either
service's code has even merged, is strictly earlier than catching it in a
staging integration test after both sides deploy.

## 6. Testing-specific traps

**Trap 1 — a pre-commit hook so slow that developers bypass it.** If
`pre-commit install`'s hook takes 90 seconds, developers will reach for
`git commit --no-verify` and the entire shift-left investment evaporates.
Keep pre-commit hooks to the truly fast layer (unit tests, linting) and never
put integration or E2E tests there.

**Trap 2 — coverage percentage as a target instead of a floor.** Chasing
`--cov-fail-under=100` produces tests that exist purely to execute lines
(asserting nothing meaningful) just to satisfy the gate — a common and
genuinely bad outcome of overzealous coverage enforcement. Treat coverage as
a floor that catches *obviously* untested code, not a quality metric on its
own (Level 4 Module 5 goes deeper on what to measure instead).

**Trap 3 — different gates locally versus in CI.** If `pre-commit` enforces
one Python version and CI runs another, "passes locally" and "fails in CI"
diverge in exactly the way shift-left is supposed to prevent. Pin versions in
both places from the same source (a `pyproject.toml` or `.python-version`
file) rather than letting them drift independently.

**Trap 4 — shifting left without also shifting *right*.** Fast local
feedback catches logic bugs; it does not catch what only shows up under real
production load, real data, or real user behavior. Shift-left is
complementary to production monitoring and canary releases, not a
replacement for them — a team that only tests pre-merge and never observes
production is still flying partially blind.

## Cheat sheet

| Goal | Tool / command |
|---|---|
| Fail the build on undertested code | `pytest --cov=app --cov-fail-under=N` |
| Catch issues before commit | `pre-commit install` + `.pre-commit-config.yaml` |
| Fast local iteration | `pytest --lf`, `pytest --ff` |
| Different depth per pipeline stage | fast subset on PR, full suite + gate on `main` |
| Avoid slow hooks getting bypassed | keep pre-commit to unit tests + lint only |
| Avoid coverage-as-vanity-metric | pair with meaningful assertions, not just line execution |
| Complement, don't replace, prod signal | shift-left + production monitoring together |

## Exercise

1. Reproduce the coverage-gate failure in section 1 yourself: write `app.py`
   and a test covering only one function, run
   `pytest --cov=app --cov-fail-under=80 --cov-report=term-missing`, and
   paste your own output showing the missing line number.
2. Add a test for the uncovered function and confirm the same command now
   exits 0 — check with `echo $?` after the run.
3. Set up `.pre-commit-config.yaml` with a fast-unit-tests hook and a linter
   hook, run `pre-commit install`, then make a commit with a deliberately
   failing unit test and confirm the commit is blocked.
4. Time the difference between `pytest` (full suite) and `pytest --lf` after
   fixing one failing test, using `--durations=0` on both runs.
5. Design (as a YAML sketch, not necessarily working CI) a two-tier pipeline
   for a hypothetical project: what runs on every PR push, and what runs only
   on merge to `main` — justify each placement in one sentence per job.
