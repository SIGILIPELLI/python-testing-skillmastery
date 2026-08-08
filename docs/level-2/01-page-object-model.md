# 01 · Page Object Model in Python

Level 1 wrote locators directly inside test functions. That works for ten
tests. It falls apart at a hundred: change one CSS class in the app and you're
grepping through dozens of test files to fix the same broken locator ten times
over. The **Page Object Model (POM)** is the fix — one class per page (or
component), owning that page's locators and actions, so a UI change means one
edit in one place.

## 1. The problem, concretely

Without POM, every test re-describes the page:

```python
def test_submitting_form_shows_confirmation(driver, wait, base_url):
    driver.get(f"{base_url}/web-form.html")
    driver.find_element(By.NAME, "my-text").send_keys("QA automation")
    wait.until(EC.element_to_be_clickable((By.CSS_SELECTOR, "button"))).click()
    message = wait.until(EC.visibility_of_element_located((By.ID, "message")))
    assert message.text == "Received!"


def test_disabled_input_cannot_be_edited(driver, base_url):
    driver.get(f"{base_url}/web-form.html")
    disabled_field = driver.find_element(By.NAME, "my-disabled")
    assert not disabled_field.is_enabled()
```

Both tests know the URL, both know `By.NAME, "my-text"` is the text box, both
know how to find the submit button. If the developer renames `my-text` to
`text-input`, you fix it in every test that touches it.

## 2. A minimal page object

A page object exposes **actions** ("submit the form", "select this dropdown
option") and **state** ("what does the message say"). It never contains
`assert` — that stays in the test.

```python
# pages/web_form_page.py
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait, Select
from selenium.webdriver.support import expected_conditions as EC


class WebFormPage:
    URL = "/web-form.html"

    # Locators — the ONLY place they're allowed to appear
    TEXT_INPUT = (By.NAME, "my-text")
    PASSWORD_INPUT = (By.NAME, "my-password")
    TEXTAREA = (By.NAME, "my-textarea")
    DISABLED_INPUT = (By.NAME, "my-disabled")
    DROPDOWN = (By.NAME, "my-select")
    CHECKBOX_2 = (By.ID, "my-check-2")
    SUBMIT_BUTTON = (By.CSS_SELECTOR, "button")
    MESSAGE = (By.ID, "message")

    def __init__(self, driver, base_url):
        self.driver = driver
        self.base_url = base_url
        self.wait = WebDriverWait(driver, timeout=10)

    def load(self):
        self.driver.get(f"{self.base_url}{self.URL}")
        return self

    def enter_text(self, value):
        self.driver.find_element(*self.TEXT_INPUT).send_keys(value)
        return self

    def enter_password(self, value):
        self.driver.find_element(*self.PASSWORD_INPUT).send_keys(value)
        return self

    def select_dropdown_option(self, visible_text):
        Select(self.driver.find_element(*self.DROPDOWN)).select_by_visible_text(visible_text)
        return self

    def check_checkbox(self):
        self.driver.find_element(*self.CHECKBOX_2).click()
        return self

    def submit(self):
        self.wait.until(EC.element_to_be_clickable(self.SUBMIT_BUTTON)).click()
        return self

    def is_text_input_disabled(self):
        return not self.driver.find_element(*self.DISABLED_INPUT).is_enabled()

    def get_confirmation_message(self):
        return self.wait.until(EC.visibility_of_element_located(self.MESSAGE)).text
```

Notice every action method returns `self` (or, when navigation happens, the
*next* page — section 4). That's a **fluent interface**, and it's what makes
tests read like a script rather than a wall of `find_element` calls.

## 3. The test, rewritten

```python
# tests/test_web_form_pom.py
import pytest
from pages.web_form_page import WebFormPage


@pytest.fixture
def form_page(driver, base_url):
    return WebFormPage(driver, base_url).load()


def test_submitting_form_shows_confirmation(form_page):
    form_page.enter_text("QA automation").submit()
    assert form_page.get_confirmation_message() == "Received!"


def test_disabled_input_cannot_be_edited(form_page):
    assert form_page.is_text_input_disabled()


def test_dropdown_selection_persists_after_submit(form_page):
    form_page.select_dropdown_option("Two").submit()
    assert "my-select=2" in form_page.driver.current_url
```

```text
tests/test_web_form_pom.py::test_submitting_form_shows_confirmation PASSED    [ 33%]
tests/test_web_form_pom.py::test_disabled_input_cannot_be_edited PASSED       [ 66%]
tests/test_web_form_pom.py::test_dropdown_selection_persists_after_submit PASSED [100%]

============================== 3 passed in 4.87s ===============================
```

The test file no longer imports `By`, never calls `find_element`, and reads as
a sequence of user actions and assertions. When the developer renames
`my-text`, exactly one line changes: `TEXT_INPUT` in `web_form_page.py`.

## 4. A base page for shared behaviour

Every page object needs `driver`, `wait`, and usually the same handful of
helpers (safe click, get text, wait for a toast). Put those in a `BasePage`
and inherit.

```python
# pages/base_page.py
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


class BasePage:
    def __init__(self, driver, base_url):
        self.driver = driver
        self.base_url = base_url
        self.wait = WebDriverWait(driver, timeout=10)

    def click(self, locator):
        self.wait.until(EC.element_to_be_clickable(locator)).click()

    def type_into(self, locator, text, clear_first=True):
        field = self.wait.until(EC.visibility_of_element_located(locator))
        if clear_first:
            field.clear()
        field.send_keys(text)

    def text_of(self, locator):
        return self.wait.until(EC.visibility_of_element_located(locator)).text

    def is_current_url(self, fragment):
        return fragment in self.driver.current_url
```

```python
# pages/web_form_page.py
from selenium.webdriver.common.by import By
from .base_page import BasePage


class WebFormPage(BasePage):
    URL = "/web-form.html"
    TEXT_INPUT = (By.NAME, "my-text")
    SUBMIT_BUTTON = (By.CSS_SELECTOR, "button")
    MESSAGE = (By.ID, "message")

    def load(self):
        self.driver.get(f"{self.base_url}{self.URL}")
        return self

    def submit_text(self, value):
        self.type_into(self.TEXT_INPUT, value)
        self.click(self.SUBMIT_BUTTON)
        return self

    def confirmation_message(self):
        return self.text_of(self.MESSAGE)
```

`WebFormPage` no longer knows *how* to wait or click — it only knows *what* to
click. That split (base page = mechanics, page object = page-specific
vocabulary) is what keeps a 40-page framework maintainable.

## 5. Chaining across pages

When an action navigates somewhere new, return the *next* page object instead
of `self`. This makes multi-page flows type-checkable and self-documenting.

```python
# pages/login_page.py
from selenium.webdriver.common.by import By
from .base_page import BasePage
from .dashboard_page import DashboardPage


class LoginPage(BasePage):
    URL = "/login"
    EMAIL = (By.ID, "email")
    PASSWORD = (By.ID, "password")
    SIGN_IN = (By.ID, "signin")
    FIELD_ERROR = (By.CSS_SELECTOR, ".field-error")

    def load(self):
        self.driver.get(f"{self.base_url}{self.URL}")
        return self

    def login(self, email, password) -> DashboardPage:
        """Successful login navigates away — return the page the user lands on."""
        self.type_into(self.EMAIL, email)
        self.type_into(self.PASSWORD, password)
        self.click(self.SIGN_IN)
        return DashboardPage(self.driver, self.base_url)

    def login_expecting_failure(self, email, password) -> str:
        """Failed login stays on this page — return the error, not a page object."""
        self.type_into(self.EMAIL, email)
        self.type_into(self.PASSWORD, password)
        self.click(self.SIGN_IN)
        return self.text_of(self.FIELD_ERROR)
```

```python
def test_valid_login_reaches_dashboard(login_page):
    dashboard = login_page.login("qa.buyer01@test.com", "Valid#123")
    assert dashboard.welcome_message().startswith("Welcome")


def test_invalid_login_shows_error(login_page):
    error = login_page.login_expecting_failure("qa.unknown@test.com", "Valid#123")
    assert error == "No account found for this email"
```

Two different return types for two different outcomes is intentional — a
reviewer sees at a glance which path leaves the page and which doesn't,
without opening the method body.

## 6. What does *not* belong in a page object

| Belongs in the page object | Belongs in the test |
|---|---|
| Locators | `assert` statements |
| "Click submit", "type into field" | "the message should equal 'Received!'" |
| Waiting for *this page's* elements | Deciding *what* success looks like |
| Returning text/state for the test to check | Interpreting that state |

!!! warning "The most common POM mistake: asserting inside the page object"
    ```python
    # ✗ Wrong — the page object now decides what "success" means
    def submit(self):
        self.click(self.SUBMIT_BUTTON)
        assert self.text_of(self.MESSAGE) == "Received!"
    ```
    This couples every test that calls `submit()` to one expected outcome,
    and a test that wants to verify a *validation error* instead has to work
    around an assertion baked into the action. Return the state; let the test
    decide what it means.

## 7. Fat page objects and the fix

A page object that grows fifty methods for one enormous page is itself a
maintenance problem. Split by **component**, not just by page: a `HeaderNav`
object, a `SearchWidget` object, and a `ProductListPage` that composes them.

```python
class ProductListPage(BasePage):
    def __init__(self, driver, base_url):
        super().__init__(driver, base_url)
        self.nav = HeaderNav(driver, base_url)
        self.search = SearchWidget(driver, base_url)

    def product_count(self):
        return len(self.driver.find_elements(By.CSS_SELECTOR, ".product-card"))
```

```python
def test_search_narrows_results(product_list_page):
    product_list_page.search.search_for("trail running shoes")
    assert product_list_page.product_count() == 3
```

Now `HeaderNav` and `SearchWidget` are reusable on every page that has them,
instead of duplicated inside `ProductListPage`, `CheckoutPage`, and
`AccountPage` separately.

## Cheat sheet

| Principle | Rule |
|---|---|
| Locators | Live only inside the page object, never in a test |
| Assertions | Live only inside the test, never in the page object |
| Method naming | Name for user intent (`login`, `add_to_cart`), not implementation (`click_button_3`) |
| Return values | `self` for same-page actions, next page object for navigation |
| Shared mechanics | Factor into a `BasePage` |
| Large pages | Split into composable component objects, not one giant class |
| Waits | Belong in the page object (or base page) — a test should never call `time.sleep` or raw `find_element` |

## Exercise

Using the Selenium practice page at
`https://www.selenium.dev/selenium/web/web-form.html`:

1. Write `pages/base_page.py` with `click`, `type_into`, `text_of`, and
   `is_current_url` exactly as shown in section 4.
2. Write `pages/web_form_page.py` inheriting from `BasePage`, covering the
   text input, textarea, dropdown, checkbox, and submit button — with no
   locator repeated between methods.
3. Rewrite five tests from [Level 1's Selenium module](../level-1/09-selenium-basics.md)
   to use your page object instead of raw `find_element` calls. Confirm they
   still pass.
4. Add one method that deliberately violates section 6 (asserts inside the
   page object), watch it make a test's failure message confusing, then fix
   it by moving the assertion back into the test.
5. Split `WebFormPage` into `WebFormPage` plus a `DropdownWidget` component
   object owning just the `Select` logic, and use it from two different
   tests.
