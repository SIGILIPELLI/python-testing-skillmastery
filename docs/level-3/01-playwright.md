# 01 · Playwright for Python

Playwright is a newer browser automation library from Microsoft. It talks to
Chromium, Firefox, and WebKit over a single protocol instead of driving each
browser's native driver the way Selenium does. The practical payoff: faster
execution, built-in auto-waiting, and — notably for this course — it runs
headlessly in sandboxed environments where Selenium's Chrome driver often
refuses to launch. Every example below was actually executed headlessly while
writing this module.

## 1. Install

```bash
pip install pytest-playwright
playwright install chromium
```

`playwright install` downloads a matched browser binary — it is not the
system Chrome, so version drift between your machine and CI mostly disappears.

## 2. Your first test

```python
from playwright.sync_api import sync_playwright

def test_example():
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        page = browser.new_page()
        page.goto("https://example.com")
        assert page.title() == "Example Domain"
        browser.close()
```

```text
$ pytest test_pw.py -v
test_pw.py::test_example PASSED
```

That ran, unmodified, in this environment — no `--no-sandbox` flags, no
display server, no workarounds. If you hit `BrowserType.launch: Executable
doesn't exist`, you skipped `playwright install chromium`.

## 3. Fixtures: browser once, page per test

Launching a browser process is expensive; opening a new page is cheap. Scope
accordingly, exactly like you scoped the Selenium `driver` fixture in Level 1
— except now the shared thing is the browser, and the per-test thing is a
*context* (an isolated, cookie-free session) plus a page inside it.

```python
# conftest.py
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
    context = browser.new_context()
    pg = context.new_page()
    yield pg
    context.close()
```

Every test gets a brand-new context — no leaked cookies or localStorage
between tests, without you writing any cleanup code. This is the built-in
version of the "clear cookies between tests" fixture you had to hand-roll for
Selenium.

## 4. Locators and auto-waiting

Selenium's `find_element` looks *once*, immediately, and throws if the element
isn't there yet — which is why Level 1 leaned so hard on `WebDriverWait`.
Playwright's `Locator` is lazy: it doesn't query the DOM until you act on it,
and then it retries automatically until the element is actionable or a
timeout elapses.

```python
def test_text_input_and_submit(page):
    page.goto("https://www.selenium.dev/selenium/web/web-form.html")
    page.fill("#my-text-id", "hello playwright")
    page.click("button[type=submit]")
    page.wait_for_selector("#message")
    assert page.inner_text("#message") == "Received!"
```

```text
test_web_form.py::test_text_input_and_submit PASSED
```

No explicit `WebDriverWait(driver, 10).until(...)` needed for the click or the
fill — Playwright already waited for the button to be visible, enabled, and
stable before clicking it. You still need `wait_for_selector` for content that
appears *after* an action, same as an explicit wait in Selenium.

## 5. Auto-waiting for state changes

```python
def test_auto_waiting_for_hidden_element(page):
    page.goto("https://www.selenium.dev/selenium/web/dynamic.html")
    page.click("#adder")
    locator = page.locator("#box0")
    locator.wait_for(state="visible")
    assert locator.is_visible()
```

```text
test_web_form.py::test_auto_waiting_for_hidden_element PASSED
```

`#box0` doesn't exist in the DOM until the button adds it via JavaScript.
`locator.wait_for(state="visible")` polls for both existence *and* visibility
— a `NoSuchElementException` and a `TimeoutException` from Selenium collapsed
into a single wait call.

## 6. Testing-specific traps

**Trap 1 — `page.click()` succeeding on a covered element.** Playwright
checks that the target is not obscured by another element before clicking,
and raises a timeout with a screenshot-style trace if it stays covered. This
catches overlay/modal bugs that Selenium's `ElementClickInterceptedException`
also catches, but Playwright's error message names the *specific* intercepting
element, which saves real debugging time.

**Trap 2 — strict mode locator ambiguity.** `page.locator("button")` matching
three buttons raises immediately instead of silently acting on the first one
(Selenium's `find_element` behavior). Prefer `get_by_role`, `get_by_text`, or
`get_by_test_id` over raw CSS to keep locators unambiguous and readable:

```python
page.get_by_role("button", name="Submit").click()
```

**Trap 3 — mixing sync and async APIs.** Playwright ships both a
`sync_api` and an `async_api`. Importing `sync_playwright` inside an `async
def` test (or vice versa) raises a confusing
`Fixture "page" called from event loop` type error. Stick to `sync_api` unless
your whole suite is already `pytest-asyncio` based — mixing the two per file
is a common source of "works locally, fails in CI" flakiness.

**Trap 4 — session-scoped browser leaking state.** Because `browser` is
session-scoped for speed, if you accidentally create pages directly on
`browser` (skipping `new_context()`) they *do* share cookies across tests —
identical to reusing one Selenium `driver` across the whole suite. Always go
through a fresh `context` per test.

## Cheat sheet

| Selenium | Playwright | Notes |
|---|---|---|
| `webdriver.Chrome()` | `p.chromium.launch()` | Playwright manages the binary itself |
| `driver.find_element(By.ID, "x")` | `page.locator("#x")` | Playwright locator is lazy, re-queries each action |
| `WebDriverWait(...).until(EC.visibility_of_element_located(...))` | `locator.wait_for(state="visible")` | Auto-waiting removes most explicit waits |
| `driver.get_cookie(...)` | `context.cookies()` | Scoped to a context, not global |
| One `driver` per suite (careful cleanup) | One `context` per test (cheap, isolated) | Contexts replace incognito-window tricks |
| `ElementClickInterceptedException` | Timeout naming the blocking element | Better diagnostics out of the box |
| N/A | `page.screenshot(path=...)` | Built-in, no extra library |
| N/A | Trace viewer (`context.tracing`) | Records a full timeline + screenshots per test |

## Exercise

Using `https://www.selenium.dev/selenium/web/web-form.html` and
`https://www.selenium.dev/selenium/web/dynamic.html`:

1. Write the `conftest.py` above with session-scoped `browser` and
   function-scoped `page` fixtures.
2. Write five tests: page title, filling and submitting the text input,
   selecting a dropdown option with `page.select_option`, checking the
   checkbox with `page.check`, and the dynamic "add box" scenario from
   section 5.
3. Rewrite one test using `get_by_role` / `get_by_label` instead of CSS
   selectors and note which version reads more like the user's own actions.
4. Deliberately call `page.locator("button")` on the web-form page (it has
   more than one button) with an action like `.click()`, capture the strict
   mode violation error, and fix it by scoping the locator more precisely.
5. Add `context.tracing.start(screenshots=True, snapshots=True)` before your
   test and `context.tracing.stop(path="trace.zip")` after; open the trace
   with `playwright show-trace trace.zip` and describe what it shows that a
   pytest failure log doesn't.
