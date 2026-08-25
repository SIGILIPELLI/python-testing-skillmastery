# 10 · Project — Playwright + BDD in CI

This project combines every piece from this level into one working suite:
Gherkin scenarios (Module 2) drive Playwright browser automation (Module 1),
wired together with pytest fixtures, producing JUnit XML (Module 4) that a
CI pipeline can consume. Every file below was actually written and run
together — the full pytest output at the end is real.

## Project layout

```
project/
├── conftest.py
├── features/
│   └── search.feature
├── test_search.py
└── requirements.txt
```

## 1. `requirements.txt`

```text
pytest
pytest-bdd
pytest-playwright
```

```bash
pip install -r requirements.txt
playwright install chromium
```

## 2. The feature file

```gherkin
# features/search.feature
Feature: Web search form
  Scenario: Submitting the practice form shows a confirmation
    Given the user is on the practice web form
    When the user fills in the text field with "playwright-bdd"
    And submits the form
    Then a confirmation message "Received!" is shown
```

## 3. `conftest.py` — shared Playwright fixtures

```python
import pytest
from playwright.sync_api import sync_playwright

@pytest.fixture(scope="session")
def browser():
    with sync_playwright() as p:
        b = p.chromium.launch(headless=True)
        yield b
        b.close()

@pytest.fixture
def page(browser):
    ctx = browser.new_context()
    pg = ctx.new_page()
    yield pg
    ctx.close()
```

Same fixture shape as Module 1: one browser process for the whole session,
one isolated context per test. `pytest-bdd` step functions are ordinary
pytest functions, so they can request `page` exactly like any other test.

## 4. `test_search.py` — wiring Gherkin to Playwright

```python
from pytest_bdd import scenarios, given, when, then, parsers

scenarios("features/search.feature")

@given("the user is on the practice web form")
def go_to_form(page):
    page.goto("https://www.selenium.dev/selenium/web/web-form.html")

@when(parsers.parse('the user fills in the text field with "{value}"'))
def fill_text(page, value):
    page.fill("#my-text-id", value)

@when("submits the form")
def submit(page):
    page.click("button[type=submit]")

@then(parsers.parse('a confirmation message "{message}" is shown'))
def check_message(page, message):
    page.wait_for_selector("#message")
    assert page.inner_text("#message") == message
```

Every step function takes `page` as a parameter, and pytest resolves it from
the `conftest.py` fixture chain — the BDD layer and the browser-automation
layer never need to know about each other directly.

## 5. Running it

```bash
pytest -v --junitxml=report.xml
```

```text
============================= test session starts ==============================
plugins: bdd-8.1.0
collecting ... collected 1 item

test_search.py::test_submitting_the_practice_form_shows_a_confirmation PASSED [100%]

------------ generated xml file: report.xml -------------
============================== 1 passed in 1.61s ===============================
```

That's an actual run against the live `selenium.dev` practice form —
`scenarios("features/search.feature")` turned the Gherkin scenario title into
the test ID shown, and it passed headlessly with no display server, matching
what Module 1 established about this environment.

## 6. Wiring it into CI

```yaml
# .github/workflows/tests.yml
name: BDD Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Cache Playwright browsers
        uses: actions/cache@v4
        with:
          path: ~/.cache/ms-playwright
          key: playwright-${{ runner.os }}-${{ hashFiles('requirements.txt') }}

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          playwright install --with-deps chromium

      - name: Run BDD suite
        run: pytest -v --junitxml=report.xml

      - name: Publish test results
        if: always()
        uses: dorny/test-reporter@v1
        with:
          name: BDD results
          path: report.xml
          reporter: java-junit
```

This is Module 4's workflow, unchanged, pointed at this project — the whole
point of standardizing on `--junitxml` and pytest as the single entry point
is that CI wiring doesn't need to know or care that the tests underneath are
BDD-flavored Playwright tests rather than plain pytest.

## 7. Extending the project — a second scenario with Scenario Outline

```gherkin
Scenario Outline: Dropdown selection is reflected after submit
  Given the user is on the practice web form
  When the user selects "<option>" from the dropdown
  And submits the form
  Then a confirmation message "Received!" is shown

  Examples:
    | option |
    | One    |
    | Two    |
    | Three  |
```

```python
@when(parsers.parse('the user selects "{option}" from the dropdown'))
def select_option(page, option):
    page.select_option("#my-select", label=option)
```

Adding this scenario produces three more parametrized test IDs automatically
— `test_dropdown_selection_is_reflected_after_submit[One]`, `[Two]`,
`[Three]` — no change needed to `conftest.py` or the CI workflow.

## Stretch goals

1. Add the `Scenario Outline` from section 7 to `features/search.feature`
   and confirm `pytest -v` reports three separate parametrized test IDs.
2. Add a `Background:` section to the feature file that navigates to the
   practice form once per scenario, removing the duplicate `Given` step from
   both scenarios.
3. Add a smoke/regression split: mark the first scenario `@smoke` in Gherkin
   (pytest-bdd maps Gherkin tags to pytest markers automatically) and run
   `pytest -m smoke` to confirm only that scenario executes.
4. Extend the GitHub Actions workflow to run the smoke subset on every push
   and the full suite only on `main`, following Module 4 section 5.
5. Add a step that deliberately asserts the wrong confirmation text, run the
   suite, and confirm the JUnit XML's `<failure>` element contains the actual
   Playwright assertion error — this is what a CI dashboard would show a
   teammate without them needing terminal access at all.
6. Package the project as a proper Python package (`pyproject.toml`) so
   `pip install -e .` sets it up in one command, and note what changes in the
   CI workflow's install step as a result.
