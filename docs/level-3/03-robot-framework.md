# 03 · Robot Framework Basics

Robot Framework is a keyword-driven test framework: test cases are written as
tables of keywords (`Add`, `Click Button`, `Should Be Equal`) rather than
Python function calls, and Python is used underneath to *implement* keywords
when the built-in library doesn't already have what you need. It's the tool
teams reach for when non-programmers (manual testers, business analysts) need
to write and read test cases directly — the tabular syntax below was actually
executed to produce the output shown.

## 1. Install

```bash
pip install robotframework
```

No separate runner binary — `pip install` gives you the `robot` command
directly.

## 2. A Python library and a `.robot` test suite

```python
# calc.py
def add(a, b):
    return int(a) + int(b)

def divide(a, b):
    return int(a) / int(b)
```

```robotframework
*** Settings ***
Library    calc.py

*** Test Cases ***
Add Two Numbers
    ${result}=    Add    2    3
    Should Be Equal As Integers    ${result}    5

Divide By Zero Raises
    Run Keyword And Expect Error    ZeroDivisionError*    Divide    10    0
```

```bash
$ robot calc.robot
```

```text
==============================================================================
Calc
==============================================================================
Add Two Numbers                                                      | PASS |
------------------------------------------------------------------------------
Divide By Zero Raises                                                | PASS |
------------------------------------------------------------------------------
Calc                                                                 | PASS |
2 tests, 2 passed, 0 failed
==============================================================================
Output:  output.xml
Log:     log.html
Report:  report.html
```

Every plain Python function in `calc.py` became a Robot **keyword**
automatically — `add` became `Add`, matched case-insensitively and with
underscores treated as spaces. `${result}=` captures a return value into a
Robot variable, and `Should Be Equal As Integers` is a keyword from Robot's
built-in library, not something you wrote.

## 3. Where the HTML report actually helps

Unlike pytest's terminal-only default output, every `robot` run produces
`report.html` (pass/fail summary) and `log.html` (a drill-down per keyword,
per test, with arguments and return values recorded at each step). For a
non-engineer stakeholder, handing over `report.html` is often more useful than
a pytest terminal transcript — this is Robot's main practical advantage over
plain pytest for cross-functional teams.

## 4. Data-driven tests with `[Template]`

```robotframework
*** Test Cases ***
Add Various Numbers
    [Template]    Add Should Equal
    2    3    5
    10   0    10
    -1   1    0

*** Keywords ***
Add Should Equal
    [Arguments]    ${a}    ${b}    ${expected}
    ${result}=    Add    ${a}    ${b}
    Should Be Equal As Integers    ${result}    ${expected}
```

`[Template]` is Robot's equivalent of `@pytest.mark.parametrize` — one keyword,
run once per data row, each row reported as a separate test result.

## 5. Custom keywords in the `.robot` file itself

Not every keyword needs a Python implementation. The `*** Keywords ***`
section above (`Add Should Equal`) composes existing keywords into a new,
higher-level one — the tabular equivalent of extracting a helper function.

## 6. Selenium/browser testing via SeleniumLibrary

Robot Framework's browser automation typically goes through the separate
`robotframework-seleniumlibrary` (Selenium-backed) or
`robotframework-browser` (Playwright-backed) packages:

```bash
pip install robotframework-browser
rfbrowser init
```

```robotframework
*** Settings ***
Library    Browser

*** Test Cases ***
Open Example Page
    New Browser    chromium    headless=True
    New Page    https://example.com
    Get Title    ==    Example Domain
```

`robotframework-browser` was not installed or run for this module — installing
it requires `rfbrowser init`, which pulls Node.js and Playwright's browser
binaries in a separate step, and depending on your sandbox this can be
slower or blocked outright. Manually verify it in your own environment before
relying on it; the syntax above is correct but unexecuted here.

## 7. Testing-specific traps

**Trap 1 — keyword name collisions across libraries.** If you import both
`SeleniumLibrary` and a custom library that also defines `Click Element`,
Robot resolves the ambiguity by import order or raises an error — silently
picking the wrong one is a real, hard-to-spot failure mode. Prefer explicit
`Library    calc.py    WITH NAME    Calc` and call `Calc.Add` when names might
collide.

**Trap 2 — `${variable}` scoping surprises.** Variables set inside one test
case with `${x}=    Set Variable    ...` do not leak into the next test case
by default (good), but variables set with `Set Global Variable` do leak across
the entire suite (often not what you want) — mixing the two in the same file
produces state that depends on execution order.

**Trap 3 — string-typed arguments by default.** Every argument coming from a
`.robot` table is a string unless you convert it — `Add    2    3` works above
only because `calc.py`'s `add` calls `int()` itself. Forgetting the conversion
in your own keyword produces `TypeError: unsupported operand type(s)` or,
worse, silent string concatenation (`"2" + "3"` behavior) if your function
uses `+` without casting.

**Trap 4 — over-nesting `Run Keyword If`.** Robot's conditional keywords read
fine one level deep; nesting several is far harder to read than the
equivalent Python `if/elif`. If a test needs real branching logic, that logic
usually belongs inside a Python keyword implementation, not the `.robot` file.

## Cheat sheet

| pytest | Robot Framework |
|---|---|
| `def test_add(): assert add(2,3)==5` | `Add Two Numbers` test case + `Should Be Equal As Integers` |
| `@pytest.mark.parametrize` | `[Template]` + data rows |
| Python function | Keyword (from a library or `*** Keywords ***`) |
| `pytest.ini` / `conftest.py` | `*** Settings ***`, `__init__.robot` for suite setup |
| Terminal pass/fail + optional HTML plugin | `report.html` + `log.html` generated by default |
| `pytest.raises(ZeroDivisionError)` | `Run Keyword And Expect Error    ZeroDivisionError*    ...` |
| Fixtures (`setup`/`teardown`) | `Suite Setup` / `Test Setup` / `Test Teardown` in `*** Settings ***` |

## Exercise

1. Extend `calc.py` with a `subtract` and `multiply` function, both casting
   arguments with `int()`.
2. Write a `.robot` suite with individual test cases for each, plus one
   `[Template]`-based data-driven test covering at least four row combinations
   of `add`.
3. Add a `*** Keywords ***` section defining a custom keyword
   `Result Should Be Positive` that takes one argument and asserts it's
   greater than zero; use it in a new test case.
4. Run `robot calc.robot`, open the generated `log.html` in a browser, and
   find the exact argument values Robot recorded for one `[Template]` row.
5. Deliberately remove the `int()` casts from `add`, rerun the suite, and
   record the exact string-concatenation result Robot reports instead of a
   `TypeError` — explain why no exception was raised.
