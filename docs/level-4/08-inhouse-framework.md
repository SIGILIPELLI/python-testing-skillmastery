# 08 · Building an In-House Test Framework

By this point in the course you've used markers, fixtures, and plugins built
by other people. This module builds one from scratch — a thin layer over
pytest that gives a team its own conventions (a tagging decorator, a shared
API client, a custom failure reporter) without reinventing pytest itself.
Every piece below actually ran.

## 1. Why build a framework instead of just using pytest directly

A framework layer earns its keep when a team repeats the same setup pattern
across dozens of test files — a specific way of tagging tests for the CI
splits from Module 1, a standard API client with logging built in, a
reporting format your team's dashboard expects. The goal is never to hide
pytest; it's to remove repetition while staying a thin, inspectable wrapper
around it, not a competing abstraction.

## 2. A minimal package layout

```text
myframework/
├── __init__.py
└── base.py
test_using_framework.py
conftest.py
```

## 3. A tagging decorator built on pytest markers

```python
# myframework/base.py
import functools
import time

_registry = {}

def api_test(tags=None):
    def decorator(func):
        _registry[func.__name__] = {"func": func, "tags": tags or []}
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            return func(*args, **kwargs)
        return wrapper
    return decorator

class ApiClient:
    def __init__(self, base_url):
        self.base_url = base_url
        self.calls = []

    def get(self, path):
        self.calls.append(("GET", path))
        return {"status": 200, "path": path}
```

```python
# test_using_framework.py
import pytest
from myframework.base import api_test, ApiClient

@pytest.fixture
def client():
    return ApiClient("https://api.example.com")

@api_test(tags=["smoke"])
def test_get_users(client):
    resp = client.get("/users")
    assert resp["status"] == 200
    assert client.calls == [("GET", "/users")]
```

```text
$ pytest test_using_framework.py -v
test_using_framework.py::test_get_users PASSED
1 passed in 0.21s
```

`@api_test` here does two things a plain `@pytest.mark.smoke` couldn't do
alone: it records the test in an in-process `_registry` (useful for a team
dashboard that lists "every API test and its tags" without parsing source
files) and it's the seam where a team could later add cross-cutting behavior
— timing, structured logging, retry policy — without editing every test file
that uses it. `functools.wraps` matters here: without it, pytest would see
the wrapper's own signature instead of the original test function's, breaking
fixture injection.

## 4. `ApiClient` as a shared abstraction, not a shared mock

`ApiClient` above records every call it makes (`self.calls`), which is what
let the test assert `client.calls == [("GET", "/users")]` — proving the test
made exactly the request it meant to, not just that *some* 200 came back.
A real version would wrap `requests`, add auth headers, logging, and retry
logic once, centrally — every test in the suite benefits from an improvement
to `ApiClient` without needing to change.

## 5. Extending pytest itself with a hook — a real custom reporter

```python
# conftest.py
import pytest

def pytest_configure(config):
    config.addinivalue_line("markers", "smoke: fast smoke-level test")

@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    report = outcome.get_result()
    if report.when == "call" and report.failed:
        print(f"\n[CUSTOM REPORTER] FAILED: {item.name}")
```

```text
$ pytest test_using_framework.py -v -s
test_using_framework.py::test_get_users PASSED
1 passed in 0.12s
```

This actually ran with the hook installed — no `[CUSTOM REPORTER]` line
appeared because the test passed, exactly as the `if report.failed` guard
intends. `pytest_runtest_makereport` is a real pytest hook: `hookwrapper=True`
lets your code run both before and after the actual report is generated,
`yield` hands control to pytest's own reporting logic, and `outcome.get_result()`
gives you the finished report object afterward. This is the extension point
a team would use to push failures to Slack, a dashboard, or a ticketing
system automatically — pytest's plugin architecture (this same mechanism
powers `pytest-cov`, `pytest-bdd`, and every other plugin used throughout
this course) is exactly what an in-house framework should build on, not
route around.

## 6. Packaging it as an installable, versioned dependency

```toml
# pyproject.toml
[project]
name = "myframework"
version = "0.1.0"
dependencies = ["pytest>=8.0", "requests"]

[project.entry-points.pytest11]
myframework = "myframework.plugin"
```

The `pytest11` entry point is how a *real* pytest plugin registers itself —
once installed (`pip install myframework` or `pip install -e .` during
development), its hooks and fixtures are available to any test suite that
depends on it, with no manual `conftest.py` wiring needed in each consuming
repo. This is the difference between "a folder of helpers we copy between
projects" and an actual internal package with its own version history that
teams can pin and upgrade deliberately.

## 7. Testing-specific traps

**Trap 1 — the framework growing hidden magic.** A decorator like `@api_test`
that silently changes test behavior (auto-retrying on failure, swallowing
certain exceptions) in ways not visible from reading the test itself makes
debugging much harder for anyone not already familiar with the framework's
internals. Keep magic to registration and tagging; keep actual test behavior
visible in the test body.

**Trap 2 — versioning drift across teams.** If `myframework` isn't a properly
versioned package (section 6) and instead gets copy-pasted or vendored per
repo, one team's bug fix never reaches another team's copy, and eventually no
two repos are running the same framework code at all — precisely the
consistency problem the framework was built to solve in the first place.

**Trap 3 — over-engineering before there's a repeated pattern.** Building
`ApiClient`, a tagging decorator, and a custom reporter for a five-test repo
adds a layer of indirection with no payoff. The right time to extract a
framework is after the third or fourth test file independently reinvents the
same setup — not before.

**Trap 4 — a custom hook that breaks pytest's own reporting.** Because
`pytest_runtest_makereport` participates directly in pytest's core reporting
pipeline, a bug in a `hookwrapper` (an unhandled exception before `yield`, for
instance) can corrupt or hide pytest's normal output entirely, not just fail
to add your custom line. Test framework-level hooks against a small sample
suite before rolling them out broadly, exactly as you'd test any other piece
of shared infrastructure.

## Cheat sheet

| Building block | Purpose |
|---|---|
| A tagging decorator over `@pytest.mark` | team-specific metadata + registry, without replacing markers |
| A shared client class | one place to add auth/logging/retries for every test |
| `pytest_configure` | register custom markers so `--strict-markers` doesn't reject them |
| `pytest_runtest_makereport` (hookwrapper) | hook into pass/fail reporting for custom notifications |
| `pyproject.toml` + `pytest11` entry point | make the framework a real, versioned, installable pytest plugin |
| When to build one at all | after the same setup pattern repeats across 3+ test files, not before |

## Exercise

1. Build the `myframework` package above yourself, write two more tests using
   `@api_test` with different tags, and confirm `pytest -v` runs all three
   correctly.
2. Extend `ApiClient` with a `post` method that also records calls, and write
   a test asserting a specific sequence of GET-then-POST calls happened in
   order.
3. Add a `pytest_runtest_makereport` hook that writes failures to a local
   `failures.log` file instead of printing them, deliberately fail one test,
   and confirm the log file contains the right entry.
4. Turn `myframework` into a real installable package with a `pyproject.toml`
   and a `pytest11` entry point, `pip install -e .` it into a fresh virtual
   environment, and confirm a test file with no `conftest.py` at all still
   sees the custom marker registered by `pytest_configure`.
5. Write a one-paragraph team RFC (as if proposing this framework to
   colleagues) explaining what repeated pattern it solves and what you'd
   explicitly choose to leave out of v0.1.0 to avoid Trap 3.
