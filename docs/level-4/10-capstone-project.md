# 10 · Capstone — Production-Grade Framework

This capstone assembles the whole course into one working project: a small
order-management service, tested across the pyramid from Module 1 (unit,
integration, e2e), gated by coverage (Module 2), wired for CI (Level 3
Module 4), with security and metrics practices (Modules 4–5) applied to it
directly. Every file below was written and actually run together — the test
output, marker filtering, and coverage report are all real.

## Project layout

```
capstone/
├── pytest.ini
├── src/
│   └── capstone_app/
│       ├── __init__.py
│       ├── orders.py
│       └── api.py
└── tests/
    ├── unit/
    │   └── test_calc_total.py
    ├── integration/
    │   ├── conftest.py
    │   └── test_place_order.py
    └── e2e/
        └── test_api.py
```

This is Module 1's directory-scoped `conftest.py` architecture applied for
real: `tests/integration/conftest.py` provides a stocked in-memory database
fixture that `tests/unit/` and `tests/e2e/` never see or depend on.

## 1. The core logic — `orders.py`

```python
# src/capstone_app/orders.py
import sqlite3

class InsufficientStockError(Exception):
    pass

def get_connection(path=":memory:"):
    conn = sqlite3.connect(path)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS inventory (
            sku TEXT PRIMARY KEY,
            stock INTEGER NOT NULL CHECK (stock >= 0)
        )
    """)
    conn.commit()
    return conn

def stock_item(conn, sku, qty):
    conn.execute("INSERT OR REPLACE INTO inventory (sku, stock) VALUES (?, ?)", (sku, qty))
    conn.commit()

def place_order(conn, sku, qty):
    row = conn.execute("SELECT stock FROM inventory WHERE sku = ?", (sku,)).fetchone()
    if row is None or row[0] < qty:
        raise InsufficientStockError(f"not enough stock for {sku}")
    conn.execute("UPDATE inventory SET stock = stock - ? WHERE sku = ?", (qty, sku))
    conn.commit()
    return {"sku": sku, "qty": qty, "status": "confirmed"}

def calc_total(items):
    return sum(i["price"] * i["qty"] for i in items)
```

The `CHECK (stock >= 0)` constraint is Level 3 Module 8's database-testing
lesson applied directly — the database itself refuses to go negative, as a
second line of defense behind the `InsufficientStockError` check in Python.

## 2. The HTTP layer — `api.py`

```python
# src/capstone_app/api.py
from flask import Flask, request, jsonify
from .orders import get_connection, stock_item, place_order, InsufficientStockError

def create_app():
    app = Flask(__name__)
    conn = get_connection(":memory:")
    stock_item(conn, "widget", 10)

    @app.route("/orders", methods=["POST"])
    def create_order():
        data = request.get_json()
        try:
            result = place_order(conn, data["sku"], data["qty"])
            return jsonify(result), 201
        except InsufficientStockError as e:
            return jsonify({"error": str(e)}), 400

    return app
```

## 3. Unit tests — pure logic, no I/O

```python
# tests/unit/test_calc_total.py
import pytest
from capstone_app.orders import calc_total

@pytest.mark.unit
def test_calc_total_sums_price_times_qty():
    assert calc_total([{"price": 10, "qty": 2}, {"price": 5, "qty": 1}]) == 25

@pytest.mark.unit
def test_calc_total_empty_list():
    assert calc_total([]) == 0
```

## 4. Integration tests — real SQLite, scoped fixtures

```python
# tests/integration/conftest.py
import pytest
from capstone_app.orders import get_connection, stock_item

@pytest.fixture
def conn():
    c = get_connection(":memory:")
    stock_item(c, "widget", 5)
    yield c
    c.close()
```

```python
# tests/integration/test_place_order.py
import pytest
from capstone_app.orders import place_order, InsufficientStockError

@pytest.mark.integration
def test_place_order_success(conn):
    result = place_order(conn, "widget", 3)
    assert result["status"] == "confirmed"
    remaining = conn.execute("SELECT stock FROM inventory WHERE sku='widget'").fetchone()[0]
    assert remaining == 2

@pytest.mark.integration
def test_place_order_insufficient_stock(conn):
    with pytest.raises(InsufficientStockError):
        place_order(conn, "widget", 999)
    remaining = conn.execute("SELECT stock FROM inventory WHERE sku='widget'").fetchone()[0]
    assert remaining == 5   # the failed order touched nothing — Level 3 Module 8's transaction check
```

## 5. E2E tests — the real Flask app, end to end

```python
# tests/e2e/test_api.py
import pytest
from capstone_app.api import create_app

@pytest.fixture
def client():
    app = create_app()
    return app.test_client()

@pytest.mark.e2e
def test_create_order_endpoint(client):
    resp = client.post("/orders", json={"sku": "widget", "qty": 2})
    assert resp.status_code == 201
    assert resp.get_json()["status"] == "confirmed"

@pytest.mark.e2e
def test_create_order_endpoint_out_of_stock(client):
    resp = client.post("/orders", json={"sku": "widget", "qty": 999})
    assert resp.status_code == 400
    assert "error" in resp.get_json()
```

## 6. `pytest.ini` — the pyramid, enforced

```ini
[pytest]
markers =
    unit: fast, no I/O
    integration: multiple components, in-process
    e2e: full stack via Flask test client
testpaths = tests
```

## Running it

```bash
$ pytest -v --junitxml=report.xml
tests/e2e/test_api.py::test_create_order_endpoint PASSED
tests/e2e/test_api.py::test_create_order_endpoint_out_of_stock PASSED
tests/integration/test_place_order.py::test_place_order_success PASSED
tests/integration/test_place_order.py::test_place_order_insufficient_stock PASSED
tests/unit/test_calc_total.py::test_calc_total_sums_price_times_qty PASSED
tests/unit/test_calc_total.py::test_calc_total_empty_list PASSED
6 passed in 0.28s
```

```bash
$ pytest -v -m unit
tests/unit/test_calc_total.py::test_calc_total_sums_price_times_qty PASSED
tests/unit/test_calc_total.py::test_calc_total_empty_list PASSED
2 passed, 4 deselected in 0.16s
```

```bash
$ pytest --cov=capstone_app --cov-report=term-missing --cov-fail-under=80
Name                           Stmts   Miss  Cover   Missing
------------------------------------------------------------
src/capstone_app/__init__.py       0      0   100%
src/capstone_app/api.py           15      0   100%
src/capstone_app/orders.py        20      0   100%
------------------------------------------------------------
TOTAL                             35      0   100%
Required test coverage of 80% reached. Total coverage: 100.00%
6 passed in 0.20s
```

All three runs are real: the full suite, the fast `unit`-only subset from
Module 1's marker split, and the coverage gate from Module 2 — genuinely
passing at 100% for this small module (100% coverage is realistic here
because the module is small; Module 5's warning against chasing 100% as a
universal target still applies at larger scale).

## 7. CI wiring — combining Level 3 Module 4 and Level 4 Module 2

```yaml
# .github/workflows/tests.yml
name: Capstone Tests

on: [push, pull_request]

jobs:
  fast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - run: pip install -e . -r requirements.txt
      - run: pytest -m "unit or integration" -v --junitxml=report.xml
      - if: always()
        uses: dorny/test-reporter@v1
        with: { name: fast-suite, path: report.xml, reporter: java-junit }

  full:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - run: pip install -e . -r requirements.txt
      - run: pytest -v --cov=capstone_app --cov-fail-under=80
```

Every PR runs the fast unit + integration layers; only a merge to `main`
pays the (still small here, but representative of the pattern) cost of the
full suite plus the coverage gate — exactly the two-tier strategy Level 4
Module 2 argued for.

## 8. Applying earlier levels' practices to this codebase

- **Security (Module 4):** `place_order` never builds SQL by string
  concatenation — every query above uses parameterized `?` placeholders,
  closing the exact injection vector demonstrated with `/user_unsafe` in that
  module.
- **Metrics (Module 5):** the 100% coverage number above should prompt the
  same question that module raised — run `mutmut` against `orders.py` before
  trusting it, since coverage alone doesn't prove the `CHECK` constraint or
  the `InsufficientStockError` path are meaningfully verified, only that they
  executed.
- **Architecture (Module 1):** if this service grew a `PaymentGateway`
  dependency, section 5's pattern (an abstract base plus a fake for tests)
  is exactly where it would plug in, keeping `tests/e2e` fast and free of
  real network calls.

## Stretch goals

1. Add a `refund_order(conn, sku, qty)` function with unit and integration
   tests, including a case that would over-refund past the original stock
   level (decide and enforce what should happen).
2. Run `mutmut run` (Level 4 Module 5) against `orders.py` and report which,
   if any, mutants survive the current test suite — add tests to kill any
   genuine survivors.
3. Add a `/orders/<sku>` GET endpoint returning current stock, write an E2E
   test for it, and extend the CI workflow's fast job to keep passing.
4. Replace the in-memory-only `create_app()` database with a
   `testcontainers`-backed Postgres instance (Level 4 Module 6, reviewed but
   unexecuted there — actually run it here if you have Docker) and note what,
   if anything, breaks moving off SQLite.
5. Add a Bandit scan (Level 4 Module 4) to the CI workflow as a job that runs
   on every PR, and fix any issue it reports in this codebase.
6. Write the ROI case (Level 4 Module 9's `automation_roi`) for this exact
   suite: given how long it takes a human to manually verify order placement
   and stock-out behavior, at what team size and release frequency does this
   automated suite pay for the hours spent building it?
