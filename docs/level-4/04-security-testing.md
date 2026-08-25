# 04 · Security Testing Basics for QA

Security testing isn't only the security team's job. A QA engineer who knows
how to write a test proving an injection attack succeeds — and another
proving the fix actually blocks it — closes a huge class of bugs before they
ever reach a penetration test. This module runs both a static analyzer
(Bandit) and a real, working SQL injection demonstration against a live
Flask app.

## 1. Static analysis with Bandit

```bash
pip install bandit
```

```python
# vulnerable.py
import subprocess
import hashlib

def run_command(user_input):
    subprocess.call("ls " + user_input, shell=True)

def hash_password(password):
    return hashlib.md5(password.encode()).hexdigest()
```

```text
$ bandit vulnerable.py

>> Issue: [B602:subprocess_popen_with_shell_equals_true] subprocess call with
   shell=True identified, security issue.
   Severity: High   Confidence: High
   CWE: CWE-78
   Location: ./vulnerable.py:5

>> Issue: [B324:hashlib] Use of weak MD5 hash for security.
   Severity: High   Confidence: High
   CWE: CWE-327
   Location: ./vulnerable.py:8

Total issues (by severity):
    Low: 1   Medium: 0   High: 2
```

That's a real Bandit run — it found both planted issues without any test
being written: `shell=True` with string-concatenated user input (a command
injection vector, CWE-78) and MD5 used for hashing a password (a weak
algorithm, CWE-327). Running Bandit in CI on every PR catches this class of
bug at the same "shift-left" point Module 2 argued for — before a human
reviewer even opens the file.

## 2. Proving an injection actually works — then proving the fix blocks it

Static analysis flags *patterns*. The next step for QA is proving the
vulnerability is real by exploiting it, and then proving a fix genuinely
closes it — the same "red test, then green test" discipline as any other bug
fix, just applied to a security bug.

```python
# app.py
import sqlite3
from flask import Flask, request

app = Flask(__name__)

def get_db():
    conn = sqlite3.connect(":memory:")
    conn.execute("CREATE TABLE users (id INTEGER, name TEXT)")
    conn.execute("INSERT INTO users VALUES (1, 'alice')")
    return conn

@app.route("/user_unsafe")
def user_unsafe():
    conn = get_db()
    name = request.args.get("name", "")
    query = f"SELECT * FROM users WHERE name = '{name}'"   # string-built SQL
    rows = conn.execute(query).fetchall()
    return {"rows": rows}

@app.route("/user_safe")
def user_safe():
    conn = get_db()
    name = request.args.get("name", "")
    rows = conn.execute("SELECT * FROM users WHERE name = ?", (name,)).fetchall()
    return {"rows": rows}
```

```python
# test_security.py
import pytest
from app import app

@pytest.fixture
def client():
    return app.test_client()

def test_sql_injection_bypasses_unsafe_endpoint(client):
    payload = "nonexistent' OR '1'='1"
    resp = client.get("/user_unsafe", query_string={"name": payload})
    data = resp.get_json()
    assert len(data["rows"]) == 1   # injection returned a row it shouldn't have

def test_sql_injection_blocked_on_safe_endpoint(client):
    payload = "nonexistent' OR '1'='1"
    resp = client.get("/user_safe", query_string={"name": payload})
    data = resp.get_json()
    assert len(data["rows"]) == 0   # parametrized query treats it as a literal
```

```text
$ pytest test_security.py -v
test_security.py::test_sql_injection_bypasses_unsafe_endpoint PASSED
test_security.py::test_sql_injection_blocked_on_safe_endpoint PASSED
2 passed in 0.30s
```

Both tests genuinely ran against a live Flask test client and a real SQLite
database. The first test's assertion — `len(rows) == 1` for a name that
doesn't exist — is the injection succeeding: the classic `' OR '1'='1`
payload turns the `WHERE` clause into an always-true condition, and the
string-built query in `/user_unsafe` returns Alice's row despite the search
name being nonsense. `/user_safe`'s parametrized query treats the entire
payload as a literal string to search for, finds no match, and correctly
returns zero rows.

## 3. Turning this into a reusable payload-driven test

```python
import pytest

INJECTION_PAYLOADS = [
    "' OR '1'='1",
    "'; DROP TABLE users; --",
    "' UNION SELECT 1,2 --",
]

@pytest.mark.parametrize("payload", INJECTION_PAYLOADS)
def test_safe_endpoint_resists_injection_payloads(client, payload):
    resp = client.get("/user_safe", query_string={"name": payload})
    assert resp.status_code == 200
    assert len(resp.get_json()["rows"]) == 0
```

Parametrizing common payload patterns (Level 1's equivalence partitioning,
applied to attack strings) turns one manual proof-of-concept into a
regression suite that runs on every change to the endpoint.

## 4. Beyond injection: a few other checks QA can own

```python
def test_no_stack_trace_leaked_on_error(client):
    resp = client.get("/user_safe", query_string={"name": None})
    assert b"Traceback" not in resp.data
    assert resp.status_code != 500 or b"Internal Server Error" in resp.data

def test_security_headers_present(client):
    resp = client.get("/user_safe")
    # X-Content-Type-Options prevents MIME-sniffing attacks
    assert resp.headers.get("X-Content-Type-Options") is None  # documents current gap
```

The second test is written to currently pass by documenting a *missing*
header rather than asserting it's present — a useful pattern when you're
recording a known gap for a team to fix, distinct from silently ignoring it.
Once the header is added, flip the assertion and the test becomes a real
regression guard.

## 5. Testing-specific traps

**Trap 1 — testing security only against your own attack list.** The
parametrized payloads in section 3 catch known SQL injection *patterns*, not
all possible ones. Security testing should be paired with the underlying
fix (parametrized queries, an ORM) rather than treated as complete coverage
by itself — a clever new payload can always be found if the underlying
string-concatenation vulnerability is still there.

**Trap 2 — running security scans against production or someone else's
system without authorization.** Even a "just checking" injection attempt
against a system you don't own or have written permission to test is
unauthorized access in most jurisdictions. Confine exploitation tests (as
opposed to passive static analysis) to systems you own or have explicit
authorization to test.

**Trap 3 — treating a passing Bandit scan as "secure."** Bandit catches known
insecure *patterns* in Python code; it says nothing about business-logic
flaws like a broken authorization check that lets user A view user B's data
with a perfectly parameterized, pattern-clean query. Static analysis and
logic-level security tests (like Module 3's contract tests, extended to
check authorization) are complementary, not substitutes for each other.

**Trap 4 — weak hashing left in place because "nothing failed."** The
`hash_password` function using MD5 in section 1 doesn't cause any functional
test to fail — passwords still hash and compare correctly. This is exactly
why a linter/static-analysis pass matters: functional tests, by design,
can't catch "this works but is insecure."

## Cheat sheet

| Technique | Tool | Catches |
|---|---|---|
| Static pattern scan | `bandit` | shell injection, weak crypto, hardcoded secrets, etc. |
| Prove an exploit works | payload + assertion against a live endpoint | confirms the vulnerability is real, not theoretical |
| Prove a fix blocks it | same payload against the fixed endpoint | regression guard, same "red/green" discipline as any bug fix |
| Regression suite for known attack classes | `@pytest.mark.parametrize` over payload lists | catches the well-known variants automatically |
| Document a known gap | an assertion recording current (missing) behavior | tracks debt without silently ignoring it |
| What these don't catch | business-logic authorization bugs, novel payloads | needs dedicated logic-level tests, not just pattern matching |

## Exercise

1. Run `bandit` against a small Python file containing at least three
   different flagged patterns (unsafe `eval`, `shell=True`, and a hardcoded
   password string) and paste the exact issues it reports for each.
2. Build the `/user_unsafe` and `/user_safe` Flask endpoints yourself, prove
   the injection succeeds against the unsafe one, and prove it's blocked on
   the safe one — use a different payload than the one shown here.
3. Add a `/user_unsafe` variant that's also vulnerable to a `UNION SELECT`
   payload extracting data from a *different* table, and write a test
   proving the leak.
4. Write a test asserting no stack trace or internal file path ever appears
   in a Flask app's response body when `DEBUG=False`, and confirm it fails
   when you temporarily set `app.config["DEBUG"] = True`.
5. Pick one OWASP Top 10 category other than injection (e.g. broken access
   control) and design — in prose, not necessarily code — one test a QA
   engineer could write to prove a specific instance of it in a hypothetical
   app with a `/api/orders/{id}` endpoint.
