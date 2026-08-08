# 09 · Cross-Browser Testing

"Works on my machine" has a browser-shaped variant: works in Chrome. Date inputs
render differently in Firefox, Safari treats `position: sticky` inside a scroll
container differently, and Edge's autofill drops an element your click was aimed
at. Cross-browser testing is how you find those before a customer does.

It is also the fastest way to double, triple, or quadruple your suite's runtime
for very little extra signal — so most of this module is about being selective.

## 1. Parametrize the driver fixture

The whole mechanism is one fixture with `params`.

```python
# conftest.py
import pytest
from selenium import webdriver


@pytest.fixture(params=["chrome", "firefox"])
def browser_name(request):
    return request.param


@pytest.fixture
def driver(browser_name):
    if browser_name == "chrome":
        options = webdriver.ChromeOptions()
        options.add_argument("--headless=new")
        options.add_argument("--window-size=1440,900")
        driver = webdriver.Chrome(options=options)
    elif browser_name == "firefox":
        options = webdriver.FirefoxOptions()
        options.add_argument("--headless")
        driver = webdriver.Firefox(options=options)
    else:
        raise ValueError(f"unsupported browser: {browser_name}")

    driver.implicitly_wait(0)          # explicit waits only — see module 02
    yield driver
    driver.quit()
```

Every test that requests `driver` now runs once per browser, with no change to
the test itself:

```python
def test_form_submits(driver, base_url):
    page = WebFormPage(driver, base_url).load()
    page.submit_text("cross-browser")
    assert page.confirmation_message() == "Received!"
```

```text
test_form.py::test_form_submits[chrome] PASSED                           [ 50%]
test_form.py::test_form_submits[firefox] PASSED                          [100%]
```

The `[chrome]` / `[firefox]` suffix is what makes the report readable and lets
you re-run one browser with `-k firefox`.

Since Selenium 4.6, **Selenium Manager downloads the matching driver binary
automatically** — no `webdriver-manager`, no `chromedriver` on your PATH. If you
still have `webdriver_manager` imports in an older suite, deleting them usually
fixes more problems than it causes.

## 2. Make the browser a command-line option

Running every browser on every local run is wasteful. Make it selectable, with a
fast default (this builds on `pytest_addoption` from module 06).

```python
# conftest.py
def pytest_addoption(parser):
    parser.addoption("--browsers", default="chrome",
                     help="comma-separated: chrome,firefox,edge")


def pytest_generate_tests(metafunc):
    if "browser_name" in metafunc.fixturenames:
        browsers = metafunc.config.getoption("--browsers").split(",")
        metafunc.parametrize("browser_name", browsers, scope="session")
```

```bash
pytest                                   # chrome only — the fast local loop
pytest --browsers chrome,firefox         # pre-merge
pytest --browsers chrome,firefox,edge -n 6   # nightly
```

`pytest_generate_tests` is the hook for parametrizing based on runtime
information. `params=[...]` on the fixture can't read a CLI flag; this can.

## 3. Selenium Grid and cloud providers

Running four browsers locally means four browser installs on every machine and no
Safari unless you're on a Mac. A Grid centralises them.

```bash
docker run -d -p 4444:4444 -p 7900:7900 --shm-size=2g selenium/standalone-chrome:latest
```

```python
@pytest.fixture
def driver(browser_name, request):
    grid_url = request.config.getoption("--grid-url")
    if grid_url:
        options = {"chrome": webdriver.ChromeOptions,
                   "firefox": webdriver.FirefoxOptions,
                   "edge": webdriver.EdgeOptions}[browser_name]()
        driver = webdriver.Remote(command_executor=grid_url, options=options)
    else:
        driver = _local_driver(browser_name)
    yield driver
    driver.quit()
```

Note `--shm-size=2g` in the docker command. The default 64 MB of shared memory
makes Chrome crash mid-session with `session deleted because of page crash` —
an error that reads like an application bug and isn't. It is the single most
common Grid-in-Docker failure.

Cloud grids (BrowserStack, Sauce Labs, LambdaTest) use the same `webdriver.Remote`
call with credentials in the URL or capabilities, and add real iOS/Android Safari
— the browsers you cannot install locally.

## 4. What actually differs between browsers

| Area | Typical difference | Test impact |
|---|---|---|
| Native form controls | `<input type="date">`, `<select>` render and open differently | A click that works in Chrome misses in Firefox |
| Fonts and metrics | Different default fonts change element widths | Element pushed off-screen; click intercepted |
| Scrolling | Firefox scrolls to a different offset before clicking | Sticky headers overlap the target |
| Shadow DOM / web components | Support and piercing behaviour vary | Locators that work in one browser find nothing in another |
| Downloads | Prefs/paths configured per browser | Download tests need per-browser setup |
| Alerts | Firefox is stricter about unhandled dialogs | `UnexpectedAlertPresentException` in one browser only |
| Timing | Different JS engines, different render timing | Races that only surface on the slower browser |

That last row is the important one: **most "cross-browser failures" are actually
timing bugs your tests always had**, exposed because a different engine changed
the order of events. Fix them with explicit waits (module 02), not with
browser-specific `sleep` calls.

!!! danger "Browser-conditional logic in tests"
    ```python
    # ✗ This is how a suite dies
    if browser_name == "firefox":
        time.sleep(2)
        driver.execute_script("arguments[0].click();", button)
    else:
        button.click()
    ```
    You now maintain two tests per test, and the Firefox path silently stops
    exercising a real click — so a genuine Firefox click bug can never be
    detected. If a browser needs different handling, put it in the driver
    *fixture* (options, preferences, window size), never in the test body.

## 5. Which tests to run everywhere

Running 400 tests × 4 browsers means 1,600 executions to find perhaps three
rendering bugs. Tier instead:

| Tier | Scope | Browsers | When |
|---|---|---|---|
| Smoke | 10–20 critical paths | All supported | Every merge |
| Full regression | Everything | Primary browser only | Every merge |
| Full cross-browser | Everything | All supported | Nightly / pre-release |
| Visual checks | Layout-heavy pages | All supported | Pre-release |

```python
@pytest.mark.smoke
@pytest.mark.ui
def test_checkout_completes(driver, base_url):
    ...
```

```bash
pytest -m smoke --browsers chrome,firefox,edge -n 6
pytest -m "ui and not smoke" --browsers chrome -n 4
```

Pick the supported browser list from your analytics, not from principle. If 0.4%
of your users are on Firefox and none on Safari, four-browser CI is spending real
money to protect almost nobody.

Cross-browser tests parallelise unusually well — different browsers touch
different processes and rarely contend — so `-n` (module 08) is where you buy the
runtime back.

## 6. Traps

!!! warning "Headless is not the same browser"
    Headless Chrome has no real window manager. Print dialogs, some file-picker
    flows, and certain focus behaviours differ, and the default headless window
    size is often smaller than a real desktop — which can silently activate your
    site's mobile layout and break locators. Always set an explicit
    `--window-size`, and run the pre-release pass headed.

!!! warning "Version drift between local and CI"
    Chrome auto-updates; your CI image doesn't. A test that fails only in CI is
    often a browser-version difference. Pin the version in CI, log it into the
    report metadata (module 07), and update deliberately.

!!! warning "Safari is not optional if you have iPhone users"
    WebKit is the only engine on iOS — even "Chrome on iPhone" is WebKit. Chrome
    and Edge both being Chromium means testing both adds far less than adding
    one WebKit browser does.

!!! warning "One driver fixture for all browsers, silently misconfigured"
    An `else` branch that falls through to Chrome means `--browsers safari`
    quietly runs Chrome and reports `[safari] PASSED`. Raise on an unknown name,
    as section 1 does.

## Cheat sheet

| Need | Code |
|---|---|
| Run every browser | `@pytest.fixture(params=["chrome", "firefox"])` |
| Browser from CLI | `pytest_generate_tests` + `--browsers` |
| Run one browser | `pytest -k chrome` |
| Headless Chrome | `options.add_argument("--headless=new")` |
| Headless Firefox | `options.add_argument("--headless")` |
| Fixed viewport | `options.add_argument("--window-size=1440,900")` |
| Remote/Grid | `webdriver.Remote(command_executor=url, options=options)` |
| Grid in Docker | `--shm-size=2g` — not optional |
| Driver binaries | Selenium Manager handles it (Selenium ≥ 4.6) |
| Watch a Grid session | Port 7900, noVNC, password `secret` |
| Speed it up | Combine with `-n` from module 08 |

## Exercise

!!! note "Browsers required"
    These steps drive real browsers. Install Chrome and Firefox (and Edge if you
    can) before starting; the Docker step needs Docker running.

1. Write the parametrized `driver` fixture from section 1 and run one existing
   POM test against Chrome and Firefox. Confirm the test ids show `[chrome]` and
   `[firefox]`.
2. Add `--browsers` via `pytest_generate_tests`. Verify the default runs Chrome
   only and that `--browsers chrome,firefox` doubles the collected count with
   `--collect-only -q`.
3. Deliberately set a headless window size of `500x800` and run a test that
   clicks something in your site's desktop navigation. Record what fails, then
   explain it in one sentence.
4. Start `selenium/standalone-chrome` in Docker *without* `--shm-size=2g`, run a
   test that loads several heavy pages, and record the exact error. Restart with
   the flag and confirm it goes away.
5. Find one test in your suite that passes in Chrome and fails in Firefox. Fix it
   with an explicit wait rather than an `if browser_name ==` branch, and write
   one sentence on what the original failure said about your locator strategy.
