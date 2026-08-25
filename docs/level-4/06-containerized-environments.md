# 06 · Containerized Test Environments

Level 3 Module 8 flagged that Postgres-in-Docker testing wasn't runnable in
this sandboxed environment — confirmed again here: `docker` is not installed
in this environment at all. Everything in this module is **reviewed for
correctness, not executed**. The syntax and commands are accurate against
current Docker/testcontainers documentation; validate them on your own
machine (or in a CI runner, which nearly always has Docker available) before
relying on them.

## 1. Why containerize test environments at all

A test suite that needs "a Postgres 16 with these extensions" or "a Redis
instance" has two options: install and manage those services on every
developer's machine and every CI runner by hand, or start disposable,
identical containers for exactly the duration of the test run. Containers
win because they guarantee every environment — a new hire's laptop, CI, a
teammate's machine — runs the *exact same* database version with no manual
setup drift.

## 2. `testcontainers-python` — containers from inside the test itself

```bash
pip install testcontainers[postgres]
```

```python
import pytest
import psycopg2
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def postgres_container():
    with PostgresContainer("postgres:16") as pg:
        yield pg

@pytest.fixture
def db_conn(postgres_container):
    conn = psycopg2.connect(postgres_container.get_connection_url())
    yield conn
    conn.close()

def test_insert_and_query(db_conn):
    db_conn.autocommit = True
    cur = db_conn.cursor()
    cur.execute("CREATE TABLE IF NOT EXISTS items (id SERIAL PRIMARY KEY, name TEXT)")
    cur.execute("INSERT INTO items (name) VALUES (%s)", ("widget",))
    cur.execute("SELECT name FROM items WHERE name = %s", ("widget",))
    assert cur.fetchone() == ("widget",)
```

`PostgresContainer("postgres:16")` pulls (or reuses a cached) real Postgres
16 image, starts it on a random free port, and tears it down when the `with`
block exits — the test never touches SQLite as a stand-in (Level 3 Module 8's
compromise), so type behavior, `JSONB` operators, and constraint enforcement
all match production exactly.

## 3. `docker-compose` for a multi-service integration suite

```yaml
# docker-compose.test.yml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: test
    ports:
      - "5432:5432"

  redis:
    image: redis:7
    ports:
      - "6379:6379"

  api:
    build: .
    depends_on:
      - postgres
      - redis
    environment:
      DATABASE_URL: postgresql://postgres:test@postgres:5432/postgres
      REDIS_URL: redis://redis:6379
    ports:
      - "8000:8000"
```

```bash
docker compose -f docker-compose.test.yml up -d
pytest tests/integration -v
docker compose -f docker-compose.test.yml down -v
```

This is the shape used when tests need several real services wired together
— the API container, its real database, and its real cache — closer to
production topology than any single-service fixture can simulate, at the
cost of a slower startup than an in-process fake.

## 4. Running the whole test suite itself inside a container

```dockerfile
# Dockerfile.test
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["pytest", "-v", "--junitxml=report.xml"]
```

```bash
docker build -f Dockerfile.test -t myapp-tests .
docker run --rm -v $(pwd)/reports:/app/reports myapp-tests
```

Building the test runner itself as an image guarantees the Python version,
OS libraries, and dependency versions are pinned identically everywhere the
image runs — eliminating an entire category of "works on my machine" reports
that don't reproduce in CI (or vice versa).

## 5. CI integration

```yaml
# .github/workflows/tests.yml
jobs:
  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install -r requirements.txt
      - run: pytest tests/integration -v
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/postgres
```

GitHub Actions' `services:` block is a lighter-weight alternative to
`testcontainers` when the CI runner (which does have Docker, even though this
sandbox doesn't) can start the service directly — the `options:` health check
ensures the job waits for Postgres to actually accept connections before
tests run, avoiding a race where the first test connects before the database
is ready.

## 6. Testing-specific traps

**Trap 1 — starting a fresh container per test instead of per session.**
`PostgresContainer` startup takes real seconds; doing it once per test
instead of once per session (as in section 2's `scope="session"` fixture)
turns a fast suite into an unbearably slow one. Share the container across
tests and reset *data* between tests (truncate tables, or wrap each test in a
transaction that rolls back) instead of restarting the container.

**Trap 2 — port collisions on a shared CI runner.** Hardcoding
`ports: ["5432:5432"]` fails if two jobs run concurrently on the same host.
`testcontainers` avoids this by binding to a random free port automatically;
raw `docker-compose` setups need either randomized host ports or one runner
per job.

**Trap 3 — forgetting the health-check race.** A container reporting
"started" and a database being ready to accept connections are different
moments — testcontainers' `PostgresContainer` waits for a real connection to
succeed before yielding, but a naive `docker run` + immediate `pytest` in a
custom script can race ahead of the database actually being ready, causing
intermittent (and confusingly "flaky," Level 3 Module 9-style) connection
failures.

**Trap 4 — not cleaning up containers on test failure.** A test that raises
an exception before reaching a manual `container.stop()` leaks a running
container. Context managers (`with PostgresContainer(...) as pg:`) or
pytest's fixture teardown (which runs even after a test failure) are the
correct way to guarantee cleanup — never rely on manual stop calls placed
after the code that might fail.

## Cheat sheet

| Need | Tool |
|---|---|
| One real service from inside a test | `testcontainers-python` |
| Multiple services, integration suite | `docker-compose.test.yml` |
| Reproducible test runner itself | `Dockerfile.test` + `docker build`/`run` |
| CI-native lightweight service | GitHub Actions `services:` block |
| Avoid port collisions | let testcontainers pick a random port |
| Avoid container-per-test slowness | session-scoped container, reset data per test |
| Avoid startup races | wait for a real health check / connection, not just "container started" |

## Exercise

*(Requires Docker — not available in the environment this module was written
in; verify each step on your own machine.)*

1. Install `testcontainers[postgres]`, write the fixture and test from
   section 2, and run it — confirm it downloads the Postgres image on first
   run and reuses it on subsequent runs.
2. Add a second test using the same session-scoped container, and add a
   per-test cleanup (`TRUNCATE items`) so the two tests don't see each
   other's rows.
3. Write a `docker-compose.test.yml` with Postgres and Redis, bring it up,
   run a quick integration test against both, and tear it down with
   `down -v` (confirm the volume-removal flag actually deletes data by
   restarting and checking the table is gone).
4. Write a `Dockerfile.test`, build it, and run it with a volume-mounted
   `reports/` directory so `report.xml` ends up on your host filesystem
   after the container exits.
5. Wire a GitHub Actions workflow using the `services:` block from section 5
   against a real repo, and confirm in the Actions log that the health check
   waits for Postgres before the test step starts.
