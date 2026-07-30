# 09 · Selenium WebDriver Basics with Python

Selenium WebDriver drives a real browser the way a user does — clicking,
typing, navigating. This is how the manual test cases you wrote in Module 2
become automated regression tests.

## 1. Install and verify

```bash
pip install selenium webdriver-manager
```

Since Selenium 4.6, drivers are downloaded and managed automatically by
**Selenium Manager**, so a bare setup usually just works:

```python
from selenium import webdriver

driver = webdriver.Chrome()
driver.get("https://example.com")
print(driver.title)
driver.quit()
```

```text
Example Domain
```

If your environment blocks the automatic download, `webdriver-manager` is the
fallback:

```python
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from webdriver_manager.chrome import ChromeDriverManager

driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()))
```

!!! warning "Always quit the driver"
    Every `webdriver.Chrome()` starts a real browser process. A test that
    crashes before `driver.quit()` leaves it running. Ten test runs later you
    have ten orphaned browsers eating memory. Section 6 fixes this properly with
    a fixture.

## 2. Browser options

You will nearly always want headless mode in CI and a fixed window size for
reproducibility.

```python
from selenium import webdriver

options = webdriver.ChromeOptions()
options.add_argument("--headless=new")       # no visible window
options.add_argument("--window-size=1920,1080")
options.add_argument("--disable-gpu")
options.add_argument("--no-sandbox")         # required in many CI containers
options.add_argument("--disable-dev-shm-usage")

driver = webdriver.Chrome(options=options)
driver.get("https://example.com")
print(driver.title)
print(driver.current_url)
driver.quit()
```

```text
Example Domain
https://example.com/
```

Fixing the window size matters: responsive sites hide elements below certain
widths, so a suite that passes on your 27-inch monitor can fail on a CI
container's default 800×600.

## 3. Locators — finding elements

Selenium finds elements with a **locator strategy** and a value.

```python
from selenium.webdriver.common.by import By

driver.find_element(By.ID, "email")
driver.find_element(By.NAME, "password")
driver.find_element(By.CLASS_NAME, "btn-primary")
driver.find_element(By.TAG_NAME, "h1")
driver.find_element(By.LINK_TEXT, "Forgot password?")
driver.find_element(By.PARTIAL_LINK_TEXT, "Forgot")
driver.find_element(By.CSS_SELECTOR, "input[type='email']")
driver.find_element(By.XPATH, "//button[text()='Sign in']")

driver.find_elements(By.CSS_SELECTOR, ".product-card")  # returns a list
```

`find_element` raises `NoSuchElementException` if nothing matches;
`find_elements` returns an empty list — which is what you use to assert absence.

### Choosing a locator

Locator choice is the single biggest factor in whether your suite is stable or
maintenance hell. Prefer, in this order:

| Priority | Strategy | Example | Why |
|---|---|---|---|
| 1 | Dedicated test attribute | `[data-testid='login-submit']` | Never changed by a redesign; ask developers to add these |
| 2 | `ID` | `By.ID, "email"` | Unique and fast — if it's stable and not generated |
| 3 | `NAME` | `By.NAME, "password"` | Stable on form fields |
| 4 | CSS selector on semantic attributes | `input[type='email'][required]` | Readable, fast |
| 5 | Link text | `By.LINK_TEXT, "Sign out"` | Readable, but breaks on translation |
| 6 | XPath with text or relationships | `//label[text()='Email']/following-sibling::input` | Powerful, use when nothing else works |
| ✗ | Absolute XPath | `/html/body/div[3]/div/form/div[2]/input` | Breaks on any layout change. Never. |
| ✗ | Generated class names | `.css-1x7k9j2` | Regenerated on every build |

### CSS vs XPath

```python
# CSS — usually shorter and faster
driver.find_element(By.CSS_SELECTOR, "#login-form input[name='email']")
driver.find_element(By.CSS_SELECTOR, ".product-card:nth-of-type(2) .price")
driver.find_element(By.CSS_SELECTOR, "button[data-testid='submit']")

# XPath — can match on text and traverse upwards, which CSS cannot
driver.find_element(By.XPATH, "//button[normalize-space()='Add to cart']")
driver.find_element(By.XPATH, "//td[text()='SKU-101']/../td[@class='price']")
driver.find_element(By.XPATH, "//div[contains(@class,'alert') and contains(.,'invalid')]")
```

Use CSS by default; reach for XPath when you need to match visible text or
navigate to a parent element.

## 4. Interacting with elements

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys

options = webdriver.ChromeOptions()
options.add_argument("--headless=new")
driver = webdriver.Chrome(options=options)

try:
    driver.get("https://www.selenium.dev/selenium/web/web-form.html")

    print("Title:", driver.title)

    text_box = driver.find_element(By.NAME, "my-text")
    text_box.clear()
    text_box.send_keys("QA automation")

    password = driver.find_element(By.NAME, "my-password")
    password.send_keys("Secret#123")

    textarea = driver.find_element(By.NAME, "my-textarea")
    textarea.send_keys("Multi-line\ninput works too")

    checkbox = driver.find_element(By.ID, "my-check-2")
    checkbox.click()
    print("Checkbox selected:", checkbox.is_selected())

    submit = driver.find_element(By.CSS_SELECTOR, "button")
    submit.click()

    message = driver.find_element(By.ID, "message")
    print("Message:", message.text)
    print("Current URL:", driver.current_url)
finally:
    driver.quit()
```

```text
Title: Web form
Checkbox selected: True
Message: Received!
Current URL: https://www.selenium.dev/selenium/web/web-form.html?my-text=QA+automation&...
```

| Method | Does |
|---|---|
| `.click()` | Click the element |
| `.send_keys(text)` | Type into it |
| `.clear()` | Empty a text field |
| `.submit()` | Submit the containing form |
| `.text` | Visible text content |
| `.get_attribute("value")` | An attribute — use this for input contents |
| `.is_displayed()` | Visible to the user |
| `.is_enabled()` | Not disabled |
| `.is_selected()` | Checkbox/radio state |

!!! tip "text vs get_attribute('value')"
    For `<input>` elements, `.text` is almost always empty — the typed content
    lives in the `value` attribute. Asserting `element.text == "QA automation"`
    on an input is a very common first-week mistake.

### Dropdowns

```python
from selenium.webdriver.support.ui import Select

dropdown = Select(driver.find_element(By.NAME, "my-select"))
dropdown.select_by_visible_text("Two")
print("Selected:", dropdown.first_selected_option.text)
print("Options:", [o.text for o in dropdown.options])
```

```text
Selected: Two
Options: ['Open this select menu', 'One', 'Two', 'Three']
```

`Select` only works on real `<select>` elements. Custom JavaScript dropdowns are
a `div` soup and need ordinary clicks plus waits.

### Keyboard input

```python
search = driver.find_element(By.NAME, "q")
search.send_keys("trail running shoes")
search.send_keys(Keys.ENTER)

# select-all then replace
field.send_keys(Keys.CONTROL, "a")
field.send_keys("replacement text")
```

## 5. Waits — the most important topic here

Web pages load asynchronously. Your script runs in milliseconds; the page takes
hundreds. Nearly every flaky UI test traces back to a missing or wrong wait.

### Never use time.sleep

```python
import time
time.sleep(5)   # ✗ wrong
```

It's slow when the page is fast, and still insufficient when the page is slow.
Multiply by 200 tests and you have a suite nobody runs.

### Explicit waits — the correct default

Wait for a **condition**, and continue the instant it's true.

```python
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.common.by import By

wait = WebDriverWait(driver, timeout=10)

element = wait.until(EC.visibility_of_element_located((By.ID, "message")))
print(element.text)

button = wait.until(EC.element_to_be_clickable((By.CSS_SELECTOR, "button[type='submit']")))
button.click()

wait.until(EC.text_to_be_present_in_element((By.ID, "status"), "Complete"))
wait.until(EC.invisibility_of_element_located((By.CLASS_NAME, "spinner")))
wait.until(EC.url_contains("/dashboard"))
wait.until(EC.title_is("Dashboard"))
wait.until(EC.alert_is_present())
```

| Expected condition | Waits until |
|---|---|
| `presence_of_element_located` | In the DOM (may still be invisible) |
| `visibility_of_element_located` | In the DOM **and** visible |
| `element_to_be_clickable` | Visible **and** enabled |
| `invisibility_of_element_located` | Gone or hidden — for spinners |
| `text_to_be_present_in_element` | Element contains the text |
| `url_contains` / `url_to_be` | Navigation happened |
| `alert_is_present` | A JS alert appeared |
| `number_of_windows_to_be` | A new tab opened |

The distinction between *presence* and *visibility* matters: an element can be
in the DOM but hidden behind a modal, and clicking it raises
`ElementClickInterceptedException`. Wait for `element_to_be_clickable` before
any click.

### Implicit wait

A global "retry finding elements for up to N seconds":

```python
driver.implicitly_wait(10)
```

It applies to every `find_element`. It's a blunt instrument — it doesn't help
with "wait for the spinner to disappear" or "wait for the text to change".

!!! warning "Never mix implicit and explicit waits"
    Combining them produces unpredictable, compounded timeouts — a documented
    Selenium pitfall. Pick explicit waits and leave the implicit wait at its
    default of zero.

## 6. Selenium + pytest — the fixture

Put the driver in a fixture and teardown becomes automatic.

`conftest.py`:

```python
import pytest
from selenium import webdriver
from selenium.webdriver.support.ui import WebDriverWait


@pytest.fixture(scope="session")
def base_url():
    import os
    return os.environ.get("APP_BASE_URL", "https://www.selenium.dev/selenium/web")


@pytest.fixture
def driver():
    """A fresh browser per test, always closed afterwards."""
    options = webdriver.ChromeOptions()
    options.add_argument("--headless=new")
    options.add_argument("--window-size=1920,1080")
    options.add_argument("--disable-gpu")

    driver = webdriver.Chrome(options=options)
    driver.set_page_load_timeout(30)

    yield driver

    driver.quit()


@pytest.fixture
def wait(driver):
    return WebDriverWait(driver, timeout=10)
```

`tests/test_web_form.py`:

```python
import pytest
from selenium.webdriver.common.by import By
from selenium.webdriver.support import expected_conditions as EC


@pytest.mark.ui
@pytest.mark.smoke
def test_web_form_page_loads(driver, base_url):
    driver.get(f"{base_url}/web-form.html")
    assert driver.title == "Web form"
    assert driver.find_element(By.TAG_NAME, "h1").text == "Web form"


@pytest.mark.ui
def test_submitting_form_shows_confirmation(driver, wait, base_url):
    # Arrange
    driver.get(f"{base_url}/web-form.html")

    # Act
    driver.find_element(By.NAME, "my-text").send_keys("QA automation")
    wait.until(EC.element_to_be_clickable((By.CSS_SELECTOR, "button"))).click()

    # Assert
    message = wait.until(EC.visibility_of_element_located((By.ID, "message")))
    assert message.text == "Received!"
    assert "my-text=QA+automation" in driver.current_url


@pytest.mark.ui
def test_disabled_input_cannot_be_edited(driver, base_url):
    driver.get(f"{base_url}/web-form.html")
    disabled_field = driver.find_element(By.NAME, "my-disabled")
    assert not disabled_field.is_enabled()


@pytest.mark.ui
@pytest.mark.parametrize(
    "option_text, expected_value",
    [("One", "1"), ("Two", "2"), ("Three", "3")],
)
def test_dropdown_selection(driver, base_url, option_text, expected_value):
    from selenium.webdriver.support.ui import Select

    driver.get(f"{base_url}/web-form.html")
    dropdown = Select(driver.find_element(By.NAME, "my-select"))
    dropdown.select_by_visible_text(option_text)
    assert dropdown.first_selected_option.get_attribute("value") == expected_value
```

```bash
pytest tests/test_web_form.py -v -m ui
```

```text
collected 6 items

tests/test_web_form.py::test_web_form_page_loads PASSED                  [ 16%]
tests/test_web_form.py::test_submitting_form_shows_confirmation PASSED   [ 33%]
tests/test_web_form.py::test_disabled_input_cannot_be_edited PASSED      [ 50%]
tests/test_web_form.py::test_dropdown_selection[One-1] PASSED            [ 66%]
tests/test_web_form.py::test_dropdown_selection[Two-2] PASSED            [ 83%]
tests/test_web_form.py::test_dropdown_selection[Three-3] PASSED          [100%]

========================== 6 passed in 8.42s ===================================
```

Six automated UI tests, each with a clean browser, each torn down whether it
passes or fails.

## 7. Screenshot on failure

The first thing anyone asks about a failed UI test is "what did the screen look
like?" Capture it automatically with a `conftest.py` hook:

```python
import pytest
from pathlib import Path


@pytest.hookimpl(hookwrapper=True, tryfirst=True)
def pytest_runtest_makereport(item, call):
    """Record each phase's result on the test item."""
    outcome = yield
    report = outcome.get_result()
    setattr(item, f"report_{report.when}", report)


@pytest.fixture
def driver(request):
    from selenium import webdriver

    options = webdriver.ChromeOptions()
    options.add_argument("--headless=new")
    options.add_argument("--window-size=1920,1080")
    driver = webdriver.Chrome(options=options)

    yield driver

    report = getattr(request.node, "report_call", None)
    if report is not None and report.failed:
        screenshots = Path("reports/screenshots")
        screenshots.mkdir(parents=True, exist_ok=True)
        path = screenshots / f"{request.node.name}.png"
        driver.save_screenshot(str(path))
        print(f"\n[screenshot] {path}")

    driver.quit()
```

```text
tests/test_web_form.py::test_submitting_form_shows_confirmation FAILED
[screenshot] reports/screenshots/test_submitting_form_shows_confirmation.png
```

Also useful on failure: `driver.page_source` (dump the HTML) and
`driver.get_log("browser")` (JavaScript console errors — often the actual cause).

## 8. Common exceptions and what they mean

| Exception | Cause | Fix |
|---|---|---|
| `NoSuchElementException` | Locator wrong, or element not there **yet** | Verify the locator in DevTools; add an explicit wait |
| `TimeoutException` | Condition never became true within the timeout | Is the condition right? Is the page genuinely slow? Is the app broken — a real bug? |
| `ElementNotInteractableException` | Element exists but is hidden or zero-sized | Wait for visibility; scroll into view |
| `ElementClickInterceptedException` | Something (modal, cookie banner, sticky header) is on top | Dismiss the overlay; wait for it to disappear |
| `StaleElementReferenceException` | The DOM re-rendered after you found the element | Re-find the element after the action that changed the page |
| `WebDriverException: session not created` | Browser/driver version mismatch | Update the browser, or let Selenium Manager handle it |

`StaleElementReferenceException` is the one that confuses newcomers most. An
element reference points at a specific DOM node; if a framework like React
re-renders that part of the page, your reference is stale even though a visually
identical element exists. Fix by finding the element again after the update, not
by storing it before.

!!! info "A TimeoutException can be a real defect"
    Don't reflexively increase the timeout. If the page genuinely takes 25
    seconds to render, that is a **performance defect worth reporting** — not
    something to hide behind `timeout=30`.

## 9. Navigation and windows

```python
driver.get("https://example.com")
driver.back()
driver.forward()
driver.refresh()

print(driver.current_url)
print(driver.title)
print(driver.page_source[:200])

# multiple tabs
original = driver.current_window_handle
driver.switch_to.new_window("tab")
driver.get("https://example.org")
driver.close()
driver.switch_to.window(original)

# iframes
driver.switch_to.frame("iframe-name")
driver.find_element(By.ID, "inside-frame").click()
driver.switch_to.default_content()

# JavaScript alerts
alert = driver.switch_to.alert
print(alert.text)
alert.accept()      # or alert.dismiss()
```

Elements inside an `<iframe>` are invisible to `find_element` until you switch
into the frame — a very common source of "the element is clearly there but
Selenium can't find it."

## 10. Turning a manual test case into an automated one

Take `TC-LOGIN-004` from Module 2 — *login fails with unregistered email* — and
map it directly:

| Test case field | Code |
|---|---|
| Preconditions | The `driver` fixture (fresh browser) |
| Test Data | Parameters or a fixture |
| Steps 1–4 | `driver.get(...)`, `send_keys(...)`, `click()` |
| Expected Result | The `assert` statements |
| Actual Result | pytest's failure output |
| Status | PASSED / FAILED |

```python
@pytest.mark.ui
@pytest.mark.regression
def test_login_fails_with_unregistered_email(driver, wait, base_url):
    """TC-LOGIN-004 · REQ-AUTH-02"""
    # Step 1 — navigate
    driver.get(f"{base_url}/login")

    # Steps 2-3 — enter credentials
    driver.find_element(By.ID, "email").send_keys("qa.unknown@test.com")
    driver.find_element(By.ID, "password").send_keys("Valid#123")

    # Step 4 — submit
    wait.until(EC.element_to_be_clickable((By.ID, "signin"))).click()

    # Step 5 — expected results, all of them
    error = wait.until(EC.visibility_of_element_located((By.CSS_SELECTOR, ".field-error")))
    assert error.text == "No account found for this email"
    assert driver.current_url.endswith("/login")
    assert driver.get_cookie("session_id") is None
```

Note the docstring carrying the test case ID and requirement ID — that's your
traceability matrix surviving into the code.

## Exercise

Using the Selenium practice page at
`https://www.selenium.dev/selenium/web/web-form.html`:

1. Build a `conftest.py` with a `driver` fixture (headless, fixed window size,
   guaranteed `quit()`), a `wait` fixture, and a `base_url` fixture reading an
   environment variable with a default.
2. Write **eight UI tests** covering: page title, heading text, text input,
   password input, textarea, the dropdown (`Select`), the checkbox, and the
   disabled input. Mark them all `@pytest.mark.ui`, and the first two also
   `@pytest.mark.smoke`.
3. Use `@pytest.mark.parametrize` to test the dropdown with all three options
   and the text input with five values from your equivalence partitions (empty,
   single char, normal, 255 chars, special characters).
4. Add the screenshot-on-failure hook from section 7 and prove it works by
   asserting the title equals something wrong. Confirm the PNG appears in
   `reports/screenshots/`.
5. Deliberately trigger three different exceptions —
   `NoSuchElementException`, `TimeoutException`, and
   `ElementClickInterceptedException` — capture each traceback, and write one
   sentence explaining the fix for each.
6. Replace one explicit wait with `time.sleep(5)`, measure the suite runtime
   difference with `--durations=10`, then put the wait back.
