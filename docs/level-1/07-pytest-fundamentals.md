# 07 · pytest Fundamentals

pytest is the standard Python test framework. Its appeal is that a test is just
a function with a plain `assert` — no class hierarchy, no `assertEqual`
ceremony — while the failure output is more informative than either.

## 1. Your first test

Create `tests/test_calculator.py`:

```python
def add(a, b):
    return a + b


def test_add_two_positive_numbers():
    assert add(2, 3) == 5
```

Run it:

```bash
pytest tests/test_calculator.py
```

```text
============================= test session starts ==============================
collected 1 item

tests/test_calculator.py::test_add_two_positive_numbers PASSED           [100%]

============================== 1 passed in 0.01s ===============================
```

That's the whole contract: a function whose name starts with `test_`, containing
an `assert`.

## 2. Discovery rules

pytest finds tests by convention. Get these wrong and your test silently doesn't
run — which is worse than failing.

| Rule | Default |
|---|---|
| Files | `test_*.py` or `*_test.py` |
| Functions | `test_*` |
| Classes | `Test*` (and they must have **no** `__init__` method) |
| Methods in those classes | `test_*` |
| Directories searched | `testpaths` in `pytest.ini`, else the current directory, recursively |

```python
class TestShoppingCart:
    """Grouping related tests in a class is optional but keeps output readable."""

    def test_new_cart_is_empty(self):
        cart = []
        assert len(cart) == 0

    def test_adding_item_increases_count(self):
        cart = []
        cart.append({"sku": "SKU-101", "qty": 1})
        assert len(cart) == 1
```

```text
tests/test_cart.py::TestShoppingCart::test_new_cart_is_empty PASSED       [ 50%]
tests/test_cart.py::TestShoppingCart::test_adding_item_increases_count PASSED [100%]
```

!!! warning "The silent-skip trap"
    A file named `login_tests.py` or a function named `check_login` is collected
    by nothing and reported by nothing. A suite that reports `47 passed` when
    you wrote 52 tests is a suite lying to you. Always check the collected
    count: `pytest --collect-only -q`.

## 3. Assertions

pytest rewrites plain `assert` statements to show you the actual values on
failure. This is its best feature.

```python
def test_order_total_is_calculated_correctly():
    subtotal = 1800
    discount = 360
    shipping = 30
    assert subtotal - discount + shipping == 1470
```

```text
============================== 1 passed in 0.01s ===============================
```

Now break it deliberately — change the expected value to `1500`:

```text
=================================== FAILURES ===================================
___________________ test_order_total_is_calculated_correctly ___________________

    def test_order_total_is_calculated_correctly():
        subtotal = 1800
        discount = 360
        shipping = 30
>       assert subtotal - discount + shipping == 1500
E       assert 1470 == 1500
E        +  where 1470 = ((1800 - 360) + 30)

tests/test_order.py:5: AssertionError
=========================== short test summary info ============================
FAILED tests/test_order.py::test_order_total_is_calculated_correctly - assert 1470 == 1500
============================== 1 failed in 0.02s ===============================
```

It shows the computed value *and* the sub-expression that produced it. No
`print` statements required.

### Assertion patterns

```python
def test_assertion_styles():
    # equality
    assert "SAVE20".lower() == "save20"

    # membership
    assert "shoes" in "trail running shoes"
    assert 404 not in [200, 201, 204]

    # truthiness — prefer explicit comparisons in tests
    assert [1, 2, 3]          # non-empty list is truthy
    assert not []             # empty list is falsy

    # identity vs equality
    value = None
    assert value is None      # correct for None
    assert [1, 2] == [1, 2]   # correct for values

    # collections
    assert {"a": 1, "b": 2}["a"] == 1
    assert sorted([3, 1, 2]) == [1, 2, 3]
    assert set(["a", "b"]) == set(["b", "a"])

    # floats — never use == on floats
    from pytest import approx
    assert 0.1 + 0.2 == approx(0.3)
```

```text
tests/test_assertions.py::test_assertion_styles PASSED                   [100%]
```

`0.1 + 0.2 == 0.3` is `False` in floating-point arithmetic. `approx` is what
you use for any monetary or measured value.

### Assertion messages

Add a message when the failure wouldn't be self-explanatory:

```python
def test_response_time_within_sla():
    response_time_ms = 2400
    assert response_time_ms < 2000, (
        f"Search exceeded the 2000 ms SLA: took {response_time_ms} ms"
    )
```

```text
E       AssertionError: Search exceeded the 2000 ms SLA: took 2400 ms
E       assert 2400 < 2000
```

!!! tip "One logical assertion per test"
    A test with eight unrelated asserts stops at the first failure and hides the
    other seven results. Multiple asserts that *together* verify one behaviour
    (status code + body + header of one response) are fine — that's one logical
    check.

## 4. Testing for expected exceptions

Negative testing in code form. `pytest.raises` asserts that something *does*
fail, and fails the test if it doesn't.

```python
import pytest


def apply_promo(order_total, code):
    if order_total < 500:
        raise ValueError("Promo codes require a minimum order of 500")
    if code != "SAVE20":
        raise KeyError(f"Unknown promo code: {code}")
    return order_total * 0.8


def test_promo_rejected_below_minimum_order():
    with pytest.raises(ValueError) as exc_info:
        apply_promo(499, "SAVE20")
    assert "minimum order of 500" in str(exc_info.value)


def test_unknown_promo_code_raises_keyerror():
    with pytest.raises(KeyError, match="Unknown promo code"):
        apply_promo(1000, "FAKECODE")


def test_valid_promo_applies_discount():
    assert apply_promo(1000, "SAVE20") == pytest.approx(800.0)
```

```text
tests/test_promo.py::test_promo_rejected_below_minimum_order PASSED       [ 33%]
tests/test_promo.py::test_unknown_promo_code_raises_keyerror PASSED       [ 66%]
tests/test_promo.py::test_valid_promo_applies_discount PASSED             [100%]

============================== 3 passed in 0.02s ===============================
```

Always assert on the *message* as well as the type. `pytest.raises(ValueError)`
alone will happily pass on a `ValueError` raised by a completely different bug.

## 5. Running and selecting tests

This is where pytest earns its place in a QA workflow — you rarely want to run
everything.

```bash
pytest                                   # everything under testpaths
pytest tests/test_cart.py                # one file
pytest tests/test_cart.py::test_add_item # one test
pytest tests/api/                        # one directory
pytest -k "promo"                        # name substring match
pytest -k "cart and not checkout"        # boolean expression on names
pytest -m smoke                          # by marker
pytest -m "smoke or api"                 # markers combined
```

```bash
pytest -k "promo" -v
```

```text
collected 12 items / 9 deselected / 3 selected

tests/test_promo.py::test_promo_rejected_below_minimum_order PASSED       [ 33%]
tests/test_promo.py::test_unknown_promo_code_raises_keyerror PASSED       [ 66%]
tests/test_promo.py::test_valid_promo_applies_discount PASSED             [100%]

======================= 3 passed, 9 deselected in 0.03s ========================
```

### Flags you will use daily

| Flag | Effect |
|---|---|
| `-v` | Verbose — one line per test with its name |
| `-q` | Quiet — dots only |
| `-x` | Stop at the first failure |
| `--maxfail=3` | Stop after 3 failures |
| `-s` | Don't capture stdout — `print()` output appears live |
| `--lf` | Re-run only last-failed tests |
| `--ff` | Run last-failed first, then the rest |
| `--tb=short` / `--tb=line` / `--tb=no` | Traceback verbosity |
| `--collect-only` | List what *would* run without running it |
| `-r a` | Summary of all non-passing outcomes at the end |
| `--durations=10` | Show the 10 slowest tests |

The `--lf` / `--ff` pair changes how debugging feels. Fix a failure, run
`pytest --lf`, and you get a one-second feedback loop instead of a five-minute
one.

```bash
pytest --durations=3
```

```text
============================= slowest 3 durations ==============================
2.31s call     tests/test_setup.py::test_network_access_to_test_api
0.02s call     tests/test_promo.py::test_valid_promo_applies_discount
0.01s setup    tests/test_cart.py::TestShoppingCart::test_new_cart_is_empty
============================== 12 passed in 2.41s ==============================
```

## 6. Markers

Markers tag tests so you can run subsets. Declare them in `pytest.ini`
(Module 6), then apply them:

```python
import pytest


@pytest.mark.smoke
def test_homepage_loads():
    assert True


@pytest.mark.regression
@pytest.mark.ui
def test_checkout_flow_end_to_end():
    assert True


@pytest.mark.skip(reason="Feature not implemented until release 2.15")
def test_wishlist_sharing():
    assert False


@pytest.mark.skipif(
    not __import__("os").environ.get("RUN_SLOW_TESTS"),
    reason="Set RUN_SLOW_TESTS=1 to include slow tests",
)
def test_full_catalog_import():
    assert True


@pytest.mark.xfail(reason="BUG-118: promo rejected on sale items", strict=True)
def test_promo_applies_to_sale_items():
    assert False  # currently broken — tracked
```

```bash
pytest -v -r a
```

```text
tests/test_markers.py::test_homepage_loads PASSED                        [ 20%]
tests/test_markers.py::test_checkout_flow_end_to_end PASSED              [ 40%]
tests/test_markers.py::test_wishlist_sharing SKIPPED (Feature not
  implemented until release 2.15)                                        [ 60%]
tests/test_markers.py::test_full_catalog_import SKIPPED (Set
  RUN_SLOW_TESTS=1 to include slow tests)                                [ 80%]
tests/test_markers.py::test_promo_applies_to_sale_items XFAIL (BUG-118:
  promo rejected on sale items)                                          [100%]

=========================== short test summary info ============================
SKIPPED [1] tests/test_markers.py:18: Feature not implemented until release 2.15
SKIPPED [1] tests/test_markers.py:23: Set RUN_SLOW_TESTS=1 to include slow tests
XFAIL tests/test_markers.py::test_promo_applies_to_sale_items - BUG-118: ...
=================== 2 passed, 2 skipped, 1 xfailed in 0.04s ====================
```

| Marker | Use it when |
|---|---|
| `skip` | The test can never run here (unimplemented feature, wrong OS) |
| `skipif(condition)` | Conditionally unrunnable (missing credentials, wrong browser) |
| `xfail` | The test *should* fail — a known, tracked bug |
| Custom (`smoke`, `ui`, …) | Grouping for selective runs |

`xfail` with `strict=True` is the QA-relevant one: the test is expected to fail
because of a known defect, and if it ever *passes*, pytest reports `XPASS` as a
**failure** — telling you the bug got fixed and the marker should be removed.
That closes the loop that "commented-out test" never does.

## 7. Test outcomes

| Outcome | Symbol | Meaning |
|---|---|---|
| PASSED | `.` | Assertions held |
| FAILED | `F` | An assertion failed |
| ERROR | `E` | Something broke *outside* the test body — usually in a fixture |
| SKIPPED | `s` | Deliberately not run |
| XFAIL | `x` | Failed as expected |
| XPASS | `X` | Passed unexpectedly — investigate |

The FAILED/ERROR distinction is diagnostically important: `FAILED` means the
product may be broken; `ERROR` usually means your *setup* is broken and the
product was never actually exercised.

## 8. Test structure — Arrange, Act, Assert

Structure every test in three visible phases. It makes tests readable by people
who didn't write them, which is the whole point.

```python
def test_applying_valid_promo_reduces_order_total():
    # Arrange — set up the state
    cart_total = 2000
    promo_code = "SAVE20"
    expected_total = 1600

    # Act — perform the single action under test
    actual_total = apply_promo(cart_total, promo_code)

    # Assert — verify the outcome
    assert actual_total == pytest.approx(expected_total)
```

### Naming tests

The test name is the failure message that appears in every report. Make it a
sentence describing the behaviour and condition:

| Poor | Good |
|---|---|
| `test_login` | `test_login_succeeds_with_valid_credentials` |
| `test_1` | `test_login_fails_with_unregistered_email` |
| `test_promo` | `test_promo_rejected_when_order_below_minimum` |
| `test_cart_stuff` | `test_removing_last_item_empties_cart` |

When CI mails "1 failed:
`test_promo_rejected_when_order_below_minimum`", everyone knows what broke
before opening anything.

### The FIRST properties

Good tests are:

| Property | Meaning |
|---|---|
| **Fast** | Milliseconds, so people run them constantly |
| **Independent** | Any order, any subset, same result |
| **Repeatable** | Same result on any machine, any day |
| **Self-validating** | Pass/fail, no human reading output |
| **Timely** | Written with (or before) the code |

Independence is the one that breaks first in a real suite. If `test_b` only
passes because `test_a` created a user, then running `pytest -k test_b` fails,
parallel execution fails, and a re-run after a fix fails. Fixtures (Module 8)
are the mechanism that prevents this.

## Exercise

Write `tests/test_shipping.py` for this specification, using pure pytest:

> *Orders under ₹500: ₹60 shipping. ₹500–₹1,999: ₹30. ₹2,000 and above: free.
> Order values must be positive numbers; anything else raises `ValueError`.
> Orders above ₹1,00,000 raise `ValueError("Order exceeds maximum value")`.*

1. Write the `calculate_shipping(order_total)` function that satisfies it.
2. Write **at least 12 tests** covering the boundary values you derived in
   Module 5 (499/500/501, 1999/2000/2001, 99999/100000/100001) plus mid-partition
   values.
3. Write **three `pytest.raises` tests** for the error cases, asserting on both
   the exception type and the message.
4. Mark the three most critical tests `@pytest.mark.smoke` and verify
   `pytest -m smoke` selects exactly those three.
5. Add an `xfail(strict=True)` test for a rule that isn't implemented yet —
   free shipping for Gold members regardless of order value — and confirm it
   reports `XFAIL`. Then implement the rule and confirm the run turns into a
   failing `XPASS`, forcing you to remove the marker.
6. Deliberately break one test and capture the failure output. Then run
   `pytest --lf` and confirm only that test re-runs.
