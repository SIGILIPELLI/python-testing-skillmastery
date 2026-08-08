# 02 · Selenium Advanced

Level 1 covered `find_element`, `click`, `send_keys`, and explicit waits — enough
to automate a static form. Real applications are not static. They render content
after an XHR resolves, bury widgets in iframes, open new tabs, throw alerts, and
rebuild the DOM underneath you while a test is mid-click. This module covers the
Selenium APIs that exist specifically to survive that.

Every snippet below was run against the public Selenium practice pages at
`https://www.selenium.dev/selenium/web/` with headless Chrome 150.

## 1. The fixture set-up used throughout

```python
# conftest.py
import pytest
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.support.ui import WebDriverWait


@pytest.fixture(scope="session")
def base_url():
    return "https://www.selenium.dev/selenium/web"


@pytest.fixture
def driver():
    options = Options()
    options.add_argument("--headless=new")
    options.add_argument("--window-size=1400,1000")
    drv = webdriver.Chrome(options=options)
    yield drv
    drv.quit()


@pytest.fixture
def wait(driver):
    return WebDriverWait(driver, timeout=10)
```

`--window-size` is not cosmetic. Headless Chrome defaults to a small viewport,
and elements below the fold can report `is_displayed() == False` or refuse
clicks. A missing window size is one of the most common "passes locally, fails
in CI" causes.

## 2. Waiting for things that do not exist yet

`presence_of_element_located` returns as soon as the node is in the DOM.
`visibility_of_element_located` also requires it to be rendered with non-zero
size. Choosing the wrong one is the single biggest source of flake.

```python
def test_dynamic_element_appears(driver, wait, base_url):
    driver.get(f"{base_url}/dynamic.html")
    driver.find_element(By.ID, "adder").click()
    box = wait.until(EC.visibility_of_element_located((By.ID, "box0")))
    assert "redbox" in box.get_attribute("class")
```

Use **presence** when you only need to read an attribute or check existence.
Use **visibility** before any interaction. Use `element_to_be_clickable` when
the element can be present and visible but still disabled or covered by an
overlay — a modal backdrop that has not finished fading out will pass a
visibility check and still swallow your click.

### Custom expected conditions

`expected_conditions` is just a module of callables that take the driver and
return either a falsy value (keep polling) or a truthy value (which becomes the
`wait.until` return value). Anything matching that contract works:

```python
def element_count_is(locator, expected):
    def predicate(drv):
        found = drv.find_elements(*locator)
        return found if len(found) == expected else False
    return predicate


def test_custom_expected_condition(driver, wait, base_url):
    driver.get(f"{base_url}/dynamic.html")
    driver.find_element(By.ID, "adder").click()
    boxes = wait.until(element_count_is((By.CSS_SELECTOR, ".redbox"), 1))
    assert len(boxes) == 1
```

This is how you wait for "the results table has finished loading all 20 rows"
or "the spinner is gone AND the total is non-zero" — conditions no built-in
covers.

!!! warning "Trap — the flaky wait"
    ```python
    time.sleep(3)                        # ✗ slow when it works, flaky when it doesn't
    driver.implicitly_wait(10)           # ✗ interacts badly with explicit waits
    wait.until(EC.presence_of_element_located(BUTTON)).click()   # ✗ present ≠ clickable
    ```
    `time.sleep` guesses. An implicit wait applies to *every* `find_element`
    and, when mixed with `WebDriverWait`, can multiply timeouts unpredictably —
    pick one strategy (explicit) and never set an implicit wait at all.

Always pass `message=` so a timeout tells you what was expected:

```python
short = WebDriverWait(driver, timeout=2)
short.until(EC.visibility_of_element_located((By.ID, "never-exists")),
            message="box0 never became visible")
```

```text
selenium.common.exceptions.TimeoutException: Message: box0 never became visible
```

Without `message`, a `TimeoutException` reports an empty message and you are
left reading a chromedriver stack trace to guess which locator failed.

## 3. StaleElementReferenceException

A `WebElement` is a *handle to a specific DOM node*, not a live query. Re-render
that part of the page and the handle dies.

```python
def test_stale_element_reference(driver, wait, base_url):
    driver.get(f"{base_url}/dynamic.html")
    driver.find_element(By.ID, "adder").click()
    box = wait.until(EC.presence_of_element_located((By.ID, "box0")))
    assert box.get_attribute("class") == "redbox"
    driver.refresh()  # the node `box` points at no longer exists
    with pytest.raises(StaleElementReferenceException):
        box.get_attribute("class")
```

The fix is never "retry the whole test". It is: **do not hold an element across
anything that can re-render**. Re-find inside the wait predicate instead.

```python
def test_retry_on_stale(driver, wait, base_url):
    driver.get(f"{base_url}/dynamic.html")
    driver.find_element(By.ID, "adder").click()
    wait.until(EC.presence_of_element_located((By.ID, "box0")))
    driver.refresh()
    driver.find_element(By.ID, "adder").click()
    # re-find on every poll — never cache the element across a reload
    css = wait.until(lambda d: d.find_element(By.ID, "box0").get_attribute("class"))
    assert css == "redbox"
```

This is also the argument for storing **locator tuples** in page objects rather
than resolved elements: a tuple can be re-resolved, a `WebElement` cannot.

## 4. Frames, windows, and tabs

The driver has exactly one "focus". Elements outside it are invisible to
`find_element` — it does not raise a helpful error, they simply are not found.

```python
def test_iframe_switching(driver, base_url):
    driver.get(f"{base_url}/iframes.html")
    assert driver.find_elements(By.ID, "imageButton") == []   # inside the frame
    driver.switch_to.frame(driver.find_element(By.ID, "iframe1"))
    assert driver.find_element(By.ID, "imageButton").is_displayed()
    driver.switch_to.default_content()                        # always switch back
    assert driver.find_elements(By.ID, "imageButton") == []
```

Forgetting `switch_to.default_content()` leaves every later step in the test
scoped to the frame, producing "element exists in the browser but Selenium
cannot see it" — a bug that looks like a locator problem and is not.

```python
def test_new_window(driver, base_url):
    driver.get(f"{base_url}/xhtmlTest.html")
    original = driver.current_window_handle
    driver.switch_to.new_window("tab")
    driver.get(f"{base_url}/formPage.html")
    assert driver.title == "We Leave From Here"
    driver.close()                          # closes the new tab only
    driver.switch_to.window(original)       # focus does NOT return automatically
    assert driver.title == "XHTML Test Page"
```

`driver.close()` closes the focused window; `driver.quit()` ends the whole
session. After `close()`, the driver has **no** valid focus until you switch
explicitly.

## 5. Alerts

Native browser dialogs are not DOM. `find_element` will never reach them, and
an unhandled alert makes the *next* unrelated command fail with
`UnexpectedAlertPresentException`.

```python
def test_alerts(driver, wait, base_url):
    driver.get(f"{base_url}/alerts.html")
    driver.find_element(By.ID, "prompt").click()
    alert = wait.until(EC.alert_is_present())
    assert alert.text == "Enter something"
    alert.send_keys("automated")
    alert.accept()          # or .dismiss() for cancel
    assert driver.find_element(By.ID, "text").text == "automated"
```

## 6. ActionChains — hover, drag, modifier keys

```python
def test_action_chains_hover(driver, base_url):
    driver.get(f"{base_url}/mouse_interaction.html")
    hover = driver.find_element(By.ID, "hover")
    ActionChains(driver).move_to_element(hover).perform()
    assert driver.find_element(By.ID, "move-status").text == "hovered"


def test_action_chains_keys(driver, base_url):
    driver.get(f"{base_url}/web-form.html")
    field = driver.find_element(By.NAME, "my-text")
    ActionChains(driver).click(field).send_keys("hello world").perform()
    ActionChains(driver).key_down(Keys.COMMAND).send_keys("a").key_up(Keys.COMMAND).perform()
    ActionChains(driver).send_keys("replaced").perform()
    assert field.get_attribute("value") == "replaced"
```

Two rules: nothing happens until `.perform()`, and every `key_down` needs a
matching `key_up` — a leaked modifier turns every later keystroke in the session
into a shortcut. Note `Keys.COMMAND` is macOS; use `Keys.CONTROL` elsewhere, so
parameterise it if your suite runs on both.

## 7. The JavaScript escape hatch

```python
def test_javascript_executor(driver, base_url):
    driver.get(f"{base_url}/web-form.html")
    height = driver.execute_script("return document.body.scrollHeight;")
    assert height > 0
    field = driver.find_element(By.NAME, "my-text")
    driver.execute_script("arguments[0].value = arguments[1];", field, "set by JS")
    assert field.get_attribute("value") == "set by JS"
```

Legitimate uses: scrolling an element into view, reading computed state, waiting
on an app-specific JS flag. Illegitimate use: `execute_script("arguments[0].click()")`
to "fix" a click Selenium refuses. Selenium refused because a real user could not
click it — an overlay was covering it, or it was off-screen. A JS click makes
the test pass and ships the bug.

## 8. Real run

```text
test_adv.py::test_dynamic_element_appears PASSED                         [  9%]
test_adv.py::test_custom_expected_condition PASSED                       [ 18%]
test_adv.py::test_stale_element_reference PASSED                         [ 27%]
test_adv.py::test_retry_on_stale PASSED                                  [ 36%]
test_adv.py::test_iframe_switching PASSED                                [ 45%]
test_adv.py::test_new_window PASSED                                      [ 54%]
test_adv.py::test_javascript_executor PASSED                             [ 63%]
test_adv.py::test_action_chains_hover PASSED                             [ 72%]
test_adv.py::test_action_chains_keys PASSED                              [ 81%]
test_adv.py::test_alerts PASSED                                          [ 90%]
test_adv.py::test_timeout_message PASSED                                 [100%]

============================= 11 passed in 23.99s ==============================
```

## Cheat sheet

| Need | API |
|---|---|
| Element in DOM, may be hidden | `EC.presence_of_element_located` |
| Element rendered, about to read it | `EC.visibility_of_element_located` |
| About to click it | `EC.element_to_be_clickable` |
| Anything else | Custom predicate passed to `wait.until` |
| Timeout that names the failure | `wait.until(cond, message="...")` |
| Enter an iframe / leave it | `switch_to.frame(el)` / `switch_to.default_content()` |
| New tab | `switch_to.new_window("tab")`, then `switch_to.window(handle)` |
| Native dialog | `EC.alert_is_present()`, then `.accept()` / `.dismiss()` |
| Hover, drag, modifier keys | `ActionChains(driver)...perform()` |
| Read state JS knows and Selenium doesn't | `driver.execute_script("return ...")` |
| Stale element | Re-find it; never cache across a re-render |

## Exercise

1. Write a custom expected condition `text_is_not_empty(locator)` and use it on
   `mouse_interaction.html` instead of asserting immediately after the hover.
   Confirm it removes the need for any sleep.
2. Reproduce `StaleElementReferenceException` on `dynamic.html`, then fix it two
   ways — re-finding the element, and re-finding inside the `wait.until`
   predicate. Explain in a comment which one belongs in a page object.
3. Take a test that uses `switch_to.frame` and deliberately delete the
   `switch_to.default_content()` call. Add a second assertion after it and record
   the exact exception you get, so you recognise it next time.
4. Write a test on `alerts.html` that clicks the prompt but never accepts it,
   then tries `driver.find_element`. Record the exception name and message.
5. Replace one `element_to_be_clickable` wait with `execute_script("arguments[0].click()")`
   and describe one real bug that change would hide from you.
