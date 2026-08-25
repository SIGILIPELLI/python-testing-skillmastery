# 08 · Database Testing from Python

Every mocking technique in Level 2 Module 4 exists partly to *avoid* hitting a
real database in a unit test. This module is about the tests that
deliberately do hit one — verifying schema constraints, transaction behavior,
and data integrity that no amount of mocking can substitute for. The examples
below use SQLite's built-in `:memory:` mode, which needs no server and ran
directly against the standard library.

## 1. An in-memory database per test

```python
# db.py
import sqlite3

def get_connection(path=":memory:"):
    conn = sqlite3.connect(path)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY,
            email TEXT UNIQUE NOT NULL,
            balance INTEGER NOT NULL DEFAULT 0
        )
    """)
    conn.commit()
    return conn
```

```python
# conftest.py
import pytest
from db import get_connection

@pytest.fixture
def conn():
    c = get_connection(":memory:")
    yield c
    c.close()
```

`:memory:` gives every test a fresh, empty, disk-free database — the database
equivalent of the fresh-browser-per-test pattern from Level 1. No cleanup
step is needed beyond `close()`; the data simply disappears with the
connection.

## 2. Testing constraints, not just happy paths

```python
def create_user(conn, email, balance=0):
    conn.execute("INSERT INTO users (email, balance) VALUES (?, ?)", (email, balance))
    conn.commit()
```

```python
import sqlite3

def test_create_user(conn):
    create_user(conn, "a@test.com", 100)
    row = conn.execute(
        "SELECT email, balance FROM users WHERE email=?", ("a@test.com",)
    ).fetchone()
    assert row == ("a@test.com", 100)

def test_duplicate_email_rejected(conn):
    create_user(conn, "dup@test.com", 10)
    with pytest.raises(sqlite3.IntegrityError):
        create_user(conn, "dup@test.com", 20)
```

```text
$ pytest test_db.py -v
test_db.py::test_create_user PASSED
test_db.py::test_duplicate_email_rejected PASSED
```

`test_duplicate_email_rejected` is the important one here — it verifies the
`UNIQUE` constraint defined in the schema actually fires, not just that your
Python code behaves. A unit test with a mocked database would never catch a
missing or mistyped `UNIQUE` clause in a real migration.

## 3. Testing transaction integrity

```python
def transfer(conn, from_email, to_email, amount):
    cur = conn.execute("SELECT balance FROM users WHERE email = ?", (from_email,))
    row = cur.fetchone()
    if row is None or row[0] < amount:
        raise ValueError("insufficient funds")
    conn.execute("UPDATE users SET balance = balance - ? WHERE email = ?", (amount, from_email))
    conn.execute("UPDATE users SET balance = balance + ? WHERE email = ?", (amount, to_email))
    conn.commit()
```

```python
def test_transfer_success(conn):
    create_user(conn, "a@test.com", 100)
    create_user(conn, "b@test.com", 0)
    transfer(conn, "a@test.com", "b@test.com", 40)
    a = conn.execute("SELECT balance FROM users WHERE email=?", ("a@test.com",)).fetchone()[0]
    b = conn.execute("SELECT balance FROM users WHERE email=?", ("b@test.com",)).fetchone()[0]
    assert (a, b) == (60, 40)

def test_transfer_insufficient_funds(conn):
    create_user(conn, "a@test.com", 10)
    create_user(conn, "b@test.com", 0)
    with pytest.raises(ValueError):
        transfer(conn, "a@test.com", "b@test.com", 40)
    a = conn.execute("SELECT balance FROM users WHERE email=?", ("a@test.com",)).fetchone()[0]
    assert a == 10   # unchanged — the failed transfer touched nothing
```

```text
test_db.py::test_transfer_success PASSED
test_db.py::test_transfer_insufficient_funds PASSED

4 passed in 0.31s
```

The second test's final assertion is the real point: after a failed transfer,
account `a`'s balance must be exactly what it started as. This checks that
the *validation* (`raise ValueError` before any `UPDATE`) actually prevents
partial writes — a genuine transaction bug would show a debited sender with
no corresponding credit to the recipient.

## 4. Testing against Postgres/MySQL locally with a test container

SQLite is a fine stand-in for exercising your own SQL logic, but it doesn't
enforce everything a production Postgres or MySQL server does (foreign keys
are enforced differently, some datatypes and functions differ). For
integration tests that must match production exactly, spin up the real engine
in a disposable container:

```python
import pytest
from testcontainers.postgres import PostgresContainer
import psycopg2

@pytest.fixture(scope="session")
def postgres_url():
    with PostgresContainer("postgres:16") as pg:
        yield pg.get_connection_url()

@pytest.fixture
def pg_conn(postgres_url):
    conn = psycopg2.connect(postgres_url)
    yield conn
    conn.close()
```

This module's `pip install testcontainers` and the actual container startup
were not run here — Docker isn't available in this environment (confirmed
absent when checked). Treat this pattern as reviewed-correct-syntax, not
executed; validate it in an environment with Docker before relying on it, and
prefer it over SQLite whenever a production-specific behavior (foreign key
cascade rules, `JSONB` operators, isolation levels) is what you're actually
testing.

## 5. Testing-specific traps

**Trap 1 — SQLite passing a test that a real engine would fail.** SQLite is
dynamically typed by default (a column declared `INTEGER` will happily store
text unless `STRICT` mode is enabled). A test asserting type coercion behaves
"correctly" on SQLite may fail identically-looking assertions against
Postgres, which enforces column types strictly. Know which database your
tests are actually validating against, and don't treat SQLite as
production-equivalent for anything type-sensitive.

**Trap 2 — forgetting `conn.commit()` and getting a false pass.** Without a
commit, some drivers still let the *same connection* read back uncommitted
writes within one test, making the test pass — while a genuinely separate
connection (as production code would use) would see nothing. Always commit
explicitly and, for extra confidence, verify with a second connection where
that matters.

**Trap 3 — test pollution from shared fixtures at the wrong scope.** A
`scope="session"` database fixture reused across tests without a
per-test rollback or cleanup step accumulates rows from every earlier test,
and later tests' assertions ("there is exactly one user") become
order-dependent. Prefer function-scoped connections (as in section 1) unless
you have an explicit transaction-per-test rollback strategy.

**Trap 4 — testing the ORM's behavior instead of your query.** A test using
an ORM's high-level API (e.g. SQLAlchemy's `session.query(...)`) can pass
while masking a raw-SQL bug that only surfaces when a report or migration
runs hand-written SQL directly. When a query's correctness genuinely matters,
test it at the same level it will actually run in production.

## Cheat sheet

| Need | Pattern |
|---|---|
| Fast, disk-free test DB | `sqlite3.connect(":memory:")` |
| Fresh DB per test | function-scoped `conn` fixture, no shared state |
| Verify a constraint fires | `pytest.raises(sqlite3.IntegrityError)` |
| Verify transaction atomicity | assert unrelated fields are unchanged after a failed operation |
| Match production engine exactly | `testcontainers` spinning up real Postgres/MySQL in Docker |
| Avoid false passes from same-connection reads | commit, then read back with a separate connection |
| Avoid cross-test pollution | function-scoped fixtures unless deliberately testing rollback behavior |

## Exercise

1. Extend `db.py` with a `delete_user(conn, email)` function and write tests
   for: successful deletion, deleting a non-existent email (decide and assert
   what should happen), and that deleting one user doesn't affect another's
   balance.
2. Add a `CHECK (balance >= 0)` constraint to the `users` table schema and
   write a test proving a direct `UPDATE ... SET balance = -5` is rejected
   with `sqlite3.IntegrityError`.
3. Write a test that runs `transfer` between two accounts inside a
   `try/except`, deliberately triggers a `ValueError` partway through by
   passing a nonexistent `from_email`, and confirms neither balance changed.
4. If you have Docker available, install `testcontainers[postgres]` and
   `psycopg2-binary`, adapt section 4's fixtures, and rerun your tests
   against real Postgres — note any assertion that behaves differently than
   it did on SQLite.
5. Deliberately remove `conn.commit()` from `create_user` and describe, based
   on running the existing tests, whether they still pass and why single
   in-process connections can hide a missing commit that a multi-connection
   production system would expose.
