# 09 · Flaky Test Diagnosis & Stabilization

A flaky test is one that passes and fails on the same code, with no changes
in between — the single most trust-corroding thing a test suite can do.
Once a team learns to ignore a red build because "it's probably just flaky
again," the suite has stopped doing its job. This module reproduces a real
flaky test, diagnoses why, and fixes it properly instead of papering over it.

## 1. Reproducing a real flaky test

```python
# test_flaky.py
import random
import time

def test_race_condition_flaky():
    start = time.time()
    delay = random.choice([0.01, 0.2])   # simulates variable operation time
    time.sleep(delay)
    elapsed = time.time() - start
    assert elapsed < 0.05
```

```text
$ pytest test_flaky.py -q
.                                                                    [100%]
1 passed in 0.17s

$ pytest test_flaky.py -q
F                                                                    [100%]
FAILED test_flaky.py::test_race_condition_flaky - assert 0.210... < 0.05

$ pytest test_flaky.py -q
F                                                                    [100%]
FAILED test_flaky.py::test_race_condition_flaky - assert 0.202... < 0.05
```

Same file, same command, no code changes — three actual runs, one pass and
two fails. This is the textbook signature of flakiness: the assertion is
correct *most of the time* by luck, not because the logic is sound. Real-world
flaky tests hide this same pattern behind more layers — a race condition, an
unmocked clock, network latency, or shared state — but the root cause is
always some form of nondeterminism the test didn't account for.

## 2. The wrong fix: reruns as a permanent solution

```bash
pip install pytest-rerunfailures
pytest test_flaky.py -v --reruns 3
```

```text
test_flaky.py::test_race_condition_flaky PASSED
1 passed in 0.17s
```

`--reruns 3` retries a failing test up to three times before reporting it as
failed, and it genuinely has legitimate uses — masking *unavoidable* external
flakiness (a third-party API's occasional hiccup) in a large suite where
fixing the root cause isn't possible. But reaching for it *first*, on a test
you own and can actually fix, trains the team to stop investigating failures.
Use it as a last resort with a tracked follow-up ticket, never as the default
response to a new flaky test.

## 3. The right fix: remove the nondeterminism

```python
def test_race_condition_fixed():
    # No arbitrary time budget — assert against the actual delay,
    # or better, remove the fake timing dependency entirely.
    delay = random.choice([0.01, 0.2])
    start = time.time()
    time.sleep(delay)
    elapsed = time.time() - start
    assert elapsed >= delay   # a property that's true regardless of scheduling jitter
```

```text
$ pytest test_flaky.py::test_race_condition_fixed -v
test_flaky.py::test_race_condition_fixed PASSED
```

The fix here is the same idea from Module 5's property-based testing: don't
assert an arbitrary threshold you hoped would hold; assert something that's
actually always true. In a real codebase, the better fix is usually to
eliminate the wall-clock dependency altogether — inject a fake clock, or
assert on an event/callback rather than elapsed time.

## 4. Common root causes and their real fixes

**Unmocked time.** A test that calls `time.sleep()` and asserts an elapsed
duration, or that compares `datetime.now()` across two calls, is flaky by
construction on a loaded CI runner. Fix: inject a fake clock (`freezegun`,
Level 2's mocking patterns) instead of measuring real time.

**Test order dependency.** A test that passes when run alone but fails inside
the full suite usually depends on state left behind by an earlier test — a
module-level variable, a shared database row, a `sys.path` mutation. Fix:
run with `pytest-randomly` (randomizes test order every run) specifically to
*surface* this class of bug, then isolate the shared state into a
function-scoped fixture.

```bash
pip install pytest-randomly
pytest -p randomly --randomly-seed=12345   # rerun with the same shuffled order to reproduce
```

**Unmocked network calls.** A test hitting a real external API is at the
mercy of that API's uptime and latency — this is precisely what Level 2
Module 4's mocking techniques (`unittest.mock`, `responses`,
`pytest-httpserver`) exist to eliminate for anything that isn't explicitly an
integration test.

**Async/threading races.** A test asserting on a background thread's side
effect immediately after starting it (`thread.start(); assert x == 1`) races
the thread's actual completion. Fix: join the thread or use a proper
synchronization primitive (`Event`, `Queue`) — never a `time.sleep()` "wait
long enough" hack, which just moves the flakiness threshold rather than
removing it.

## 5. Finding flaky tests before they erode trust

```bash
# Run the whole suite N times and look for tests that don't always agree with themselves
pytest --count=20 test_flaky.py  # requires pytest-repeat
```

```bash
pip install pytest-repeat
pytest --count=20 -q test_flaky.py
```

Running a suspect test dozens of times in a loop is the most direct way to
confirm flakiness before spending time on root-causing it — a test that fails
1 time in 20 local runs will absolutely fail regularly across hundreds of CI
runs a month.

## 6. Testing-specific traps

**Trap 1 — quarantining a flaky test and forgetting about it.**
`@pytest.mark.skip(reason="flaky, investigating")` with no ticket and no
follow-up is how coverage silently erodes over months — six months later
nobody remembers what the test was even checking. Always pair a skip with a
tracked issue and a re-enable date.

**Trap 2 — "fixing" flakiness by loosening the assertion.** Changing
`assert elapsed < 0.05` to `assert elapsed < 5.0` makes the test stop
failing, but it also makes the test stop testing anything meaningful — a
regression that makes the operation take 3 seconds instead of 0.05 now
passes silently. A flaky assertion and a useless assertion are both bad; make
sure the fix produces a *useful* deterministic assertion, not just a
non-flaky one.

**Trap 3 — blaming CI infrastructure before checking the test.** It's
tempting to assume "the runner is just slow today" when a test fails once in
CI and passes on rerun. Section 5's repeat-run technique costs a few minutes
and usually reveals whether the problem is genuinely environmental or sitting
in the test's own logic.

**Trap 4 — flakiness introduced by parallel execution (Level 2 Module 8).**
A test that was reliable running serially can become flaky under
`pytest-xdist` if it shares a file, port, or database row with a test running
concurrently in another worker. If flakiness only appears with `-n auto`, the
bug is almost certainly shared mutable state, not timing.

## Cheat sheet

| Symptom | Likely cause | Real fix |
|---|---|---|
| Fails ~sometimes, no code change | timing/threshold assertion | assert an invariant, not a magic number |
| Passes alone, fails in full suite | test order / shared state | function-scoped fixtures; `pytest-randomly` to surface it |
| Fails only in CI, not locally | unmocked network / real external dependency | mock it (Level 2 Module 4) unless it's an intentional integration test |
| Fails only under `-n auto` | shared mutable state across workers | isolate per-test fixtures, unique ports/files/table rows |
| Async assertion right after `.start()` | race condition | join the thread / await properly, never `sleep()`-and-hope |
| "It's flaky, just rerun it" as policy | root cause never investigated | `--reruns` only as a last resort, with a tracked ticket |

## Exercise

1. Reproduce the flaky test in section 1 yourself, run it five times in a
   row, and record the actual pass/fail sequence you got — flakiness rates
   vary by machine load, so expect a different sequence than the one shown
   here.
2. Fix it using the invariant-based approach from section 3, then run it 20
   times with `pytest-repeat`'s `--count=20` and confirm zero failures.
3. Write a test with genuine test-order dependency (a module-level list one
   test appends to and another test asserts the length of), run it normally
   (passes), then run it with `pytest-randomly` and show it fail once order
   is shuffled.
4. Fix the order-dependent test by moving the shared list into a
   function-scoped fixture, and confirm it now passes under `pytest-randomly`
   regardless of seed.
5. Write one paragraph on when `--reruns` is a legitimate tool versus when
   it's covering up technical debt, using a concrete example from your own
   experience or coursework.
