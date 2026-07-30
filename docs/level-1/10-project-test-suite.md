# 10 · Project — Test Plan + Automated Suite

This project combines everything in Level 1: you'll write a real **test plan**
and **test cases** for a live web page (manual testing, Modules 1–5), then
automate the regression-worthy parts with **pytest and Selenium** (Modules 6–9),
and produce a **traceability matrix** and **test summary report** that tie the
two together.

## The application under test

We'll use the Selenium project's own practice page — publicly available, stable,
and rich enough to exercise every technique:

**`https://www.selenium.dev/selenium/web/web-form.html`**

It contains: a text input, a password input, a textarea, a disabled input, a
readonly input, a dropdown (`<select>`), a datalist, a file input, checkboxes,
radio buttons, a colour picker, a date picker, a range slider, and a submit
button that navigates to a confirmation state showing "Received!".

Treat it as if it were a production form, and these were the requirements:

| ID | Requirement |
|---|---|
| REQ-FORM-01 | The form page loads with the title "Web form" and a visible heading |
| REQ-FORM-02 | The text input accepts free text up to 255 characters |
| REQ-FORM-03 | The password input masks entered characters |
| REQ-FORM-04 | The textarea accepts multi-line input |
| REQ-FORM-05 | The disabled input cannot be edited |
| REQ-FORM-06 | The readonly input displays a value that cannot be changed |
| REQ-FORM-07 | The dropdown offers exactly three selectable options |
| REQ-FORM-08 | Checkboxes and radio buttons are independently selectable |
| REQ-FORM-09 | Submitting the form displays the confirmation message "Received!" |
| REQ-FORM-10 | Submitted values appear as query parameters in the resulting URL |

## Deliverable 1 — The test plan

Write `docs/test-plan.md`. This is a real document; keep it to two pages.

### Required sections

```markdown
# Test Plan — Web Form

## 1. Introduction
Purpose of this document and the feature under test.

## 2. Scope
### In scope
List what will be tested (functional behaviour of each form control,
submission, confirmation, URL parameters).
### Out of scope
List what will not, **and why** (e.g. performance testing — no SLA defined;
cross-browser beyond Chrome — single-browser mandate for this phase;
accessibility audit — separate specialist engagement).

## 3. Test approach
Which techniques from Module 5 you will apply to which controls, and the
manual/automation split with your justification for each side of the line.

## 4. Test environment
Browser and version, OS, screen resolution, URL, Python and package versions.

## 5. Entry and exit criteria
### Entry criteria
e.g. page reachable, environment configured, test cases reviewed.
### Exit criteria
e.g. 100% of P1 cases executed, 0 open S1/S2 defects, ≥95% pass rate,
all automated tests green in two consecutive runs.

## 6. Test deliverables
Test cases, RTM, automated suite, defect reports, test summary report.

## 7. Risks and mitigations
At least four, in a table: risk · likelihood · impact · mitigation.
e.g. "Public practice page may change without notice" ·
Medium · High · "Pin expectations to stable attributes; monitor daily run".

## 8. Schedule and effort
Phase-by-phase estimate in hours.

## 9. Roles and responsibilities
Who writes cases, who executes, who fixes, who signs off.
```

## Deliverable 2 — Manual test cases

Write `docs/test-cases.md` with **at least 20 test cases** using the full
template from Module 2 — every field, numbered steps, specific and observable
expected results.

Coverage requirements:

| Requirement | Minimum cases | Techniques to apply |
|---|---|---|
| REQ-FORM-01 | 2 | Smoke |
| REQ-FORM-02 | 5 | Equivalence partitioning + BVA (0, 1, 254, 255, 256 chars) |
| REQ-FORM-03 | 2 | Functional + negative |
| REQ-FORM-04 | 2 | Functional (multi-line, special characters) |
| REQ-FORM-05/06 | 2 | Negative |
| REQ-FORM-07 | 3 | Equivalence partitioning (each option) |
| REQ-FORM-08 | 2 | State transition (toggle on/off, radio exclusivity) |
| REQ-FORM-09/10 | 4 | Functional + integration |

At least four of your cases must be **negative** cases, and at least four must
be **boundary** cases. Assign a priority (P1–P4) to every case and mark the
five you would automate first.

## Deliverable 3 — The automated suite

### Project structure

```text
web-form-tests/
├── .gitignore
├── requirements.txt
├── pytest.ini
├── conftest.py
├── README.md
├── docs/
│   ├── test-plan.md
│   ├── test-cases.md
│   ├── traceability-matrix.md
│   └── test-summary-report.md
├── tests/
│   ├── __init__.py
│   ├── test_form_load.py
│   ├── test_text_inputs.py
│   ├── test_selection_controls.py
│   └── test_form_submission.py
└── reports/
    └── screenshots/
```

### pytest.ini

```ini
[pytest]
testpaths = tests
addopts = -v --strict-markers --tb=short -r a
markers =
    smoke: build-verification checks
    regression: full regression suite
    ui: drives a real browser
    boundary: boundary value analysis cases
    negative: negative test cases
```

### conftest.py

```python
import os
from pathlib import Path

import pytest
from selenium import webdriver
from selenium.webdriver.support.ui import WebDriverWait

FORM_PATH = "/selenium/web/web-form.html"


@pytest.fixture(scope="session")
def base_url():
    return os.environ.get("APP_BASE_URL", "https://www.selenium.dev")


@pytest.fixture(scope="session")
def form_url(base_url):
    return f"{base_url}{FORM_PATH}"


@pytest.hookimpl(hookwrapper=True, tryfirst=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    report = outcome.get_result()
    setattr(item, f"report_{report.when}", report)


@pytest.fixture
def driver(request):
    options = webdriver.ChromeOptions()
    if os.environ.get("HEADLESS", "1") == "1":
        options.add_argument("--headless=new")
    options.add_argument("--window-size=1920,1080")
    options.add_argument("--disable-gpu")
    options.add_argument("--no-sandbox")
    options.add_argument("--disable-dev-shm-usage")

    driver = webdriver.Chrome(options=options)
    driver.set_page_load_timeout(30)

    yield driver

    report = getattr(request.node, "report_call", None)
    if report is not None and report.failed:
        folder = Path("reports/screenshots")
        folder.mkdir(parents=True, exist_ok=True)
        path = folder / f"{request.node.name}.png"
        driver.save_screenshot(str(path))
        print(f"\n[screenshot] {path}")

    driver.quit()


@pytest.fixture
def wait(driver):
    return WebDriverWait(driver, timeout=10)


@pytest.fixture
def form_page(driver, form_url, wait):
    """Navigate to the form and return the driver, ready for interaction."""
    driver.get(form_url)
    from selenium.webdriver.common.by import By
    from selenium.webdriver.support import expected_conditions as EC
    wait.until(EC.visibility_of_element_located((By.NAME, "my-text")))
    return driver
```

### tests/test_form_load.py

```python
"""REQ-FORM-01 — page loads correctly."""
import pytest
from selenium.webdriver.common.by import By


@pytest.mark.ui
@pytest.mark.smoke
def test_page_title_is_web_form(form_page):
    """TC-FORM-001 · REQ-FORM-01"""
    assert form_page.title == "Web form"


@pytest.mark.ui
@pytest.mark.smoke
def test_page_heading_is_visible(form_page):
    """TC-FORM-002 · REQ-FORM-01"""
    heading = form_page.find_element(By.TAG_NAME, "h1")
    assert heading.is_displayed()
    assert heading.text == "Web form"


@pytest.mark.ui
@pytest.mark.smoke
def test_all_primary_controls_are_present(form_page):
    """TC-FORM-003 · REQ-FORM-01"""
    expected = ["my-text", "my-password", "my-textarea", "my-select"]
    for name in expected:
        element = form_page.find_element(By.NAME, name)
        assert element.is_displayed(), f"{name} is not displayed"
```

```text
tests/test_form_load.py::test_page_title_is_web_form PASSED              [ 33%]
tests/test_form_load.py::test_page_heading_is_visible PASSED             [ 66%]
tests/test_form_load.py::test_all_primary_controls_are_present PASSED    [100%]

============================== 3 passed in 3.71s ===============================
```

### tests/test_text_inputs.py

Boundary value analysis, executed:

```python
"""REQ-FORM-02 to 06 — text-based inputs."""
import pytest
from selenium.webdriver.common.by import By


@pytest.mark.ui
@pytest.mark.boundary
@pytest.mark.parametrize(
    "length",
    [
        pytest.param(0, id="empty"),
        pytest.param(1, id="single_char"),
        pytest.param(50, id="typical"),
        pytest.param(254, id="max_minus_one"),
        pytest.param(255, id="max"),
        pytest.param(256, id="max_plus_one"),
    ],
)
def test_text_input_accepts_boundary_lengths(form_page, length):
    """TC-FORM-004..009 · REQ-FORM-02"""
    value = "a" * length
    field = form_page.find_element(By.NAME, "my-text")
    field.clear()
    field.send_keys(value)
    assert field.get_attribute("value") == value


@pytest.mark.ui
@pytest.mark.parametrize(
    "value",
    [
        pytest.param("plain text", id="plain"),
        pytest.param("with spaces   ", id="trailing_spaces"),
        pytest.param("special!@#$%^&*()", id="special_chars"),
        pytest.param("unicode — naïve café", id="unicode"),
        pytest.param("<script>alert(1)</script>", id="script_tag"),
    ],
)
def test_text_input_accepts_various_characters(form_page, value):
    """TC-FORM-010..014 · REQ-FORM-02"""
    field = form_page.find_element(By.NAME, "my-text")
    field.clear()
    field.send_keys(value)
    assert field.get_attribute("value") == value


@pytest.mark.ui
def test_password_field_masks_input(form_page):
    """TC-FORM-015 · REQ-FORM-03"""
    field = form_page.find_element(By.NAME, "my-password")
    field.send_keys("Secret#123")
    assert field.get_attribute("type") == "password"
    assert field.get_attribute("value") == "Secret#123"


@pytest.mark.ui
def test_textarea_accepts_multiline_input(form_page):
    """TC-FORM-016 · REQ-FORM-04"""
    field = form_page.find_element(By.NAME, "my-textarea")
    field.send_keys("line one\nline two\nline three")
    assert field.get_attribute("value").count("\n") == 2


@pytest.mark.ui
@pytest.mark.negative
def test_disabled_input_is_not_editable(form_page):
    """TC-FORM-017 · REQ-FORM-05"""
    field = form_page.find_element(By.NAME, "my-disabled")
    assert not field.is_enabled()


@pytest.mark.ui
@pytest.mark.negative
def test_readonly_input_retains_original_value(form_page):
    """TC-FORM-018 · REQ-FORM-06"""
    field = form_page.find_element(By.NAME, "my-readonly")
    original = field.get_attribute("value")
    field.send_keys("attempted change")
    assert field.get_attribute("value") == original
```

```text
tests/test_text_inputs.py::test_text_input_accepts_boundary_lengths[empty] PASSED
tests/test_text_inputs.py::test_text_input_accepts_boundary_lengths[single_char] PASSED
tests/test_text_inputs.py::test_text_input_accepts_boundary_lengths[typical] PASSED
tests/test_text_inputs.py::test_text_input_accepts_boundary_lengths[max_minus_one] PASSED
tests/test_text_inputs.py::test_text_input_accepts_boundary_lengths[max] PASSED
tests/test_text_inputs.py::test_text_input_accepts_boundary_lengths[max_plus_one] PASSED
tests/test_text_inputs.py::test_text_input_accepts_various_characters[plain] PASSED
...
tests/test_text_inputs.py::test_readonly_input_retains_original_value PASSED

============================= 15 passed in 21.30s ==============================
```

### tests/test_selection_controls.py

```python
"""REQ-FORM-07/08 — dropdown, checkboxes, radio buttons."""
import pytest
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import Select


@pytest.mark.ui
@pytest.mark.parametrize(
    "option_text, expected_value",
    [("One", "1"), ("Two", "2"), ("Three", "3")],
)
def test_dropdown_options_are_selectable(form_page, option_text, expected_value):
    """TC-FORM-019..021 · REQ-FORM-07"""
    dropdown = Select(form_page.find_element(By.NAME, "my-select"))
    dropdown.select_by_visible_text(option_text)
    assert dropdown.first_selected_option.get_attribute("value") == expected_value


@pytest.mark.ui
def test_dropdown_offers_exactly_three_selectable_options(form_page):
    """TC-FORM-022 · REQ-FORM-07"""
    dropdown = Select(form_page.find_element(By.NAME, "my-select"))
    selectable = [o for o in dropdown.options if o.get_attribute("value")]
    assert len(selectable) == 3


@pytest.mark.ui
def test_checkbox_toggles_independently(form_page):
    """TC-FORM-023 · REQ-FORM-08 — state transition"""
    checked = form_page.find_element(By.ID, "my-check-1")
    unchecked = form_page.find_element(By.ID, "my-check-2")

    assert checked.is_selected()
    assert not unchecked.is_selected()

    unchecked.click()
    assert unchecked.is_selected()
    assert checked.is_selected(), "toggling one checkbox must not affect the other"

    unchecked.click()
    assert not unchecked.is_selected()


@pytest.mark.ui
def test_radio_buttons_are_mutually_exclusive(form_page):
    """TC-FORM-024 · REQ-FORM-08 — state transition"""
    first = form_page.find_element(By.ID, "my-radio-1")
    second = form_page.find_element(By.ID, "my-radio-2")

    assert first.is_selected()
    second.click()
    assert second.is_selected()
    assert not first.is_selected(), "selecting a radio must deselect its sibling"
```

```text
tests/test_selection_controls.py::test_dropdown_options_are_selectable[One-1] PASSED
tests/test_selection_controls.py::test_dropdown_options_are_selectable[Two-2] PASSED
tests/test_selection_controls.py::test_dropdown_options_are_selectable[Three-3] PASSED
tests/test_selection_controls.py::test_dropdown_offers_exactly_three_selectable_options PASSED
tests/test_selection_controls.py::test_checkbox_toggles_independently PASSED
tests/test_selection_controls.py::test_radio_buttons_are_mutually_exclusive PASSED

============================== 6 passed in 9.12s ===============================
```

### tests/test_form_submission.py

```python
"""REQ-FORM-09/10 — submission and confirmation."""
import pytest
from selenium.webdriver.common.by import By
from selenium.webdriver.support import expected_conditions as EC


@pytest.mark.ui
@pytest.mark.smoke
def test_submitting_form_shows_confirmation(form_page, wait):
    """TC-FORM-025 · REQ-FORM-09"""
    wait.until(EC.element_to_be_clickable((By.CSS_SELECTOR, "button"))).click()
    message = wait.until(EC.visibility_of_element_located((By.ID, "message")))
    assert message.text == "Received!"


@pytest.mark.ui
@pytest.mark.regression
def test_submitted_text_appears_in_url(form_page, wait):
    """TC-FORM-026 · REQ-FORM-10"""
    form_page.find_element(By.NAME, "my-text").send_keys("QA automation")
    wait.until(EC.element_to_be_clickable((By.CSS_SELECTOR, "button"))).click()
    wait.until(EC.visibility_of_element_located((By.ID, "message")))
    assert "my-text=QA+automation" in form_page.current_url


@pytest.mark.ui
@pytest.mark.regression
def test_all_entered_values_are_submitted(form_page, wait):
    """TC-FORM-027 · REQ-FORM-10 — integration of every control"""
    from selenium.webdriver.support.ui import Select

    form_page.find_element(By.NAME, "my-text").send_keys("regression")
    form_page.find_element(By.NAME, "my-password").send_keys("Secret123")
    form_page.find_element(By.NAME, "my-textarea").send_keys("notes")
    Select(form_page.find_element(By.NAME, "my-select")).select_by_visible_text("Two")

    wait.until(EC.element_to_be_clickable((By.CSS_SELECTOR, "button"))).click()
    wait.until(EC.visibility_of_element_located((By.ID, "message")))

    url = form_page.current_url
    for expected in ["my-text=regression", "my-password=Secret123",
                     "my-textarea=notes", "my-select=2"]:
        assert expected in url, f"missing {expected} in {url}"


@pytest.mark.ui
@pytest.mark.negative
def test_empty_form_still_submits(form_page, wait):
    """TC-FORM-028 · REQ-FORM-09 — no fields are mandatory"""
    wait.until(EC.element_to_be_clickable((By.CSS_SELECTOR, "button"))).click()
    message = wait.until(EC.visibility_of_element_located((By.ID, "message")))
    assert message.text == "Received!"
```

### Running it

```bash
pytest                      # everything
pytest -m smoke             # build verification only
pytest -m "boundary or negative"
pytest -m ui --durations=5
HEADLESS=0 pytest -m smoke  # watch it run in a visible browser
```

```text
============================= test session starts ==============================
platform darwin -- Python 3.11.9, pytest-8.3.2, pluggy-1.5.0
rootdir: /Users/you/web-form-tests
configfile: pytest.ini
testpaths: tests
collected 28 items

tests/test_form_load.py ...                                              [ 10%]
tests/test_form_submission.py ....                                       [ 25%]
tests/test_selection_controls.py ......                                  [ 46%]
tests/test_text_inputs.py ...............                                [100%]

============================= slowest 5 durations ==============================
2.41s call     tests/test_text_inputs.py::test_text_input_accepts_boundary_lengths[max_plus_one]
2.38s call     tests/test_text_inputs.py::test_text_input_accepts_boundary_lengths[max]
1.62s setup    tests/test_form_load.py::test_page_title_is_web_form
...
============================= 28 passed in 47.16s ==============================
```

## Deliverable 4 — Traceability matrix

`docs/traceability-matrix.md`, in the format from Module 2:

| Req ID | Requirement | Manual cases | Automated tests | Coverage | Status |
|---|---|---|---|---|---|
| REQ-FORM-01 | Page loads with correct title | TC-FORM-001..003 | `test_form_load.py` (3) | Full | Pass |
| REQ-FORM-02 | Text input up to 255 chars | TC-FORM-004..014 | `test_text_inputs.py` (11) | Full | Pass |
| REQ-FORM-03 | Password masked | TC-FORM-015 | `test_password_field_masks_input` | Full | Pass |
| … | … | … | … | … | … |

Fill in all ten requirements, and explicitly flag any with **zero** coverage.

## Deliverable 5 — Test summary report

`docs/test-summary-report.md`:

```markdown
# Test Summary Report — Web Form

## 1. Overview
Feature, build/date tested, environment, who executed.

## 2. Execution summary
| Metric | Value |
|---|---|
| Test cases planned | 28 |
| Executed | 28 |
| Passed | 26 |
| Failed | 1 |
| Blocked | 0 |
| Skipped | 1 |
| Pass rate | 92.9% |
| Automated | 28 of 28 (100%) |
| Total execution time | 47.2 s |

## 3. Requirement coverage
Summarise the RTM: X of 10 requirements fully covered, any gaps named.

## 4. Defects
| ID | Title | Severity | Priority | Status |
|---|---|---|---|---|
Full bug reports (Module 4 template) as an appendix.

## 5. Exit criteria assessment
Each criterion from the test plan, with met/not-met and evidence.

## 6. Risks and recommendation
Outstanding risks, and a clear go / no-go recommendation with justification.

## 7. Lessons learned
What you'd do differently, including anything about testability of the page.
```

## Acceptance criteria

Your project is complete when:

- [ ] `docs/test-plan.md` covers all nine required sections, with at least four risks
- [ ] `docs/test-cases.md` has ≥20 fully-specified cases, including ≥4 negative and ≥4 boundary
- [ ] Every case has a priority and a requirement ID
- [ ] The automated suite has ≥25 collected tests and passes cleanly twice in a row
- [ ] Every test has a docstring citing its test case ID and requirement ID
- [ ] Fixtures handle all setup and teardown — no `driver.quit()` inside a test body
- [ ] No `time.sleep()` anywhere; all waits are explicit
- [ ] Markers work: `pytest -m smoke`, `-m boundary`, `-m negative` each select the right subset
- [ ] Screenshot-on-failure is proven working
- [ ] `requirements.txt` is pinned and the suite runs from a clean venv
- [ ] `docs/traceability-matrix.md` maps all ten requirements
- [ ] `docs/test-summary-report.md` is complete with real numbers from your run
- [ ] `README.md` explains how to install and run in under five commands

## Extension challenges

1. **Cross-browser.** Convert the `driver` fixture into a parametrized fixture
   over Chrome and Firefox, and run the whole suite against both.
2. **HTML report.** `pip install pytest-html`, then
   `pytest --html=reports/report.html --self-contained-html`. Attach it to your
   summary report.
3. **Data-driven from a file.** Move the boundary values into
   `data/text_input_cases.json` and load them in the parametrize decorator, so a
   non-programmer can add cases.
4. **Deliberate defect hunt.** Test the file upload, colour picker, date picker
   and range slider — controls this suite ignores. Write proper bug reports for
   anything that surprises you, including usability issues an automated test
   could never catch.
5. **CI.** Add `.github/workflows/tests.yml` running the suite on every push.
   You'll do this properly in Level 3, but attempting it now shows you what
   Level 3 solves.

Keep this project. A working suite with a real test plan and traceability matrix
is a stronger portfolio piece than any certificate, because it demonstrates both
halves of the job: the thinking and the automation.
