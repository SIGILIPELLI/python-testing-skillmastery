# 10 · Project — POM Framework with API Tests

Everything in Level 2 becomes one deliverable here: a test framework you could
hand to a new team member on their first day. Page objects (01), robust waits
(02), API tests (03), mocks (04), factories (05), configuration (06), reports
(07), parallel execution (08), and cross-browser support (09) — assembled, not
demonstrated.

The target: a repo where `pytest -m smoke` runs in under a minute, the full suite
runs in parallel, and a failure produces a screenshot, a URL, and a report you can
attach to a ticket.

## 1. Repository layout

```text
qa-framework/
├── pytest.ini
├── requirements.txt
├── README.md
├── config/
│   └── environments.yaml        # base URLs and API roots per environment
├── framework/
│   ├── __init__.py
│   ├── api_client.py            # requests.Session wrapper
│   └── factories.py             # factory_boy definitions
├── pages/
│   ├── __init__.py
│   ├── base_page.py             # click / type_into / text_of — mechanics only
│   ├── login_page.py
│   ├── product_list_page.py
│   ├── cart_page.py
│   └── components/
│       ├── header_nav.py
│       └── search_widget.py
├── tests/
│   ├── conftest.py              # driver, api, base_url, screenshot hook
│   ├── api/
│   │   ├── conftest.py
│   │   ├── test_auth.py
│   │   └── test_orders.py
│   ├── ui/
│   │   ├── conftest.py
│   │   ├── test_login.py
│   │   └── test_checkout.py
│   └── unit/
│       └── test_price_rules.py  # mocked collaborators, no I/O
└── reports/                     # gitignored; junit.xml, report.html, screenshots/
```

Three rules this layout enforces: locators live only in `pages/`, HTTP lives only
in `framework/api_client.py`, and `tests/` contains assertions and nothing else.

## 2. Configuration

```ini
# pytest.ini
[pytest]
testpaths = tests
addopts =
    -ra
    --strict-markers
    --tb=short
    --junitxml=reports/junit.xml
    --html=reports/report.html --self-contained-html
markers =
    smoke: critical path, runs on every merge
    api: hits a real HTTP service
    ui: drives a browser
    slow: over five seconds
```

```yaml
# config/environments.yaml
local:
  base_url: http://localhost:8000
  api_root: http://localhost:8000/api/v1
staging:
  base_url: https://staging.example.test
  api_root: https://staging.example.test/api/v1
```

## 3. Root `conftest.py`

```python
# tests/conftest.py
import os
from pathlib import Path

import pytest
import requests
import yaml
from selenium import webdriver

REPORTS = Path("reports")


def pytest_addoption(parser):
    parser.addoption("--env", default="staging", choices=["local", "staging"])
    parser.addoption("--browsers", default="chrome")
    parser.addoption("--grid-url", default=None)


def pytest_generate_tests(metafunc):
    if "browser_name" in metafunc.fixturenames:
        metafunc.parametrize(
            "browser_name",
            metafunc.config.getoption("--browsers").split(","),
            scope="session",
        )


def pytest_configure(config):
    (REPORTS / "screenshots").mkdir(parents=True, exist_ok=True)


@pytest.fixture(scope="session")
def settings(request):
    env = request.config.getoption("--env")
    return yaml.safe_load(Path("config/environments.yaml").read_text())[env]


@pytest.fixture(scope="session")
def base_url(settings):
    return settings["base_url"]


@pytest.fixture(scope="session")
def api(settings):
    with requests.Session() as session:
        session.headers.update({"Accept": "application/json"})
        session.base_url = settings["api_root"]
        yield session


@pytest.fixture
def driver(browser_name, request):
    grid_url = request.config.getoption("--grid-url")
    options = {
        "chrome": webdriver.ChromeOptions,
        "firefox": webdriver.FirefoxOptions,
    }[browser_name]()
    options.add_argument("--headless=new" if browser_name == "chrome" else "--headless")
    options.add_argument("--window-size=1440,900")

    if grid_url:
        driver = webdriver.Remote(command_executor=grid_url, options=options)
    elif browser_name == "chrome":
        driver = webdriver.Chrome(options=options)
    else:
        driver = webdriver.Firefox(options=options)

    yield driver

    report = getattr(request.node, "report_call", None)
    if report is not None and report.failed:
        driver.save_screenshot(str(REPORTS / "screenshots" / f"{request.node.name}.png"))
    driver.quit()


@pytest.hookimpl(hookwrapper=True, tryfirst=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    report = outcome.get_result()
    setattr(item, f"report_{report.when}", report)
```

Note what is and isn't session-scoped. `api` and `settings` are cheap, shared, and
read-only — session scope is right. `driver` is function-scoped because a browser
carries cookies, local storage, and scroll position from test to test, and that's
exactly the state leak module 05 warned about.

## 4. Fast login — the biggest single speed win

Driving the login form in every UI test costs 5–10 seconds each. Authenticate
once over the API, then inject the cookie.

```python
# tests/ui/conftest.py
import pytest


@pytest.fixture
def logged_in(driver, api, base_url, make_user):
    user = make_user(password="Valid#123")
    response = api.post(f"{api.base_url}/auth/login",
                        json={"email": user.email, "password": "Valid#123"},
                        timeout=10)
    assert response.status_code == 200, response.text[:500]

    driver.get(f"{base_url}/favicon.ico")     # must be on the domain to set a cookie
    driver.add_cookie({"name": "session", "value": response.json()["token"]})
    return user
```

Keep **one** test that logs in through the real form — that path needs coverage.
Every other test uses `logged_in`.

## 5. Tests, by layer

```python
# tests/api/test_orders.py
import pytest


@pytest.mark.api
@pytest.mark.smoke
def test_create_order(api, make_user):
    user = make_user()
    response = api.post(f"{api.base_url}/orders",
                        json={"user_id": user.id, "sku": "SKU-1000", "qty": 2},
                        timeout=10)
    assert response.status_code == 201, (
        f"POST /orders returned {response.status_code}: {response.text[:500]}"
    )
    order = response.json()
    assert order["total_cents"] == 200_000


@pytest.mark.api
@pytest.mark.parametrize("qty", [0, -1, 1_000_000])
def test_invalid_quantities_rejected(api, make_user, qty):
    user = make_user()
    response = api.post(f"{api.base_url}/orders",
                        json={"user_id": user.id, "sku": "SKU-1000", "qty": qty},
                        timeout=10)
    assert response.status_code == 422
```

```python
# tests/ui/test_checkout.py
import pytest
from pages.cart_page import CartPage


@pytest.mark.ui
@pytest.mark.smoke
def test_checkout_shows_correct_total(driver, base_url, logged_in):
    cart = CartPage(driver, base_url).load()
    cart.add_item("SKU-1000", qty=2)
    assert cart.total_text() == "₹2,000.00"
```

```python
# tests/unit/test_price_rules.py
from unittest.mock import patch
from framework import pricing


def test_discount_ignored_when_service_times_out():
    with patch("framework.pricing.requests.get", side_effect=TimeoutError):
        assert pricing.final_price(1000, code="SAVE10") == 1000
```

Three layers, three costs. The unit test runs in milliseconds and covers a
failure mode the other two can't reach; the API test covers the calculation; the
UI test covers only that the number reaches the screen.

## 6. How it runs

```bash
# local loop — chrome only, smoke only
pytest -m smoke

# pre-merge
pytest -n auto --env staging

# nightly, all browsers
pytest -n 8 --browsers chrome,firefox --env staging --alluredir=reports/allure-results

# a single failing test, serially, with full traceback
pytest tests/ui/test_checkout.py::test_checkout_shows_correct_total --tb=long
```

## 7. Acceptance checklist

Your framework is done when every row is true.

| # | Requirement | How to verify |
|---|---|---|
| 1 | No locator appears outside `pages/` | `grep -rn "By\." tests/` returns nothing |
| 2 | No `assert` inside a page object | `grep -rn "assert" pages/` returns nothing |
| 3 | No `time.sleep` anywhere | `grep -rn "time.sleep" .` returns nothing |
| 4 | Tests are order-independent | `pytest -p randomly` passes three times running |
| 5 | Suite is parallel-safe | `pytest -n 4` passes; `--lf` then passes serially |
| 6 | Every test makes its own data | No shared fixed emails or ids |
| 7 | Data is cleaned up | DB/API record count is unchanged after a full run |
| 8 | Failures produce evidence | Force a failure; a screenshot exists in `reports/` |
| 9 | Environment is a flag | `--env local` and `--env staging` both work unedited |
| 10 | Markers are enforced | A typo'd marker fails collection (`--strict-markers`) |
| 11 | Smoke suite is fast | `pytest -m smoke` finishes in under 60 seconds |
| 12 | A stranger can run it | README: install, run, and interpret the report in 3 commands |

## 8. Common ways this project goes wrong

!!! warning "A `BasePage` that grows into a god object"
    If `BasePage` gains `login()`, `add_to_cart()`, and `get_order_total()`, you
    have one enormous page object wearing a base class's name. `BasePage` holds
    mechanics — click, type, wait, read text. Page vocabulary goes in the page.

!!! warning "UI tests asserting business logic"
    Verifying tax rules, discount tiers, and rounding through the checkout screen
    means fifteen slow browser tests for something four API tests cover in a
    second. Test the calculation at the API layer; test *once* through the UI
    that the number is displayed.

!!! warning "Reports written but never published"
    `reports/` in `.gitignore` and not declared as a CI artifact means the
    evidence is deleted with the build workspace. Wire the artifact upload the
    same day you add the flag.

!!! warning "A framework with no README"
    The measure of this project is not that it works on your machine — it's that
    someone else can run it without asking you. Three commands: install, run
    smoke, open the report.

## Stretch goals

1. **Visual regression.** Add a fixture that captures a full-page screenshot and
   compares it against a committed baseline with a pixel-difference threshold.
   Make one CSS change and confirm it's caught.
2. **Contract testing.** Validate every API response against a JSON Schema
   (`jsonschema` library) instead of hand-written key sets, so a dropped field
   fails one shared check rather than slipping past six tests.
3. **CI pipeline.** Write a GitHub Actions workflow: smoke on every push, full
   parallel suite nightly, JUnit annotations on the PR, and `reports/` uploaded
   as an artifact.
4. **Flakiness dashboard.** Persist Allure history across runs and use it to list
   the five most-frequently-flaky tests over the last 30 builds. Fix the top one.
5. **Grid in Docker Compose.** Stand up a hub with 2 Chrome and 2 Firefox nodes,
   point the suite at it with `--grid-url`, and compare wall time against local
   `-n 4`.
6. **Accessibility pass.** Inject `axe-core` via `execute_script` on your three
   most important pages and fail the test on any critical violation.
7. **Self-healing locators.** Log every `NoSuchElementException` with the page,
   locator, and a screenshot, and produce a weekly report of the most brittle
   locators in the suite. Then rewrite the worst three.
