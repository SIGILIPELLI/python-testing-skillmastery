# 03 · API Testing with requests + pytest

A browser test that checks "the order total shows ₹1,499" spends thirty seconds
launching Chrome, logging in, and clicking through a cart — to verify one number
the server computed. The API test that checks the same number takes 200
milliseconds and fails with a message that points at the actual service. UI tests
prove the screen works; API tests prove the *system* works, and they're the layer
where most of your automated coverage should live.

This module uses `requests` plus plain pytest. No special framework is needed —
an HTTP response is just an object with a status code, headers, and a body.

## 1. Your first API test

Every example below runs against the free
[JSONPlaceholder](https://jsonplaceholder.typicode.com) service, so you can copy
the file and run it right now.

```python
# test_api.py
import requests

BASE_URL = "https://jsonplaceholder.typicode.com"


def test_get_post_returns_200():
    response = requests.get(f"{BASE_URL}/posts/1", timeout=10)
    assert response.status_code == 200
    assert response.headers["Content-Type"].startswith("application/json")
```

Three things are already worth noticing:

- **`timeout=10` is not optional.** `requests` has *no* default timeout. Omit it
  and a hung service hangs your CI job forever, not for ten seconds.
- Assert on `status_code`, not on `response.ok` — `ok` is true for anything under
  400, so a 302 redirect passes a check you meant as "the resource exists".
- Headers are part of the contract. A service that starts returning `text/html`
  (a login page, a proxy error) will still hand you a 200.

## 2. A session fixture

Creating a fresh TCP connection per request is wasteful, and every test repeating
`headers={"Accept": "application/json"}` is the same duplication POM fixed for UI
tests. A `requests.Session` fixture solves both.

```python
import pytest
import requests

BASE_URL = "https://jsonplaceholder.typicode.com"


@pytest.fixture(scope="session")
def api():
    with requests.Session() as session:
        session.headers.update({"Accept": "application/json"})
        yield session
```

Session scope means one connection pool for the whole run. Auth headers, a base
retry policy, and TLS settings all belong here too:

```python
@pytest.fixture(scope="session")
def api(auth_token):
    with requests.Session() as session:
        session.headers.update({
            "Accept": "application/json",
            "Authorization": f"Bearer {auth_token}",
        })
        yield session
```

## 3. Asserting on the body

Status codes tell you the request was accepted. The body tells you the answer was
right. Check both — and check the *shape*, not just one field, so a service that
silently drops a key fails loudly.

```python
def test_post_payload_shape(api):
    body = api.get(f"{BASE_URL}/posts/1", timeout=10).json()
    assert set(body) == {"userId", "id", "title", "body"}
    assert body["id"] == 1
    assert isinstance(body["title"], str) and body["title"]
```

`set(body) == {...}` catches both a missing key and an unexpected new one. If you
only want "these keys must exist, extras are fine", use
`assert {"id", "title"} <= set(body)` instead — pick deliberately, because the
two choices catch different regressions.

## 4. Writes, query parameters, and errors

```python
def test_missing_post_returns_404(api):
    response = api.get(f"{BASE_URL}/posts/999999", timeout=10)
    assert response.status_code == 404


def test_create_post(api):
    payload = {"title": "regression sweep", "body": "nightly", "userId": 7}
    response = api.post(f"{BASE_URL}/posts", json=payload, timeout=10)
    assert response.status_code == 201
    created = response.json()
    assert created["title"] == payload["title"]
    assert "id" in created


def test_query_filter(api):
    response = api.get(f"{BASE_URL}/comments", params={"postId": 1}, timeout=10)
    comments = response.json()
    assert response.status_code == 200
    assert len(comments) == 5
    assert all(c["postId"] == 1 for c in comments)
```

Use `json=payload`, not `data=payload`. `json=` serialises the dict *and* sets
`Content-Type: application/json`; `data=` form-encodes it and many APIs will
reject that with a confusing 400.

Build query strings with `params={...}` rather than f-stringing them into the
URL — `requests` handles the URL-encoding, so a search term containing `&` or a
space doesn't quietly corrupt the request.

## 5. Parametrizing across resources

```python
import pytest


@pytest.mark.parametrize("post_id", [1, 2, 3])
def test_each_post_belongs_to_a_user(api, post_id):
    body = api.get(f"{BASE_URL}/posts/{post_id}", timeout=10).json()
    assert body["userId"] >= 1
```

## 6. Real run

```text
============================= test session starts ==============================
platform darwin -- Python 3.11.2, pytest-9.1.1, pluggy-1.6.0
collected 9 items

test_api.py::test_get_post_returns_200 PASSED                            [ 11%]
test_api.py::test_post_payload_shape PASSED                              [ 22%]
test_api.py::test_missing_post_returns_404 PASSED                        [ 33%]
test_api.py::test_create_post PASSED                                     [ 44%]
test_api.py::test_each_post_belongs_to_a_user[1] PASSED                  [ 55%]
test_api.py::test_each_post_belongs_to_a_user[2] PASSED                  [ 66%]
test_api.py::test_each_post_belongs_to_a_user[3] PASSED                  [ 77%]
test_api.py::test_query_filter PASSED                                    [ 88%]
test_api.py::test_response_time_under_budget PASSED                      [100%]

============================== 9 passed in 3.69s ===============================
```

Nine API tests in 3.7 seconds — including nine real network round trips. The
equivalent coverage through the UI would run several minutes.

## 7. Failure messages that actually help

A bare `assert response.status_code == 200` tells you the number was wrong but
nothing about *why*. When a test fails at 3 a.m. in CI, you want the body:

```python
def test_create_order(api):
    response = api.post(f"{BASE_URL}/posts", json={"title": "x"}, timeout=10)
    assert response.status_code == 201, (
        f"POST /posts returned {response.status_code}\n"
        f"body: {response.text[:500]}"
    )
```

```text
E   AssertionError: POST /orders returned 422
E     body: {"error":"validation_failed","details":[{"field":"userId","msg":"required"}]}
E   assert 422 == 201
```

That message diagnoses the bug without a single re-run.

## 8. Timing as an assertion

`response.elapsed` is a `timedelta` covering the time from sending the request to
finishing the headers — free performance monitoring on tests you already run.

```python
def test_response_time_under_budget(api):
    response = api.get(f"{BASE_URL}/posts", timeout=10)
    assert response.elapsed.total_seconds() < 2.0
```

Keep the budget loose (a smoke-level "not catastrophically slow" check). A tight
budget in a shared CI runner is a flaky test, not a performance test — real load
testing belongs in a dedicated tool, not your functional suite.

## 9. Traps

!!! warning "Tests that depend on each other's writes"
    ```python
    created_id = None  # module-level state — don't

    def test_create():
        global created_id
        created_id = api.post(...).json()["id"]

    def test_delete():
        api.delete(f"{BASE_URL}/posts/{created_id}")
    ```
    Run these with `-k test_delete`, or in parallel (module 08), or in a random
    order, and `created_id` is `None`. Each test must create what it needs — use
    a fixture that creates *and cleans up* its own record.

!!! warning "`.json()` on a non-JSON response"
    When a gateway returns an HTML error page, `response.json()` raises
    `requests.exceptions.JSONDecodeError` and the traceback says nothing about
    the 502 that caused it. Assert the status code *before* parsing the body —
    the order of your assertions is part of your error message.

!!! warning "Asserting on data you don't control"
    `assert len(response.json()) == 100` against a shared staging database
    passes until somebody adds a row. Assert on invariants (every returned
    comment has `postId == 1`), or on records your own fixture created.

## Cheat sheet

| Task | Code |
|---|---|
| GET with params | `session.get(url, params={"q": "shoes"}, timeout=10)` |
| POST JSON | `session.post(url, json=payload, timeout=10)` |
| Auth for all requests | `session.headers.update({"Authorization": ...})` |
| Parse body | `response.json()` — after checking the status code |
| Raw body for error messages | `response.text` |
| Fail on 4xx/5xx immediately | `response.raise_for_status()` |
| Response duration | `response.elapsed.total_seconds()` |
| Redirect history | `response.history` |
| Never omit | `timeout=` |

| Status | Means | Typical test |
|---|---|---|
| 200 | OK | Read succeeded, body matches |
| 201 | Created | POST returned the new resource + id |
| 204 | No content | DELETE succeeded, body is empty |
| 400 | Bad request | Malformed payload rejected |
| 401 / 403 | Unauthenticated / forbidden | Token missing vs. token lacks the role |
| 404 | Not found | Unknown id |
| 422 | Validation failed | Well-formed but semantically invalid input |
| 429 | Rate limited | Client backs off rather than hammering |

## Exercise

Using `https://jsonplaceholder.typicode.com`:

1. Write the `api` session fixture from section 2 and port all your tests to it.
   Confirm with `-v` that the fixture is created once, not once per test.
2. Write a test for `GET /users/1` that asserts the full key set of the nested
   `address` object, then delete one key from your expected set and read the
   failure message pytest produces for set comparison.
3. Write a fixture that POSTs a new post, yields its id, and DELETEs it in
   teardown. Use it in two tests and confirm neither leaks data into the other.
4. Deliberately drop `timeout=` from one request and point it at
   `https://httpbin.org/delay/10`. Record how long the test hangs, then add the
   timeout back and record the exception name you get instead.
5. Write a negative test for `POST /posts` with an empty body, and give its
   assertion a message that prints the status code and the first 500 characters
   of the response so the failure is diagnosable without re-running.
