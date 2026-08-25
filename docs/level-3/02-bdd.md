# 02 · BDD with pytest-bdd / Behave

Behavior-Driven Development writes test scenarios in plain English (Gherkin)
first, then wires each line to Python code. The point isn't ceremony — it's
that a `.feature` file becomes a living, executable spec that a product owner
can read and a `pytest` run can verify, closing the gap between the test case
tables you wrote by hand in Level 1 Module 2 and the automated suite that
checks them.

## 1. Install

```bash
pip install pytest-bdd
```

`pytest-bdd` plugs Gherkin into the pytest runner you already use — no
separate test runner, no separate reporting pipeline. (Behave is a standalone
alternative with its own runner; the concepts below transfer directly, only
the wiring syntax differs.)

## 2. A feature file

```gherkin
# features/login.feature
Feature: Login
  Scenario: Successful login with valid credentials
    Given a user with username "alice" and password "secret123"
    When the user attempts to log in
    Then the login should succeed

  Scenario Outline: Rejected logins
    Given a user with username "<username>" and password "<password>"
    When the user attempts to log in
    Then the login should fail

    Examples:
      | username | password  |
      | alice    | wrong     |
      | bob      | secret123 |
```

`Scenario Outline` with `Examples` is Gherkin's version of
`@pytest.mark.parametrize` — one scenario, run once per row.

## 3. Step definitions

```python
# test_login.py
import pytest
from pytest_bdd import scenarios, given, when, then, parsers

scenarios("features/login.feature")

VALID_USERS = {"alice": "secret123"}

@pytest.fixture
def context():
    return {}

@given(parsers.parse('a user with username "{username}" and password "{password}"'))
def set_credentials(context, username, password):
    context["username"] = username
    context["password"] = password

@when("the user attempts to log in")
def attempt_login(context):
    context["result"] = VALID_USERS.get(context["username"]) == context["password"]

@then("the login should succeed")
def assert_success(context):
    assert context["result"] is True

@then("the login should fail")
def assert_failure(context):
    assert context["result"] is False
```

```text
$ pytest test_login.py -v
test_login.py::test_successful_login_with_valid_credentials PASSED
test_login.py::test_rejected_logins[alice-wrong] PASSED
test_login.py::test_rejected_logins[bob-secret123] PASSED
```

`scenarios("features/login.feature")` auto-generates one pytest test function
per scenario — `test_successful_login_with_valid_credentials` came straight
from the scenario title, and each Examples row became its own parametrized
test ID, just like `@pytest.mark.parametrize` IDs.

## 4. Sharing state between steps with a fixture

Gherkin steps are separate function calls, so they need a shared place to
stash data — that's what `context` is above. It's an ordinary pytest fixture
(function-scoped, fresh per scenario), not a special BDD concept. This is the
same pattern as passing data through a fixture chain in Level 1 Module 8; BDD
just adds a naming layer (`Given`/`When`/`Then`) on top.

## 5. `parsers.parse` vs `parsers.re`

`parsers.parse('... "{username}" ...')` uses simple `{name}` placeholders —
readable, but each placeholder captures greedily up to the next literal
character. For patterns needing real regex (optional groups, alternation),
use `parsers.re`:

```python
from pytest_bdd import parsers

@given(parsers.re(r'a user with username "(?P<username>\w+)"'))
def set_username(context, username):
    context["username"] = username
```

## 6. Testing-specific traps

**Trap 1 — step text that doesn't match anything.** A typo between the
`.feature` file's step text and the `@given`/`@when`/`@then` decorator string
produces `StepDefinitionNotFoundError`, not a normal assertion failure. Always
run with `-v` right after writing a new scenario so a missing step definition
surfaces immediately rather than being confused with a logic bug later.

**Trap 2 — reusing one step definition with subtly different meanings.**
`Given "a user exists"` used in one feature to mean "seeded in the database"
and in another to mean "logged in" will silently produce wrong behavior if
`pytest-bdd` matches the same decorator against both. Keep step text specific
enough that a step means exactly one thing everywhere it's used.

**Trap 3 — fixture-scope mismatches between steps and setup.** If `context` is
function-scoped but a `Background` section (steps run before every scenario in
a feature) needs to persist data set up in an earlier scenario, that's a
signal the state should live in a database or file, not a fixture — carrying
state *across* scenarios violates test isolation and produces order-dependent
failures.

**Trap 4 — treating Gherkin as documentation instead of the source of truth.**
If the `.feature` file says "the login should fail" but the step definition
actually asserts `result is True`, the scenario name is now lying to whoever
reads it, even though pytest still reports a pass or fail correctly. Review
feature files as carefully as you review code — they're the artifact a
non-engineer will actually read.

## Cheat sheet

| Concept | Gherkin | pytest-bdd wiring |
|---|---|---|
| Setup | `Given ...` | `@given("...")` |
| Action | `When ...` | `@when("...")` |
| Assertion | `Then ...` | `@then("...")` |
| Parametrization | `Scenario Outline` + `Examples` | one pytest test per row, auto-IDed |
| Shared state | (implicit) | ordinary pytest fixture, e.g. `context` |
| Placeholder capture | `"{username}"` in step text | `parsers.parse` / `parsers.re` |
| Regrouping steps | `Background:` | steps run before each `Scenario` in the file |
| Run it | — | plain `pytest`, no separate runner |

## Exercise

1. Write a `features/checkout.feature` with a `Scenario Outline` covering: an
   empty cart, a cart under the free-shipping threshold, and a cart over it —
   parametrize cart total and expect a computed shipping fee.
2. Implement the step definitions with a `context` fixture, using
   `parsers.parse` for the cart total.
3. Add a `Background:` section that seeds a fixed tax rate before every
   scenario in the file, and reference it from the `Then` step.
4. Deliberately misspell one step in the `.feature` file so it no longer
   matches its `@then` decorator, run `pytest -v`, and paste the exact error
   pytest-bdd raises.
5. Convert one `Scenario` (not Outline) into a `Scenario Outline` with three
   `Examples` rows and confirm three separate test IDs appear in `pytest -v`
   output.
