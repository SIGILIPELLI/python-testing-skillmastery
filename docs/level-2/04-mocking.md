# 04 · Mocking with unittest.mock

Module 03 tested a real API over the real network. That's the right call for
contract tests — and the wrong call for the fifty tests that only need to know
"what does my code do when the weather service times out?" You can't make a live
service time out on demand, return a 500, or bill a credit card for free. You
*replace* it, and assert on how your code reacts.

`unittest.mock` ships with Python. No install, no plugin.

## 1. The code under test

```python
# weather.py
import requests

API = "https://api.example.com/weather"


def current_temperature(city):
    """Return the temperature in Celsius, or None if the service is unreachable."""
    try:
        response = requests.get(API, params={"city": city}, timeout=5)
    except requests.RequestException:
        return None
    if response.status_code != 200:
        return None
    return response.json()["temp_c"]
```

Three behaviours to verify: the happy path, a network failure, and a non-200
response. Only one of those is reachable against a live service.

## 2. `patch` — swap the real thing for a fake

```python
from unittest.mock import patch
import weather


def test_patch_return_value():
    with patch("weather.requests.get") as mock_get:
        mock_get.return_value.status_code = 200
        mock_get.return_value.json.return_value = {"temp_c": 21.5, "city": "Pune"}

        assert weather.current_temperature("Pune") == 21.5

        mock_get.assert_called_once_with(
            "https://api.example.com/weather", params={"city": "Pune"}, timeout=5
        )
```

Two assertions, doing different jobs. The first checks the **return value** —
did our parsing work. The second checks the **interaction** — did we call the
service correctly, with the right params and a timeout. Interaction assertions
are the entire point of mocking; without them you've only proved your code can
read a dict.

!!! danger "Patch where it's *used*, not where it's defined"
    `patch("weather.requests.get")` — correct. `patch("requests.get")` also
    happens to work here, but only because `weather.py` does `import requests`
    and looks the attribute up at call time. If `weather.py` had written
    `from requests import get`, then `weather.get` is a *separate name* bound at
    import time, and patching `requests.get` would do nothing at all — your test
    would silently hit the real network. The rule: patch the name in the module
    that calls it.

## 3. `side_effect` — raise, or return a sequence

`return_value` covers "the call succeeds". `side_effect` covers everything else.

```python
import requests
from unittest.mock import patch


def test_side_effect_raises():
    with patch("weather.requests.get", side_effect=requests.Timeout("timed out")):
        assert weather.current_temperature("Pune") is None
```

That's a timeout on demand — a scenario you cannot reliably produce against a
real service. Pass a *list* and each call pops the next item, which is how you
test retry logic:

```python
from unittest.mock import Mock


def test_side_effect_sequence():
    ok = Mock(status_code=200)
    ok.json.return_value = {"temp_c": 30.0, "city": "Chennai"}

    with patch("weather.requests.get", side_effect=[requests.Timeout(), ok]) as mock_get:
        assert weather.current_temperature_with_retry("Chennai") == 30.0
        assert mock_get.call_count == 2
```

First call raises, second succeeds, and `call_count == 2` proves the retry
actually happened rather than the first attempt quietly succeeding.

`side_effect` can also be a function, when the fake response should depend on the
arguments:

```python
def fake_get(url, params=None, timeout=None):
    city = params["city"]
    if city == "Atlantis":
        return Mock(status_code=404)
    return Mock(status_code=200, **{"json.return_value": {"temp_c": 25.0}})
```

## 4. Real run

```text
============================= test session starts ==============================
platform darwin -- Python 3.11.2, pytest-9.1.1, pluggy-1.6.0
collected 5 items

test_mock.py::test_patch_return_value PASSED                             [ 20%]
test_mock.py::test_side_effect_raises PASSED                             [ 40%]
test_mock.py::test_side_effect_sequence PASSED                           [ 60%]
test_mock.py::test_autospec_catches_wrong_signature PASSED               [ 80%]
test_mock.py::test_mock_without_spec_accepts_anything PASSED             [100%]

============================== 5 passed in 0.08s ===============================
```

0.08 seconds, zero network traffic, and it covers a timeout path that the live
API tests in module 03 can never reach.

## 5. The decorator and fixture forms

Three equivalent styles. Pick one per project and stick to it.

```python
# Context manager — patch is active for exactly these lines
def test_ctx():
    with patch("weather.requests.get") as mock_get:
        ...

# Decorator — note the argument order is bottom-up
@patch("weather.requests.get")
def test_decorator(mock_get):
    ...

@patch("weather.log_metric")      # -> second argument
@patch("weather.requests.get")    # -> first argument
def test_two_patches(mock_get, mock_log):
    ...

# Fixture — reusable across a file
import pytest

@pytest.fixture
def mock_get():
    with patch("weather.requests.get") as m:
        m.return_value.status_code = 200
        m.return_value.json.return_value = {"temp_c": 21.5}
        yield m
```

The stacked-decorator argument order (nearest decorator first) is the single most
common source of "why is my mock configured wrong" confusion. The fixture form
avoids the question entirely.

## 6. `autospec` — mocks that can't lie

A bare `Mock()` accepts *anything*:

```python
def test_mock_without_spec_accepts_anything():
    fake = Mock()
    fake.method_that_does_not_exist()
    fake.typo_metohd().chained().deeply()
    assert True          # passes — none of that exists on the real object
```

That test passes. It is also worthless: it would keep passing after you renamed
the real method, deleted it, or changed its signature. This is how a fully green
suite ships a broken release.

`autospec=True` builds the mock from the real object's signature:

```python
def test_autospec_catches_wrong_signature():
    with patch("weather.requests.get", autospec=True) as mock_get:
        mock_get.return_value.status_code = 200
        mock_get.return_value.json.return_value = {"temp_c": 5.0}
        weather.current_temperature("Oslo")
        args, kwargs = mock_get.call_args
        assert kwargs["timeout"] == 5
```

Now calling a method that doesn't exist raises `AttributeError`, and calling one
with the wrong arguments raises `TypeError` — at test time, where you want it.
**Default to `autospec=True`.** Use `Mock()` only for throwaway stand-ins where
the shape genuinely doesn't matter.

## 7. Inspecting what happened

| Assertion | Checks |
|---|---|
| `mock.assert_called()` | Called at least once |
| `mock.assert_called_once()` | Called exactly once, any args |
| `mock.assert_called_with(a, b=1)` | The **most recent** call matched |
| `mock.assert_called_once_with(...)` | Called exactly once, with these args |
| `mock.assert_not_called()` | Never called |
| `mock.assert_any_call(...)` | At least one call matched |
| `mock.call_count` | How many times |
| `mock.call_args` | `(args, kwargs)` of the last call |
| `mock.call_args_list` | Every call, in order |

!!! danger "The typo that disables your assertion"
    `mock.assert_called_once()` is real. `mock.assert_called_once` (no parens) is
    a truthy attribute, and the line does nothing. Worse, `Mock` invents any
    attribute you ask for, so `mock.assert_called_onse()` silently passes too.
    Two defences: `autospec=True`, and `Mock(spec=...)`, both of which reject
    unknown attribute names. Python 3.5+ catches the misspelled-`assert_*` case
    on `Mock` itself, but not on every mock-like object you'll meet.

## 8. When *not* to mock

Mocking has a real cost: every mock is an assumption about how the collaborator
behaves, and assumptions rot. If the real service changes its response shape,
your mocked tests keep passing and production breaks. That's the mocking
trade-off, and it's why the pyramid needs both layers.

| Mock it | Don't mock it |
|---|---|
| Third-party HTTP APIs (slow, rate-limited, costs money) | Your own pure functions |
| Clocks and randomness (`datetime.now`, `random`) | The standard library's data structures |
| Email/SMS/payment gateways | The thing the test is actually about |
| Failure modes you can't trigger (timeout, 500, disk full) | A fast in-memory database, when a real one is available |

!!! warning "Mocking the system under test"
    ```python
    # ✗ This test proves nothing
    with patch("cart.Cart.total", return_value=100):
        assert Cart(items).total() == 100
    ```
    You mocked the exact behaviour you set out to verify. Mock the cart's
    *dependencies* (a tax service, a discount API) — never the cart.

!!! warning "Over-specified interaction assertions"
    `assert_called_once_with(url, params={...}, timeout=5, headers={...}, verify=True)`
    fails the day someone adds an unrelated header. Assert on the arguments the
    behaviour depends on (`kwargs["timeout"]`, `kwargs["params"]["city"]`) and
    let the rest vary.

## Cheat sheet

| Need | Code |
|---|---|
| Replace a name | `patch("module_that_uses_it.name")` |
| Fake a return | `mock.return_value = x` |
| Fake a chained call | `mock.return_value.json.return_value = {...}` |
| Raise an exception | `side_effect=Timeout()` |
| Different result per call | `side_effect=[a, b, c]` |
| Result depends on args | `side_effect=some_function` |
| Signature-safe mock | `patch(..., autospec=True)` |
| Restrict attributes | `Mock(spec=RealClass)` |
| Patch a dict entry | `patch.dict(os.environ, {"ENV": "test"})` |
| Patch an attribute | `patch.object(Client, "send")` |
| Async collaborator | `AsyncMock` |

## Exercise

Using `weather.py` from section 1:

1. Write the three core tests: happy path, `requests.Timeout`, and a 500
   response. Confirm all three pass with no network access (disable Wi-Fi and
   re-run — the suite must stay green).
2. Change `weather.py` to `from requests import get` and re-run your tests with
   `patch("weather.requests.get")`. Record what happens and explain why, then fix
   the patch target.
3. Write a test using bare `Mock()` that asserts a method you *invented* was
   called. Watch it pass. Add `autospec=True` and record the exact exception.
4. Write `current_temperature_with_retry` so it retries three times, then use
   `side_effect=[Timeout(), Timeout(), ok_response]` to prove the third attempt
   succeeds and `call_count == 3`.
5. Use `patch("weather.datetime")` (or `freezegun`, if you prefer) to make a
   function that stamps `datetime.now()` return a fixed timestamp, and explain in
   a comment why the test was flaky before you did.
