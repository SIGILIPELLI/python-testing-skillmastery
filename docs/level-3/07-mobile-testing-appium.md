# 07 · Mobile Testing with Appium

Appium extends the WebDriver protocol from Selenium (Level 1 Module 9) to
native and hybrid mobile apps on Android and iOS. If you know
`driver.find_element(By.ID, ...).click()`, you already know most of Appium's
API — the differences are in what's being driven (a real or emulated device,
plus an Appium server process) rather than in the Python code you write.

**What actually ran for this module:** only `pip install
Appium-Python-Client`, which installed cleanly. Running any of the code below
needs an Appium server (`npm install -g appium`), a configured Android
emulator or iOS simulator, and a built app under test — none of which are
available in this sandboxed environment. The code is syntactically correct
and follows the official client's API, but treat it as **reviewed, not
verified** until you run it against a real emulator on your own machine.

## 1. Install

```bash
pip install Appium-Python-Client
npm install -g appium
appium driver install uiautomator2   # Android
appium driver install xcuitest       # iOS
```

The Python package only gives you a client. `appium` itself is a Node.js
server that receives WebDriver commands and translates them into
platform-specific automation calls — you must have it running
(`appium` in a terminal, listening on `http://localhost:4723` by default)
before any test can connect.

## 2. Connecting to a device

```python
from appium import webdriver
from appium.options.android import UiAutomator2Options

options = UiAutomator2Options()
options.platform_name = "Android"
options.device_name = "emulator-5554"
options.app = "/path/to/app-debug.apk"
options.automation_name = "UiAutomator2"

driver = webdriver.Remote("http://localhost:4723", options=options)
```

This mirrors `webdriver.Chrome()` from Level 1 almost exactly — `Remote(...)`
here just points at the Appium server instead of a local browser process, and
`options` describes the target device and app instead of browser flags.

## 3. Finding elements — mobile-specific locator strategies

```python
from appium.webdriver.common.appiumby import AppiumBy

# Android resource-id (most stable — like an HTML id attribute)
login_button = driver.find_element(AppiumBy.ID, "com.example.app:id/login_button")

# Accessibility ID — cross-platform, matches iOS accessibility labels too
username_field = driver.find_element(AppiumBy.ACCESSIBILITY_ID, "username_input")

# UiAutomator (Android-only DSL for complex matches)
element = driver.find_element(
    AppiumBy.ANDROID_UIAUTOMATOR,
    'new UiSelector().text("Sign In")'
)
```

`AppiumBy.ACCESSIBILITY_ID` deserves special attention: it's the one locator
strategy that works identically on Android and iOS, because both platforms
expose an accessibility label for screen readers — reusing it as a test hook
means one test can often run against both platform builds with no locator
changes.

## 4. A full test structure (pytest, unexecuted)

```python
import pytest
from appium import webdriver
from appium.options.android import UiAutomator2Options
from appium.webdriver.common.appiumby import AppiumBy
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

@pytest.fixture
def driver():
    options = UiAutomator2Options()
    options.platform_name = "Android"
    options.device_name = "emulator-5554"
    options.app = "/path/to/app-debug.apk"
    d = webdriver.Remote("http://localhost:4723", options=options)
    yield d
    d.quit()

def test_login_success(driver):
    wait = WebDriverWait(driver, 15)
    username = wait.until(
        EC.presence_of_element_located((AppiumBy.ACCESSIBILITY_ID, "username_input"))
    )
    username.send_keys("qa_user")
    driver.find_element(AppiumBy.ACCESSIBILITY_ID, "password_input").send_keys("Valid#123")
    driver.find_element(AppiumBy.ID, "com.example.app:id/login_button").click()

    home_title = wait.until(
        EC.visibility_of_element_located((AppiumBy.ACCESSIBILITY_ID, "home_screen_title"))
    )
    assert home_title.text == "Welcome"
```

Note this is the *exact* fixture-plus-`WebDriverWait` shape from Level 1
Module 9 — Selenium's explicit-wait patterns (Level 1 Module 9, Section 7)
transfer to Appium unchanged, because Appium literally implements the
WebDriver wire protocol.

## 5. Mobile-only gestures

```python
from appium.webdriver.common.touch_action import TouchAction

# Swipe (deprecated TouchAction API shown for clarity; W3C actions are current)
driver.swipe(start_x=500, start_y=1500, end_x=500, end_y=300, duration=400)

# Native context switch — WebViews inside a hybrid app expose a browser-like DOM
contexts = driver.contexts             # e.g. ['NATIVE_APP', 'WEBVIEW_com.example.app']
driver.switch_to.context("WEBVIEW_com.example.app")
driver.find_element(AppiumBy.CSS_SELECTOR, "#submit").click()
driver.switch_to.context("NATIVE_APP")
```

Hybrid apps (native shell + embedded web content) need this context switch —
without it, `find_element` searches the native view hierarchy and simply
won't see anything inside the WebView, producing a `NoSuchElementException`
that looks identical to a normal missing-element bug but has a completely
different fix.

## 6. Testing-specific traps

**Trap 1 — resource IDs that aren't stable across app versions.** Unlike a
web app's HTML `id`, Android resource IDs can shift when a developer
refactors a layout file, silently breaking every test using `AppiumBy.ID`.
Prefer `ACCESSIBILITY_ID` where the app exposes one — it's tied to the
user-facing label, which changes far less often than internal resource names.

**Trap 2 — device/emulator state bleeding between tests.** A test that
installs the app, logs in, and never resets state leaves the *next* test
starting from a logged-in home screen instead of a fresh install — Appium
supports `driver.reset()` or, better, uninstalling and reinstalling the app
between test runs (`options.full_reset = True`) for real isolation, mirroring
why Level 1's Selenium fixture created a fresh browser per test.

**Trap 3 — flakiness from animation timing.** Native mobile UIs animate
screen transitions far more than most web pages; clicking a button mid-
animation can land the tap in the wrong place. `WebDriverWait` with
`EC.element_to_be_clickable` helps, but some teams also disable animations
in the test build (`adb shell settings put global window_animation_scale 0`)
to remove this entire class of flakiness.

**Trap 4 — confusing simulator/emulator differences with real-device bugs.**
An iOS Simulator or Android Emulator doesn't reproduce real hardware
constraints (camera, GPS accuracy, low-memory conditions, real network
latency). A test suite passing entirely on emulators can still ship bugs that
only appear on physical devices — treat emulator-only coverage as necessary,
not sufficient, especially for anything hardware-adjacent.

## Cheat sheet

| Selenium (Level 1) | Appium |
|---|---|
| `webdriver.Chrome()` | `webdriver.Remote("http://localhost:4723", options=...)` |
| `By.ID` | `AppiumBy.ID` (platform resource-id) |
| N/A | `AppiumBy.ACCESSIBILITY_ID` (cross-platform) |
| N/A | `AppiumBy.ANDROID_UIAUTOMATOR` (Android-only DSL) |
| `WebDriverWait` + `EC` | identical API, same import |
| Browser tabs/windows | `driver.contexts` — `NATIVE_APP` vs `WEBVIEW_*` |
| Fresh browser per test | `options.full_reset = True` or `driver.reset()` |
| N/A | disable device animations to reduce transition flakiness |

## Exercise

*(This exercise assumes access to Appium server + an Android emulator or iOS
simulator — set that up first if you want to actually execute it.)*

1. Install `appium`, run `appium driver install uiautomator2`, start an
   Android emulator, and confirm `adb devices` lists it as `device` (not
   `offline`).
2. Start the Appium server and connect with the `webdriver.Remote(...)`
   snippet from section 2 against any installed app (even a system app like
   Settings) — confirm `driver.current_activity` returns something sensible.
3. Find three elements on that app's home screen using three different
   locator strategies (`ID`, `ACCESSIBILITY_ID`, `ANDROID_UIAUTOMATOR`) and
   print each element's `.text` or `.get_attribute("content-desc")`.
4. Write a pytest fixture that launches the app fresh
   (`options.full_reset = True`) before each test and confirm two
   independent tests don't see each other's leftover state.
5. If you don't have emulator access: write out, in prose, exactly which
   three setup steps (server, driver, device) blocked you, and what the error
   message was at each step — that diagnostic skill transfers directly to
   debugging a real Appium environment later.
