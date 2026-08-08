# 07 · Reporting (pytest-html, Allure)

Terminal output is for you, while you're sitting in front of it. A report is for
everyone else: the developer who wants the failure without re-running the suite,
the release manager deciding whether to ship, the tester next Tuesday trying to
work out when a test started failing. Reports are how automation results leave
your laptop.

## 1. JUnit XML — the CI lingua franca

Before any HTML, produce the machine-readable format every CI system already
understands.

```bash
pytest --junitxml=reports/junit.xml
```

```xml
<testsuites>
  <testsuite name="pytest" errors="0" failures="1" skipped="1" tests="5" time="0.412">
    <testcase classname="tests.test_suite" name="test_login_page_loads" time="0.001"/>
    <testcase classname="tests.test_suite" name="test_orders_endpoint" time="0.002">
      <failure message="AssertionError: expected 201, got 422">...</failure>
    </testcase>
    <testcase classname="tests.test_suite" name="test_refund_flow" time="0.000">
      <skipped message="blocked by DEV-4412"/>
    </testcase>
  </testsuite>
</testsuites>
```

Jenkins, GitLab CI, GitHub Actions (via an action), Azure DevOps, and TeamCity
all read this natively — that's what turns a failure into an annotated line in a
pull request instead of a wall of console text. Generate it on every run,
including local ones; it costs nothing.

## 2. pytest-html — the shareable artifact

```bash
pip install pytest-html
pytest --html=reports/report.html --self-contained-html
```

```text
...sx                                                                    [100%]
- Generated html report: file:///work/demo/reports/report.html -
=========================== short test summary info ============================
SKIPPED [1] tests/test_suite.py:20: blocked by DEV-4412
XFAIL tests/test_suite.py::test_tax_rounding - known rounding bug DEV-4498
3 passed, 1 skipped, 1 xfailed in 0.04s
```

That run produced a 34 KB file containing the environment table, per-test
durations, filter checkboxes by outcome, and the full traceback of every failure.

`--self-contained-html` inlines the CSS and JavaScript into the single file. Skip
it and you get an HTML file plus an `assets/` directory — which arrives at your
colleague's desk as an unstyled mess the moment they email just the `.html`.
Always use it for artifacts you intend to share.

Add environment metadata so a report six months old still explains itself:

```python
# conftest.py
def pytest_metadata(metadata):
    metadata["Environment"] = "staging"
    metadata["Browser"] = "chrome 141"
    metadata["Build"] = os.getenv("BUILD_NUMBER", "local")
```

## 3. Screenshot on failure

This is the single highest-value thing in this module. A UI failure without a
screenshot is a guess; with one it's usually a five-second diagnosis.

```python
# conftest.py
import pytest


@pytest.hookimpl(hookwrapper=True, tryfirst=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    report = outcome.get_result()
    setattr(item, f"report_{report.when}", report)
```

```python
@pytest.fixture
def driver(request, browser_name):
    driver = make_driver(browser_name)
    yield driver

    report = getattr(request.node, "report_call", None)
    if report is not None and report.failed:
        path = f"reports/screenshots/{request.node.name}.png"
        driver.save_screenshot(path)
        driver.get_log("browser")          # console errors, if the driver supports it
    driver.quit()
```

Why the hook is needed: a fixture's teardown code has no idea whether the test
passed. `pytest_runtest_makereport` stashes the outcome on the test item, and the
fixture reads it back. `report_call` is the test body specifically — check
`report_setup` too if you want screenshots from failures during fixture setup.

Attach the screenshot to the HTML report so it travels with the result:

```python
# conftest.py
import pytest_html


@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    report = outcome.get_result()
    extras = getattr(report, "extras", [])
    if report.when == "call" and report.failed:
        path = f"reports/screenshots/{item.name}.png"
        extras.append(pytest_html.extras.image(path))
        extras.append(pytest_html.extras.url(item.funcargs["driver"].current_url))
    report.extras = extras
```

Save the **URL** alongside the image. Half of all UI failures turn out to be "the
test was on the wrong page", and the screenshot alone won't always show it.

## 4. Allure — history and structure

pytest-html shows you one run. Allure shows you *this test's last twenty runs*,
which is what tells flaky apart from broken.

```bash
pip install allure-pytest
pytest --alluredir=reports/allure-results
allure serve reports/allure-results
```

```python
import allure


@allure.feature("Checkout")
@allure.story("Discount codes")
@allure.severity(allure.severity_level.CRITICAL)
@allure.issue("DEV-4498", "Tax rounding on discounted totals")
def test_percentage_discount_applies(checkout_page):
    with allure.step("Add a ₹1,000 item to the cart"):
        checkout_page.add_item("SKU-1000")

    with allure.step("Apply discount code SAVE10"):
        checkout_page.apply_code("SAVE10")

    with allure.step("Verify the total"):
        allure.attach(
            checkout_page.screenshot_png(),
            name="cart-total",
            attachment_type=allure.attachment_type.PNG,
        )
        assert checkout_page.total() == "₹900.00"
```

The steps become a collapsible timeline in the report, so a failure shows exactly
which step broke — no reading the traceback to work out how far the test got.
`@allure.issue` links straight to the tracker.

The catch: Allure needs a separate Java-based CLI to render results, and history
requires you to copy the previous run's `history/` folder into the new results
directory before generating. That's a CI job, not a one-liner.

## 5. Choosing

| Format | Audience | Strength | Cost |
|---|---|---|---|
| Terminal (`-ra`, `--tb=short`) | You, right now | Instant | Gone when the shell closes |
| JUnit XML | CI system | Native PR annotations, trend graphs | Unreadable by humans |
| pytest-html | Any teammate | One file, email-able, zero setup | No history across runs |
| Allure | Team + management | Steps, attachments, flakiness history | Extra CLI, CI wiring |
| Coverage HTML (`pytest-cov`) | Developers | Shows untested lines | Measures execution, not verification |

A typical pipeline emits the first three on every run:

```bash
pytest \
  --junitxml=reports/junit.xml \
  --html=reports/report.html --self-contained-html \
  --alluredir=reports/allure-results \
  -ra
```

## 6. Traps

!!! warning "Reports written where CI can't find them"
    Write everything under one directory (`reports/`) and publish that directory
    as a build artifact. Reports scattered across the working tree, or written to
    a path that doesn't exist yet, are the most common reason a green-looking
    pipeline has no evidence attached. Create the directory in `conftest.py`
    rather than assuming it exists.

!!! warning "Screenshots that leak data"
    A failure screenshot of a logged-in page can contain a customer's name,
    address, or a partial card number, and build artifacts are usually readable
    by the whole org. Test against synthetic data (module 05), and set a
    retention policy on the artifact bucket.

!!! warning "Coverage percentage as a quality target"
    `pytest-cov` reports which lines *executed*, not which behaviours were
    *verified*. A test that calls a function and asserts nothing scores the same
    as one that checks every branch. Use coverage to find untested code; never as
    a KPI.

!!! warning "The report nobody reads"
    A daily HTML artifact with 14 long-standing failures trains the team to
    ignore the report entirely. Either fix them, or mark them `xfail` with a
    ticket reference so the summary line means something again.

## Cheat sheet

| Need | Command |
|---|---|
| CI-readable results | `--junitxml=reports/junit.xml` |
| Shareable single file | `--html=r.html --self-contained-html` |
| Allure results | `--alluredir=reports/allure-results` |
| Render Allure | `allure serve reports/allure-results` |
| Coverage report | `--cov=src --cov-report=html` |
| Summary of skips/xfails | `-ra` |
| Compact tracebacks | `--tb=short` |
| Slowest 10 tests | `--durations=10` |
| Attach an image | `pytest_html.extras.image(path)` |
| Allure step | `with allure.step("..."):` |
| Detect failure in teardown | `pytest_runtest_makereport` hook + `report_call` |

## Exercise

1. Add `--junitxml` and `--html=... --self-contained-html` to `addopts` in your
   `pytest.ini`. Confirm both files appear after a run with zero extra typing.
2. Open the HTML report and use the outcome checkboxes to show only failures.
   Then re-run without `--self-contained-html`, move just the `.html` file to
   another folder, open it, and describe what broke.
3. Implement the `pytest_runtest_makereport` hook plus a `driver` fixture that
   saves a screenshot on failure. Force a UI test to fail and confirm the PNG
   lands in `reports/screenshots/`.
4. Extend it to attach both the screenshot and `driver.current_url` to the HTML
   report, then write one sentence on a bug the URL would reveal that the image
   would not.
5. Add `allure-pytest`, wrap one test in three `allure.step` blocks, and generate
   the report. Compare how quickly you can identify the failing step versus
   reading the raw traceback.
