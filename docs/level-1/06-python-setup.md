# 06 · Python Setup for Testers

From here on, the course is hands-on Python. This module gets you a working,
isolated environment and a project layout that scales from three tests to three
thousand — which matters more than it sounds, because most broken test suites
started as a folder of loose scripts.

## 1. Check your Python

You need Python 3.9 or newer. Check what you have:

```bash
python3 --version
```

```text
Python 3.11.9
```

If that fails, install Python from [python.org](https://www.python.org/downloads/)
(Windows/macOS), or via your package manager on Linux (`sudo apt install
python3 python3-venv` on Debian/Ubuntu).

!!! warning "python vs python3"
    On macOS and Linux, `python` may not exist or may point at an old Python 2.
    Use `python3` explicitly. On Windows, `python` is usually correct, and the
    launcher `py -3` is the most reliable option.

## 2. Virtual environments

A virtual environment is a private, per-project copy of Python and its packages.
Without one, every project on your machine shares the same site-packages
directory — so upgrading Selenium for one project silently breaks another. For a
tester this is not a stylistic preference: **an environment you cannot reproduce
means test results you cannot trust.**

Create one inside your project folder:

```bash
mkdir shop-tests
cd shop-tests
python3 -m venv .venv
```

Activate it:

```bash
# macOS / Linux
source .venv/bin/activate

# Windows (PowerShell)
.venv\Scripts\Activate.ps1

# Windows (cmd)
.venv\Scripts\activate.bat
```

Your prompt changes to show the environment is active:

```text
(.venv) $ which python
/Users/you/shop-tests/.venv/bin/python
```

Deactivate when you're done:

```bash
deactivate
```

!!! tip "Skip activation entirely"
    You can always call the environment's interpreter directly —
    `.venv/bin/python -m pytest` on macOS/Linux, `.venv\Scripts\python -m pytest`
    on Windows. This is what CI does, and it removes a whole class of "wrong
    Python" mistakes. Every command in this course works either way.

## 3. Installing packages with pip

With the environment active:

```bash
pip install pytest
```

```text
Collecting pytest
  Downloading pytest-8.3.2-py3-none-any.whl (341 kB)
...
Successfully installed iniconfig-2.0.0 packaging-24.1 pluggy-1.5.0 pytest-8.3.2
```

Verify:

```bash
pytest --version
```

```text
pytest 8.3.2
```

Useful pip commands:

```bash
pip list                      # what's installed
pip show pytest               # details for one package
pip install "selenium==4.23.1"  # pin an exact version
pip install --upgrade pytest  # upgrade
pip uninstall requests        # remove
```

### The starter toolkit

Install the packages this level uses:

```bash
pip install pytest selenium requests webdriver-manager
```

| Package | Role |
|---|---|
| `pytest` | The test framework — runner, assertions, fixtures |
| `selenium` | Browser automation (Module 9) |
| `requests` | HTTP client, for API testing |
| `webdriver-manager` | Downloads the right browser driver automatically |

## 4. Pinning dependencies

Freeze exactly what's installed so anyone — a teammate, or CI — reproduces it:

```bash
pip freeze > requirements.txt
```

```text
attrs==24.2.0
certifi==2024.8.30
iniconfig==2.0.0
packaging==24.1
pluggy==1.5.0
pytest==8.3.2
requests==2.32.3
selenium==4.23.1
...
```

Recreate that environment anywhere:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

!!! info "Why pinning matters more in testing than in most code"
    When a test that passed yesterday fails today with no code change, the first
    question is "what changed?" An unpinned dependency means the answer might be
    "Selenium released 4.24 overnight." Pinned versions turn a mystery into a
    deliberate, reviewable upgrade.

## 5. Project layout for a test suite

Use this structure. Every piece has a reason.

```text
shop-tests/
├── .venv/                  # virtual environment (never committed)
├── .gitignore
├── requirements.txt        # pinned dependencies
├── pytest.ini              # pytest configuration
├── conftest.py             # shared fixtures, discovered automatically
├── README.md
├── pages/                  # Page Object classes (Level 2)
│   └── __init__.py
├── tests/
│   ├── __init__.py
│   ├── test_login.py
│   ├── test_cart.py
│   └── api/
│       ├── __init__.py
│       └── test_products_api.py
├── data/                   # test data files
│   └── users.json
└── reports/                # generated output (never committed)
```

| File / folder | Why |
|---|---|
| `tests/` separate from source | Tests are discoverable as a unit and shippable separately |
| `conftest.py` at root | pytest auto-loads it; shared fixtures live here with no imports needed |
| `pytest.ini` | One place for markers, default flags, and test paths |
| `pages/` | Page Objects — keeps locators out of tests (Level 2) |
| `data/` | Test data as files, not hard-coded in tests |
| `reports/` | Generated artefacts, git-ignored |
| `__init__.py` in test dirs | Prevents module-name collisions when two folders both hold `test_login.py` |

### .gitignore

```text
.venv/
__pycache__/
*.pyc
.pytest_cache/
reports/
screenshots/
.env
```

Never commit `.venv/` (it's large and machine-specific) or `.env` (it holds
credentials).

## 6. pytest.ini

Create `pytest.ini` in the project root:

```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = -v --strict-markers --tb=short
markers =
    smoke: quick checks that the build is testable
    regression: full regression suite
    ui: tests that drive a real browser
    api: tests that call the HTTP API only
```

| Setting | Effect |
|---|---|
| `testpaths` | pytest only searches here — faster collection |
| `python_files/classes/functions` | The discovery naming rules (these are the defaults, stated explicitly) |
| `addopts` | Flags applied to every run: verbose, short tracebacks |
| `--strict-markers` | A typo'd marker becomes an error instead of a silent no-op |
| `markers` | Declare markers so you can run subsets: `pytest -m smoke` |

`--strict-markers` is worth calling out. Without it, `@pytest.mark.smoek` runs
happily and your smoke suite quietly excludes that test forever.

## 7. Verify the whole setup

Create `tests/test_setup.py`:

```python
"""Smoke check that the test environment itself is working."""
import sys

import pytest
import requests
import selenium


def test_python_version_is_supported():
    assert sys.version_info >= (3, 9), f"Python 3.9+ required, got {sys.version}"


def test_core_packages_are_importable():
    assert pytest.__version__
    assert requests.__version__
    assert selenium.__version__


@pytest.mark.api
def test_network_access_to_test_api():
    response = requests.get("https://httpbin.org/status/200", timeout=10)
    assert response.status_code == 200
```

Run it:

```bash
pytest
```

```text
============================= test session starts ==============================
platform darwin -- Python 3.11.9, pytest-8.3.2, pluggy-1.5.0
rootdir: /Users/you/shop-tests
configfile: pytest.ini
testpaths: tests
collected 3 items

tests/test_setup.py::test_python_version_is_supported PASSED             [ 33%]
tests/test_setup.py::test_core_packages_are_importable PASSED            [ 66%]
tests/test_setup.py::test_network_access_to_test_api PASSED              [100%]

============================== 3 passed in 0.94s ===============================
```

Three passes means Python, pytest, the packages, and outbound network access all
work. If the third test fails behind a corporate proxy, that's a real
environment finding — note it, because your CI will hit the same wall.

Run only the non-network tests:

```bash
pytest -m "not api"
```

```text
collected 3 items / 1 deselected / 2 selected

tests/test_setup.py::test_python_version_is_supported PASSED             [ 50%]
tests/test_setup.py::test_core_packages_are_importable PASSED            [100%]

======================= 2 passed, 1 deselected in 0.06s ========================
```

## 8. Handling secrets

Never hard-code credentials in a test file that goes into version control.
Read them from the environment:

```python
import os

import pytest

BASE_URL = os.environ.get("APP_BASE_URL", "https://staging.example.com")
TEST_USER = os.environ.get("TEST_USER")
TEST_PASSWORD = os.environ.get("TEST_PASSWORD")


@pytest.mark.skipif(
    not TEST_USER or not TEST_PASSWORD,
    reason="TEST_USER and TEST_PASSWORD environment variables are not set",
)
def test_login_credentials_are_available():
    assert TEST_USER
    assert TEST_PASSWORD
```

Set them for a session:

```bash
export TEST_USER="qa.buyer01@test.com"
export TEST_PASSWORD="…"
pytest
```

```text
tests/test_setup.py::test_login_credentials_are_available PASSED         [100%]
```

Without them set:

```text
tests/test_setup.py::test_login_credentials_are_available SKIPPED
  (TEST_USER and TEST_PASSWORD environment variables are not set)          [100%]
```

A clean skip with a stated reason, rather than a confusing failure — that
distinction matters when someone else runs your suite for the first time.

For local development, a `.env` file (git-ignored) plus `python-dotenv` is the
usual convenience. In CI, use the platform's secret store — GitHub Actions
secrets, Jenkins credentials — never a committed file.

## 9. Editor setup

Any editor works, but VS Code with the Python extension gives you test discovery
in the sidebar, run/debug per test, and breakpoints inside tests. Point it at the
project interpreter (`.venv/bin/python`) — selecting the system Python instead is
the most common reason for "it runs in the terminal but not in the IDE."

## Exercise

1. Create a project called `booking-tests` with the exact structure from
   section 5 — every folder and file, including `.gitignore` and `pytest.ini`
   with all four markers.
2. Create and activate a virtual environment, install `pytest`, `requests`,
   `selenium` and `webdriver-manager`, and generate a pinned
   `requirements.txt`.
3. Prove the isolation works: run `pip list` inside the environment and again
   after `deactivate`, and note the difference. Explain in two sentences why
   this matters for reproducing a test failure.
4. Write `tests/test_environment.py` with four tests: Python version is 3.9+,
   each of the three packages imports, `requests` can reach
   `https://httpbin.org/get`, and a test that reads a `BASE_URL` environment
   variable and skips cleanly when it isn't set.
5. Run the suite three ways and record the output of each: everything
   (`pytest`), smoke only (`pytest -m smoke`), and everything except API tests
   (`pytest -m "not api"`). Then introduce a deliberate marker typo and confirm
   `--strict-markers` catches it.
