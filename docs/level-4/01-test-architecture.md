# 01 · Test Architecture at Scale

Every technique in Levels 1–3 answers "how do I write this test?" This
module answers a different question: once a project has thousands of tests
across unit, API, UI, and performance layers, how do you organize them so the
suite stays fast, reliable, and trustworthy instead of becoming a two-hour
liability nobody wants to run?

## 1. The test pyramid, in actual code

```python
# unit layer — pure logic, no I/O, milliseconds
def calc_total(items):
    return sum(i["price"] * i["qty"] for i in items)

import pytest

@pytest.mark.unit
def test_calc_total():
    assert calc_total([{"price": 10, "qty": 2}, {"price": 5, "qty": 1}]) == 25

# service/integration layer — components wired together, still no network
class OrderService:
    def __init__(self):
        self.orders = {}
    def place_order(self, order_id, items):
        total = calc_total(items)
        self.orders[order_id] = total
        return total

@pytest.mark.integration
def test_place_order():
    svc = OrderService()
    total = svc.place_order("o1", [{"price": 10, "qty": 3}])
    assert svc.orders["o1"] == 30
    assert total == 30
```

```text
$ pytest test_pyramid_demo.py -v -m unit
test_pyramid_demo.py::test_calc_total PASSED
1 passed, 1 deselected

$ pytest test_pyramid_demo.py -v
test_pyramid_demo.py::test_calc_total PASSED
test_pyramid_demo.py::test_place_order PASSED
2 passed
```

Note the actual warning this run produced:

```text
PytestUnknownMarkWarning: Unknown pytest.mark.integration - is this a typo?
```

That's `pytest` telling you something real: a marker used without being
registered in `pytest.ini`/`pyproject.toml` is indistinguishable from a typo
at scale. Section 3 fixes this properly — it's exactly the kind of small gap
that turns into a silently-broken filter (`-m intgration` matching nothing,
with no error) once a suite has hundreds of contributors.

## 2. Why the pyramid shape, not a snapshot of what to test

The pyramid isn't about ratios for their own sake — it's about **cost and
feedback speed**. A unit test runs in milliseconds and pinpoints the exact
function that broke. A UI test (Level 3 Module 1) might take seconds, needs a
browser, and a failure could mean the UI broke, the API broke, the network
was slow, or the test itself is flaky (Level 3 Module 9) — much more to
untangle. A healthy suite has far more of the cheap, precise layer and far
fewer of the expensive, ambiguous one, because that's the only shape that lets
you run "the suite" in minutes instead of hours.

## 3. Registering markers and separating suites by directory

```ini
# pytest.ini
[pytest]
markers =
    unit: fast, no I/O
    integration: multiple components, still in-process
    e2e: full stack, real browser or real network
testpaths = tests
```

```text
tests/
├── unit/
│   └── test_calc_total.py
├── integration/
│   └── test_order_service.py
└── e2e/
    └── test_checkout_flow.py
```

Directory separation plus registered markers gives you two independent ways
to select a slice of the suite — `pytest tests/unit` for the fastest possible
signal, `pytest -m e2e` for a pre-release gate — and re-running
`pytest -v` after registering the marker produces zero warnings, confirming
the fix.

## 4. Shared fixtures without shared coupling

A large suite's biggest architectural risk is a `conftest.py` that every test
file depends on, making the whole suite fragile to one change. The fix is
scoping `conftest.py` files to match the directory structure:

```text
tests/
├── conftest.py              # truly universal fixtures only (e.g. a temp dir)
├── unit/
│   └── conftest.py          # unit-only fixtures
├── integration/
│   └── conftest.py          # a test database, an in-memory service registry
└── e2e/
    └── conftest.py          # a browser fixture (Level 3 Module 1), base_url
```

pytest resolves fixtures by walking up the directory tree, so an `e2e/`
test never even sees `unit/conftest.py`'s fixtures, and a change scoped to
one layer's fixtures can't accidentally break another layer's tests.

## 5. Test doubles as an architectural boundary, not just a mocking trick

Level 2 Module 4 covered `unittest.mock` mechanically. At scale, the more
important decision is *where* the boundary between real and faked
dependencies sits, and keeping it consistent:

```python
# A clear seam: OrderService depends on an abstract PaymentGateway,
# not directly on a specific SDK.
class PaymentGateway:
    def charge(self, amount): raise NotImplementedError

class StripeGateway(PaymentGateway):
    def charge(self, amount):
        ...  # real network call

class FakePaymentGateway(PaymentGateway):
    def __init__(self):
        self.charges = []
    def charge(self, amount):
        self.charges.append(amount)
        return {"status": "success"}
```

Unit and integration tests inject `FakePaymentGateway`; only a small,
dedicated set of contract/e2e tests (Level 4 Module 3 covers this in depth)
ever touches `StripeGateway`. This is the architectural payoff of dependency
injection: the *test* decides which implementation runs, and the production
code never has an `if TESTING:` branch anywhere in it.

## 6. Testing-specific traps

**Trap 1 — an inverted pyramid.** A suite with hundreds of slow E2E tests and
almost no unit tests looks thorough but is expensive and slow to run, and a
single failure is hard to localize. If your CI run takes 40 minutes and most
of that is UI tests, that's an architecture problem, not a "buy faster CI
runners" problem.

**Trap 2 — global `conftest.py` bloat.** Every fixture that "might be useful
elsewhere" ending up in the root `conftest.py` eventually makes every test
file implicitly coupled to every other layer's setup — a change to an e2e
fixture can break a unit test's collection phase entirely. Push fixtures down
to the narrowest `conftest.py` they're actually needed in.

**Trap 3 — unregistered or inconsistent markers across teams.** As seen in
section 1's actual warning, an unregistered marker silently fails to alert
you to typos. At scale, `pytest --strict-markers` turns that warning into a
hard collection error — worth enabling once markers are standardized, so a
new contributor's typo is caught immediately instead of silently excluding
tests from a CI filter.

**Trap 4 — treating "more tests" as strictly better.** Two tests that assert
the same thing at different layers (a unit test and an E2E test both checking
"total is calculated correctly") add maintenance cost without added
confidence. Each layer should test what only *that* layer can catch: unit
tests for logic, integration tests for component wiring, E2E tests for real
user flows across the whole stack.

## Cheat sheet

| Layer | Tests | Speed | Failure tells you |
|---|---|---|---|
| Unit | pure functions, no I/O | ms | exactly which function broke |
| Integration | components wired together, in-process | ~seconds | wiring/contract mismatch between components |
| E2E | full stack, real browser/network | seconds–minutes | something in the whole user flow broke (needs more digging) |
| Enforce the split | `pytest.ini` markers + `testpaths` | — | `-m unit` / `-m e2e` selects a layer |
| Keep suites decoupled | directory-scoped `conftest.py` | — | a layer's fixture change can't break another layer |
| Catch marker typos | `--strict-markers` | — | unregistered markers become collection errors, not warnings |

## Exercise

1. Reorganize a small project's tests into `tests/unit`, `tests/integration`,
   `tests/e2e` directories, each with its own `conftest.py`, and register all
   three markers in `pytest.ini`.
2. Run `pytest --strict-markers` with one deliberately misspelled marker and
   paste the exact collection error it produces (versus the warning shown
   in section 1 without `--strict-markers`).
3. Introduce a `PaymentGateway` abstract base and a `FakePaymentGateway` as in
   section 5, and write one unit test and one integration test that both use
   the fake — confirm neither test makes a real network call.
4. Time your full suite with `pytest --durations=10` and identify which
   layer dominates total runtime; propose (in a comment) one test you'd move
   down a layer to speed up the suite without losing coverage.
5. Write a short (150-word) architecture note explaining, for a team of five
   new contributors, which `conftest.py` a new fixture should go in and why —
   use the directory structure from section 4 as your reference.
