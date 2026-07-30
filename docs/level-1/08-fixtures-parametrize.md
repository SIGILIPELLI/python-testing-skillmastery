# 08 · pytest Fixtures & Parametrize

Two features do most of the work in a real pytest suite. **Fixtures** handle
setup and teardown — the "Preconditions" row of your test case template, in code.
**Parametrize** runs one test body against many data sets — which is exactly how
equivalence partitions and boundary values from Module 5 become executable.

## 1. The problem fixtures solve

Without fixtures, setup gets duplicated into every test:

```python
def test_cart_subtotal():
    cart = ShoppingCart()          # duplicated
    cart.add("SKU-101", 500, 2)    # duplicated
    assert cart.subtotal() == 1000


def test_cart_item_count():
    cart = ShoppingCart()          # duplicated
    cart.add("SKU-101", 500, 2)    # duplicated
    assert cart.item_count() == 2
```

Change how a cart is constructed and you edit every test. Fixtures fix that.

## 2. Your first fixture

```python
import pytest


class ShoppingCart:
    def __init__(self):
        self.items = []

    def add(self, sku, price, qty=1):
        self.items.append({"sku": sku, "price": price, "qty": qty})

    def item_count(self):
        return sum(item["qty"] for item in self.items)

    def subtotal(self):
        return sum(item["price"] * item["qty"] for item in self.items)


@pytest.fixture
def cart():
    """A cart containing 2 units of SKU-101 at 500 each."""
    c = ShoppingCart()
    c.add("SKU-101", 500, 2)
    return c


def test_cart_subtotal(cart):
    assert cart.subtotal() == 1000


def test_cart_item_count(cart):
    assert cart.item_count() == 2


def test_adding_to_cart_updates_subtotal(cart):
    cart.add("SKU-202", 300, 1)
    assert cart.subtotal() == 1300
```

```text
tests/test_cart.py::test_cart_subtotal PASSED                            [ 33%]
tests/test_cart.py::test_cart_item_count PASSED                          [ 66%]
tests/test_cart.py::test_adding_to_cart_updates_subtotal PASSED          [100%]

============================== 3 passed in 0.02s ===============================
```

A test requests a fixture by **naming it as a parameter**. pytest matches the
name, runs the fixture, and passes the result in.

Critically, the third test mutates the cart and the first two are unaffected —
each test gets a **fresh** cart. That's test independence, enforced by the
framework rather than by discipline.

## 3. Setup and teardown with yield

Anything after `yield` runs as teardown, whether the test passed, failed, or
raised.

```python
import pytest


@pytest.fixture
def temp_test_file(tmp_path):
    """Create a data file before the test; delete it afterwards."""
    file_path = tmp_path / "users.csv"
    file_path.write_text("email,role\nqa.buyer01@test.com,customer\n")
    print(f"\n[setup] created {file_path}")

    yield file_path                       # the test runs here

    file_path.unlink(missing_ok=True)
    print(f"[teardown] removed {file_path}")


def test_data_file_has_one_user(temp_test_file):
    lines = temp_test_file.read_text().strip().splitlines()
    assert len(lines) == 2       # header + 1 user
    assert "qa.buyer01@test.com" in lines[1]
```

```bash
pytest tests/test_datafile.py -s
```

```text
tests/test_datafile.py::test_data_file_has_one_user
[setup] created /private/var/.../test_data_file_has_one_us0/users.csv
PASSED
[teardown] removed /private/var/.../test_data_file_has_one_us0/users.csv

============================== 1 passed in 0.02s ===============================
```

Teardown running even on failure is what stops a failing test from leaving a
locked database row, an open browser, or a logged-in session behind for the next
test to trip over.

`tmp_path` used above is one of pytest's **built-in fixtures** — a unique
temporary directory per test, cleaned up automatically.

| Built-in fixture | Provides |
|---|---|
| `tmp_path` | A unique `pathlib.Path` temp directory |
| `capsys` | Captured stdout/stderr |
| `monkeypatch` | Safe, auto-reverted patching of env vars, attributes, `sys.path` |
| `request` | Metadata about the requesting test |
| `caplog` | Captured log records |

```python
def test_error_message_is_printed(capsys):
    print("ERROR: promo code invalid")
    captured = capsys.readouterr()
    assert "promo code invalid" in captured.out


def test_base_url_defaults_to_staging(monkeypatch):
    monkeypatch.delenv("APP_BASE_URL", raising=False)
    import os
    assert os.environ.get("APP_BASE_URL", "https://staging.example.com") \
        == "https://staging.example.com"
```

```text
tests/test_builtins.py::test_error_message_is_printed PASSED             [ 50%]
tests/test_builtins.py::test_base_url_defaults_to_staging PASSED         [100%]
```

## 4. Fixture scope

Scope controls how often a fixture is created. It's the main lever on suite
speed — and the main source of test-pollution bugs when set too broadly.

| Scope | Created once per | Use for |
|---|---|---|
| `function` (default) | Each test function | Anything mutable — carts, users, records |
| `class` | Each test class | Shared read-only state within a class |
| `module` | Each test file | An expensive read-only setup, e.g. a loaded data file |
| `package` | Each package | Rare |
| `session` | The whole run | Browser drivers, DB connections, auth tokens |

```python
import pytest


@pytest.fixture(scope="session")
def api_base_url():
    print("\n[session] resolving base URL")
    return "https://api.staging.example.com"


@pytest.fixture(scope="module")
def product_catalog():
    print("[module] loading catalog")
    return [
        {"sku": "SKU-101", "name": "Trail Runner Shoes", "price": 1800, "sale": True},
        {"sku": "SKU-202", "name": "Water Bottle", "price": 300, "sale": False},
    ]


@pytest.fixture
def empty_cart():
    print("[function] new cart")
    return []


def test_catalog_has_products(product_catalog):
    assert len(product_catalog) == 2


def test_sale_item_exists(product_catalog):
    assert any(p["sale"] for p in product_catalog)


def test_cart_starts_empty(empty_cart, api_base_url):
    assert empty_cart == []
    assert api_base_url.startswith("https://")
```

```bash
pytest tests/test_scopes.py -s -q
```

```text
[session] resolving base URL
[module] loading catalog
tests/test_scopes.py .[function] new cart
..
                                                                          [100%]
3 passed in 0.02s
```

The catalog loaded once for three tests; a new cart was created only for the
test that asked for one.

!!! warning "Session scope and shared mutable state"
    A `session`-scoped fixture returning a mutable object is a classic source of
    order-dependent failures: test A appends to the list, test B asserts on its
    length and passes locally but fails when run alone. **Session scope is for
    expensive, read-only, or externally-managed resources** — a browser, a
    connection, a token. Never for test data a test might modify.

## 5. conftest.py — sharing fixtures

Fixtures defined in `conftest.py` are available to every test in that directory
and below, with no import.

`conftest.py` at project root:

```python
import os

import pytest


@pytest.fixture(scope="session")
def base_url():
    return os.environ.get("APP_BASE_URL", "https://staging.example.com")


@pytest.fixture(scope="session")
def credentials():
    return {
        "email": os.environ.get("TEST_USER", "qa.buyer01@test.com"),
        "password": os.environ.get("TEST_PASSWORD", "Valid#123"),
    }


@pytest.fixture
def cart():
    return ShoppingCart()
```

`tests/test_login.py` — no import needed:

```python
def test_login_page_url_is_correct(base_url):
    assert f"{base_url}/login".endswith("/login")


def test_credentials_are_populated(credentials):
    assert "@" in credentials["email"]
    assert len(credentials["password"]) >= 8
```

```text
tests/test_login.py::test_login_page_url_is_correct PASSED               [ 50%]
tests/test_login.py::test_credentials_are_populated PASSED               [100%]
```

You can nest `conftest.py` files: one at the root for global fixtures, one in
`tests/api/` for API-only fixtures, one in `tests/ui/` for the browser driver.
The closest one wins when names collide.

## 6. Fixtures using other fixtures

Fixtures compose — request one from another, exactly as a test does.

```python
import pytest


@pytest.fixture(scope="session")
def api_client(base_url):
    import requests
    session = requests.Session()
    session.headers.update({"Accept": "application/json"})
    session.base_url = base_url
    yield session
    session.close()


@pytest.fixture
def registered_user(api_client):
    """Create a user for this test, then delete it."""
    user = {"email": "temp.user@test.com", "role": "customer"}
    # response = api_client.post(f"{api_client.base_url}/users", json=user)
    print(f"\n[setup] created user {user['email']}")

    yield user

    # api_client.delete(f"{api_client.base_url}/users/{user['email']}")
    print(f"[teardown] deleted user {user['email']}")


def test_new_user_has_customer_role(registered_user):
    assert registered_user["role"] == "customer"
```

```text
[setup] created user temp.user@test.com
PASSED
[teardown] deleted user temp.user@test.com

============================== 1 passed in 0.02s ===============================
```

This chain — `base_url` → `api_client` → `registered_user` → test — is the
backbone of every real automation framework. Each layer sets up exactly what it
owns and tears down exactly what it created.

## 7. autouse fixtures

An `autouse=True` fixture runs for every test in its scope without being
requested.

```python
import time

import pytest


@pytest.fixture(autouse=True)
def log_test_boundaries(request):
    start = time.time()
    print(f"\n>>> START {request.node.name}")
    yield
    print(f"<<< END   {request.node.name} ({time.time() - start:.3f}s)")


def test_one():
    assert True


def test_two():
    assert 1 + 1 == 2
```

```text
>>> START test_one
PASSED<<< END   test_one (0.000s)

>>> START test_two
PASSED<<< END   test_two (0.000s)

============================== 2 passed in 0.02s ===============================
```

Good uses: logging, resetting global state, clearing caches, taking a screenshot
on failure. Use sparingly — an autouse fixture is invisible in the test
signature, so a test that mysteriously depends on hidden setup is harder to
debug.

## 8. Parametrize — one test, many data sets

`@pytest.mark.parametrize` runs the same test body against multiple inputs, each
reported as a separate test. This is the direct executable form of equivalence
partitioning and boundary value analysis.

```python
import pytest


def calculate_shipping(order_total):
    if order_total < 0:
        raise ValueError("Order total must be positive")
    if order_total >= 2000:
        return 0
    if order_total >= 500:
        return 30
    return 60


@pytest.mark.parametrize(
    "order_total, expected_shipping",
    [
        (250, 60),       # EP: mid low partition
        (499, 60),       # BVA: min - 1
        (500, 30),       # BVA: boundary
        (501, 30),       # BVA: min + 1
        (1200, 30),      # EP: mid middle partition
        (1999, 30),      # BVA: max - 1
        (2000, 0),       # BVA: boundary
        (2001, 0),       # BVA: max + 1
        (50000, 0),      # EP: mid high partition
    ],
)
def test_shipping_cost_for_order_value(order_total, expected_shipping):
    assert calculate_shipping(order_total) == expected_shipping
```

```bash
pytest tests/test_shipping.py -v
```

```text
collected 9 items

tests/test_shipping.py::test_shipping_cost_for_order_value[250-60] PASSED  [ 11%]
tests/test_shipping.py::test_shipping_cost_for_order_value[499-60] PASSED  [ 22%]
tests/test_shipping.py::test_shipping_cost_for_order_value[500-30] PASSED  [ 33%]
tests/test_shipping.py::test_shipping_cost_for_order_value[501-30] PASSED  [ 44%]
tests/test_shipping.py::test_shipping_cost_for_order_value[1200-30] PASSED [ 55%]
tests/test_shipping.py::test_shipping_cost_for_order_value[1999-30] PASSED [ 66%]
tests/test_shipping.py::test_shipping_cost_for_order_value[2000-0] PASSED  [ 77%]
tests/test_shipping.py::test_shipping_cost_for_order_value[2001-0] PASSED  [ 88%]
tests/test_shipping.py::test_shipping_cost_for_order_value[50000-0] PASSED [100%]

============================== 9 passed in 0.03s ===============================
```

Nine test cases, one test body. And when one fails, the report names the exact
data set:

```text
FAILED tests/test_shipping.py::test_shipping_cost_for_order_value[2000-0]
  - assert 30 == 0
```

That's the boundary bug, identified precisely, with no debugging.

### Readable IDs

Auto-generated IDs get unreadable with complex data. Name them:

```python
@pytest.mark.parametrize(
    "email, is_valid",
    [
        pytest.param("user@example.com", True, id="standard"),
        pytest.param("user+alias@example.com", True, id="plus_alias"),
        pytest.param("user.name@sub.example.co.in", True, id="subdomain"),
        pytest.param("user@", False, id="missing_domain"),
        pytest.param("@example.com", False, id="missing_local_part"),
        pytest.param("user@@example.com", False, id="double_at"),
        pytest.param("", False, id="empty"),
        pytest.param("   ", False, id="whitespace_only"),
        pytest.param("a" * 250 + "@example.com", False, id="over_length"),
    ],
)
def test_email_validation(email, is_valid):
    import re
    pattern = r"^[^@\s]+@[^@\s]+\.[^@\s]+$"
    assert bool(re.match(pattern, email)) and len(email) <= 254 or not is_valid
```

```text
tests/test_email.py::test_email_validation[standard] PASSED               [ 11%]
tests/test_email.py::test_email_validation[plus_alias] PASSED             [ 22%]
tests/test_email.py::test_email_validation[subdomain] PASSED              [ 33%]
tests/test_email.py::test_email_validation[missing_domain] PASSED         [ 44%]
tests/test_email.py::test_email_validation[missing_local_part] PASSED     [ 55%]
tests/test_email.py::test_email_validation[double_at] PASSED              [ 66%]
tests/test_email.py::test_email_validation[empty] PASSED                  [ 77%]
tests/test_email.py::test_email_validation[whitespace_only] PASSED        [ 88%]
tests/test_email.py::test_email_validation[over_length] PASSED            [100%]
```

Now `pytest -k "plus_alias"` runs exactly that partition.

### Marking individual parameters

```python
@pytest.mark.parametrize(
    "code, expected_discount",
    [
        pytest.param("SAVE20", 0.20, id="valid"),
        pytest.param("save20", 0.20, id="lowercase"),
        pytest.param(" SAVE20 ", 0.20, id="padded",
                     marks=pytest.mark.xfail(reason="BUG-125: whitespace not trimmed")),
        pytest.param("EXPIRED10", 0.0, id="expired"),
    ],
)
def test_promo_discount(code, expected_discount):
    normalised = code.strip().upper()
    discount = 0.20 if normalised == "SAVE20" else 0.0
    assert discount == pytest.approx(expected_discount)
```

You can mark one parameter `xfail` or `skip` without disabling the whole test —
so a known bug in one partition doesn't cost you coverage of the other eight.

### Stacking parametrize decorators

Two stacked decorators produce the Cartesian product — useful for cross-browser
or role × permission matrices:

```python
@pytest.mark.parametrize("browser", ["chrome", "firefox"])
@pytest.mark.parametrize("role", ["guest", "customer", "admin"])
def test_page_access(browser, role):
    assert browser in ("chrome", "firefox")
    assert role in ("guest", "customer", "admin")
```

```text
collected 6 items

tests/test_matrix.py::test_page_access[guest-chrome] PASSED              [ 16%]
tests/test_matrix.py::test_page_access[guest-firefox] PASSED             [ 33%]
tests/test_matrix.py::test_page_access[customer-chrome] PASSED           [ 50%]
tests/test_matrix.py::test_page_access[customer-firefox] PASSED          [ 66%]
tests/test_matrix.py::test_page_access[admin-chrome] PASSED              [ 83%]
tests/test_matrix.py::test_page_access[admin-firefox] PASSED             [100%]
```

Two lines of parameters, six executed test cases. Be deliberate — three stacked
decorators of five values each is 125 tests.

## 9. Parametrized fixtures

Parametrize the *fixture* instead of the test, and every test using it runs once
per parameter. This is how cross-browser suites work (Level 2).

```python
import pytest


@pytest.fixture(params=["chrome", "firefox", "edge"])
def browser_name(request):
    return request.param


def test_browser_is_supported(browser_name):
    assert browser_name in {"chrome", "firefox", "edge"}


def test_browser_name_is_lowercase(browser_name):
    assert browser_name == browser_name.lower()
```

```text
collected 6 items

tests/test_browsers.py::test_browser_is_supported[chrome] PASSED         [ 16%]
tests/test_browsers.py::test_browser_is_supported[firefox] PASSED        [ 33%]
tests/test_browsers.py::test_browser_is_supported[edge] PASSED           [ 50%]
tests/test_browsers.py::test_browser_name_is_lowercase[chrome] PASSED    [ 66%]
tests/test_browsers.py::test_browser_name_is_lowercase[firefox] PASSED   [ 83%]
tests/test_browsers.py::test_browser_name_is_lowercase[edge] PASSED      [100%]
```

## 10. Fixtures vs parametrize — which one

| Need | Use |
|---|---|
| Same setup, different assertions | **Fixture** |
| Same assertion, different data | **Parametrize** |
| Expensive resource shared by many tests | **Fixture**, scoped up |
| Boundary values and equivalence partitions | **Parametrize** |
| Run the whole suite against several configurations | **Parametrized fixture** |
| Cleanup required after the test | **Fixture** with `yield` |

## Exercise

Build `tests/test_booking.py` for a hotel-booking price calculator:

> *Base rate ₹4,000/night. Stays of 7+ nights get 10% off. Gold members get 15%
> off; Silver 5%; Bronze 0%. Discounts do not stack — the larger applies. Guests
> beyond 2 add ₹800/night each, up to 8 guests. Fewer than 1 night or more than
> 8 guests raises `ValueError`.*

1. Write `calculate_price(nights, guests, tier)` implementing this.
2. Create a `conftest.py` with a session-scoped `pricing_config` fixture holding
   the base rate and discount percentages, and a function-scoped `booking`
   fixture returning a default 2-night, 2-guest, Bronze booking dict.
3. Write a **parametrized test with at least 12 cases** covering the nights
   boundary (6/7/8), the guest boundary (2/3/8/9), and all three tiers. Give
   every case a readable `id`.
4. Write a **stacked parametrize** covering tier × guest-count (3 × 4 = 12
   generated cases) asserting the price is always positive and increases with
   guest count.
5. Add a fixture with `yield` that logs the booking under test and prints the
   computed price at teardown; verify with `-s` that teardown runs even when a
   test fails.
6. Add one `xfail`-marked parameter for a rule not yet implemented (children
   under 5 are free) and confirm it reports `XFAIL` without failing the run.
