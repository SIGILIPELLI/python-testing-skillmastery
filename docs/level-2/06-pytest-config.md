# 06 · pytest Plugins & Configuration

Every command you've typed so far — `pytest -v`, `pytest -k login`, `pytest -m
smoke` — is a decision your team has to remember and repeat identically. A
configuration file makes those decisions once, so CI, your laptop, and the new
hire's laptop all run the suite the same way.

This module covers the config file, markers, `conftest.py`, custom command-line
options, and the plugins worth installing.

## 1. One config file

pytest reads `pytest.ini`, `pyproject.toml`, `tox.ini`, or `setup.cfg`. Pick one.
`pytest.ini` wins over the others when several exist, which is exactly the kind
of surprise you don't want — so don't keep several.

```ini
# pytest.ini
[pytest]
testpaths = tests
addopts = -ra --strict-markers --tb=short
markers =
    smoke: fast checks that must pass before anything else runs
    api: hits a real HTTP service
    ui: drives a browser
```

The same thing in `pyproject.toml`, if your project already has one:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-ra --strict-markers --tb=short"
markers = [
    "smoke: fast checks that must pass before anything else runs",
    "api: hits a real HTTP service",
    "ui: drives a browser",
]
```

The directory containing this file becomes the **rootdir**, which is also what
pytest adds to `sys.path` — so a config file at the repo root is usually what
makes `from pages.login_page import LoginPage` resolve.

| Setting | Does |
|---|---|
| `testpaths` | Where to look when no path is given — stops pytest scanning `.venv` |
| `addopts` | Flags applied to every run |
| `markers` | Declares valid marker names |
| `python_files` / `python_classes` / `python_functions` | Override the `test_*` naming convention |
| `norecursedirs` | Directories never to collect from |
| `filterwarnings` | Turn specific warnings into errors |
| `log_cli = true` | Stream `logging` output live during the run |

Two flags in `addopts` earn their place immediately: `-ra` prints a summary of
every skip/xfail/error at the end (otherwise skipped tests are invisible), and
`--strict-markers` is covered next.

## 2. Markers

A marker is a label you attach to tests so you can select them later.

```python
import pytest


@pytest.mark.smoke
def test_login_page_loads():
    assert True


@pytest.mark.api
def test_orders_endpoint():
    assert 1 + 1 == 2


@pytest.mark.ui
@pytest.mark.smoke
def test_checkout_button_visible():
    assert True


@pytest.mark.skip(reason="blocked by DEV-4412")
def test_refund_flow():
    assert False


@pytest.mark.xfail(reason="known rounding bug DEV-4498")
def test_tax_rounding():
    assert 0.1 + 0.2 == 0.3
```

```text
rootdir: /work/demo
configfile: pytest.ini
testpaths: tests
collected 5 items

tests/test_suite.py ...sx                                                [100%]

=========================== short test summary info ============================
SKIPPED [1] tests/test_suite.py:20: blocked by DEV-4412
XFAIL tests/test_suite.py::test_tax_rounding - known rounding bug DEV-4498
=================== 3 passed, 1 skipped, 1 xfailed in 0.05s ====================
```

Note the header confirms which config file and testpaths were used — the first
thing to check when a run does something unexpected.

Selecting with `-m`:

```text
$ pytest -m smoke -q
..                                                                       [100%]
2 passed, 3 deselected in 0.03s
```

`-m` takes a boolean expression: `-m "smoke and not ui"`, `-m "api or ui"`.

!!! warning "`--strict-markers` protects declarations, not expressions"
    `--strict-markers` turns `@pytest.mark.smole` (typo) into a collection
    **error** instead of a silently-ignored marker — which is exactly what you
    want, because an unregistered marker means those tests quietly never run in
    your smoke job. It does **not** validate `-m` expressions:
    ```text
    $ pytest -m "smoke and typo" -q
    5 deselected in 0.00s
    ```
    Zero tests, exit code 5, no error. A CI job that treats "nothing ran" as
    success will stay green forever. Add `--exitfirst`-style guards or check the
    collected count in your pipeline.

`skip` vs `xfail` is a real distinction, not a style choice:

| Marker | Meaning | When the bug is fixed |
|---|---|---|
| `@pytest.mark.skip` | Don't run this at all | Stays skipped; nobody notices |
| `@pytest.mark.skipif(cond)` | Don't run under these conditions | Runs again when the condition changes |
| `@pytest.mark.xfail` | Run it; it's expected to fail | Reports **XPASS** — tells you the bug is fixed |

Prefer `xfail` for known bugs. It's the only one that tells you when to close the
ticket.

## 3. `conftest.py`

`conftest.py` holds fixtures and hooks shared without importing. pytest finds it
automatically, and directory nesting means scope:

```text
tests/
  conftest.py            # fixtures for every test
  api/
    conftest.py          # only for tests/api/**
    test_orders.py
  ui/
    conftest.py          # only for tests/ui/**
    test_checkout.py
```

A nested `conftest.py` can override a fixture of the same name from a parent —
useful, and occasionally baffling when you forget it's there. `pytest --fixtures
tests/api/test_orders.py` prints every fixture visible to that file and where it
came from.

## 4. Custom command-line options

You need `--env staging` and `--browser firefox` more often than you'd expect.
Two hooks give you both.

```python
# conftest.py
import pytest


def pytest_addoption(parser):
    parser.addoption("--env", action="store", default="staging",
                     choices=["local", "staging", "prod"],
                     help="target environment")
    parser.addoption("--browser", action="store", default="chrome",
                     help="browser for UI tests")


@pytest.fixture(scope="session")
def env(request):
    return request.config.getoption("--env")


@pytest.fixture(scope="session")
def base_url(env):
    return {
        "local": "http://localhost:8000",
        "staging": "https://staging.example.test",
        "prod": "https://www.example.test",
    }[env]
```

```text
$ pytest --env local -m api
$ pytest --browser firefox -m ui
```

Wrapping the option in a fixture (rather than calling `request.config.getoption`
in every test) means one place to change when the option is renamed.

## 5. Hooks worth knowing

```python
# conftest.py

def pytest_collection_modifyitems(config, items):
    """Auto-mark every test under tests/ui/ as ui — no decorator needed."""
    for item in items:
        if "/ui/" in str(item.fspath):
            item.add_marker(pytest.mark.ui)


def pytest_configure(config):
    """Register markers in code, if you'd rather not list them in the ini file."""
    config.addinivalue_line("markers", "slow: takes more than 5 seconds")
```

The `pytest_runtest_makereport` hook is the one that makes screenshot-on-failure
work; module 07 uses it.

## 6. Plugins worth installing

```bash
pip install pytest-xdist pytest-html pytest-rerunfailures pytest-cov pytest-timeout
```

| Plugin | Gives you | Covered in |
|---|---|---|
| `pytest-xdist` | Parallel execution (`-n auto`) | Module 08 |
| `pytest-html` | Self-contained HTML report | Module 07 |
| `allure-pytest` | Rich report with steps and attachments | Module 07 |
| `pytest-rerunfailures` | `--reruns 2` for genuinely flaky externals | — |
| `pytest-cov` | Coverage measurement | — |
| `pytest-timeout` | Kills a test that hangs past N seconds | — |
| `pytest-randomly` | Randomises order — exposes inter-test coupling | — |
| `pytest-sugar` | Nicer progress output | — |

`pytest --version` lists everything installed, and the run header does too — the
fastest way to answer "why does CI behave differently from my machine".

!!! warning "`--reruns` is a painkiller, not a cure"
    Re-running a failure until it passes hides real race conditions. Use it only
    for provably external flakiness (a third-party sandbox), and log every rerun
    so the count is visible. A test that needs `--reruns 3` is a bug report.

## Cheat sheet

| Task | Command / setting |
|---|---|
| Run a marker | `pytest -m smoke` |
| Exclude a marker | `pytest -m "not slow"` |
| Match by name | `pytest -k "login and not admin"` |
| Stop at first failure | `pytest -x` |
| Stop after N failures | `pytest --maxfail=3` |
| Re-run only last failures | `pytest --lf` |
| Failures first, then the rest | `pytest --ff` |
| List tests without running | `pytest --collect-only -q` |
| Show available fixtures | `pytest --fixtures` |
| Show why a fixture ran | `pytest --setup-show` |
| Full traceback / one line | `--tb=long` / `--tb=line` |
| Show print output | `pytest -s` |
| Which config was used | Read the `configfile:` line in the header |

## Exercise

1. Create a `pytest.ini` with `testpaths`, `addopts = -ra --strict-markers`, and
   three markers. Confirm the run header prints your `configfile` and
   `testpaths`.
2. Misspell a marker on one test and confirm `--strict-markers` turns it into an
   error. Then misspell it in a `-m` expression instead and record the exit code
   and the "N deselected" line — explain why CI could go green on zero tests.
3. Convert a `@pytest.mark.skip` on a known bug into `@pytest.mark.xfail`, then
   fix the bug and record what pytest reports instead of PASSED.
4. Add `--env` via `pytest_addoption` plus a `base_url` fixture that maps it to
   three URLs. Run the same test against two environments without editing any
   test file.
5. Write a `pytest_collection_modifyitems` hook that auto-marks everything under
   `tests/ui/` as `ui`, then verify with `pytest -m ui --collect-only -q` that
   the right tests were selected without a single decorator.
