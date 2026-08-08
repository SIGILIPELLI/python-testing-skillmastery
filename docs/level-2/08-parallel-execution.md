# 08 · Parallel Execution (pytest-xdist)

A 400-test UI suite at 20 seconds a test is two hours and thirteen minutes. Nobody
runs that per pull request, so it degrades into a nightly job, and by the time it
reports, six more commits have landed and nobody knows which one broke it. Cutting
that to fifteen minutes changes what the suite is *for*.

`pytest-xdist` distributes tests across processes. It is one flag — and a set of
assumptions about your tests that this module is really about.

## 1. Install and run

```bash
pip install pytest-xdist
```

```text
$ pytest test_slow.py -q                    # serial
........                                                                 [100%]
8 passed in 4.11s

$ pytest test_slow.py -q -n 2
bringing up nodes...
........                                                                 [100%]
8 passed in 2.37s

$ pytest test_slow.py -q -n 4
bringing up nodes...
........                                                                 [100%]
8 passed in 1.46s
```

Eight tests sleeping half a second each: 4.11s → 1.46s on four workers. The
scaling isn't perfectly linear because each worker costs roughly 150–300 ms to
start and must re-import your whole test module.

`-n auto` uses one worker per physical CPU:

```text
$ pytest test_slow.py -n auto
plugins: xdist-3.8.0, metadata-3.1.1, html-4.2.0
created: 8/8 workers
8 workers [8 items]

........                                                                 [100%]
============================== 8 passed in 1.26s ===============================
```

Eight workers for eight tests bought only 0.2s over four — because startup cost
now dominates. More workers is not monotonically faster.

## 2. Choosing a worker count

| Suite type | Bottleneck | Sensible `-n` |
|---|---|---|
| Pure unit tests | CPU | `auto` (= CPU count) |
| API tests | Network latency | 2–4× CPU count — they're mostly waiting |
| Selenium/browser | RAM + CPU per browser | 1 per ~2 GB free RAM, often 4–8 |
| Anything hitting one shared DB | The database | Whatever the DB tolerates; often 2–4 |

`-n auto` is a reasonable default; `-n logical` uses hyperthreads. For browser
suites, measure — each Chrome instance is 300–500 MB, and a machine that starts
swapping will produce timeouts that look exactly like application bugs.

## 3. The real requirement: independence

xdist assigns tests to workers in a way you do not control. Any test that depends
on another test having run first will fail — intermittently, and differently on
each run.

```python
# ✗ Passes serially, fails under -n 4
user_id = None

def test_create_user():
    global user_id
    user_id = api.post("/users", json={...}).json()["id"]

def test_user_appears_in_list():
    assert user_id in [u["id"] for u in api.get("/users").json()]
```

Under xdist these land on different workers — different *processes*, with
different memory. `user_id` is `None` in the second one. Module-level state is
not shared, ever.

```python
# ✓ Each test creates what it needs
def test_user_appears_in_list(make_user):
    user = make_user()
    assert user.id in [u["id"] for u in api.get("/users").json()]
```

The `make_user` factory fixture from module 05 is exactly the tool for this. In
practice, **making a suite parallel-safe is mostly a test-data problem.**

Other shared resources that break under parallelism:

| Shared thing | Symptom | Fix |
|---|---|---|
| A fixed username/email | Duplicate-key errors, random 409s | Sequence or UUID per test (module 05) |
| A hard-coded port | `Address already in use` | Bind port 0, or derive from `worker_id` |
| One temp file path | Truncated/garbled content | pytest's `tmp_path` — already per-test |
| One log/report file | Interleaved or lost lines | Suffix with `worker_id` |
| A single Selenium Grid slot | Tests queue, then time out | Raise grid capacity or lower `-n` |
| Global env vars set by a test | Random unrelated failures | `monkeypatch.setenv`, never `os.environ[...] =` |

## 4. `worker_id` — per-worker resources

xdist provides a `worker_id` fixture (`"gw0"`, `"gw1"`, … or `"master"` when
running serially).

```python
# conftest.py
import pytest


@pytest.fixture(scope="session")
def db_name(worker_id):
    """Give each worker its own database so writes can't collide."""
    if worker_id == "master":
        return "test_db"
    return f"test_db_{worker_id}"


@pytest.fixture(scope="session")
def log_path(worker_id, tmp_path_factory):
    return tmp_path_factory.mktemp("logs") / f"run_{worker_id}.log"
```

!!! warning "'session' scope is per worker, not per run"
    With `-n 4`, a `scope="session"` fixture runs **four times** — once in each
    process. If it creates a database schema, seeds reference data, or starts a
    server, you now have four of them racing. For genuinely once-per-run setup,
    use a file lock:
    ```python
    @pytest.fixture(scope="session")
    def schema(tmp_path_factory, worker_id):
        if worker_id == "master":
            return create_schema()
        root = tmp_path_factory.getbasetemp().parent
        marker = root / "schema.done"
        with FileLock(str(marker) + ".lock"):     # pip install filelock
            if not marker.exists():
                create_schema()
                marker.write_text("done")
        return True
    ```

## 5. Distribution modes

```bash
pytest -n 4 --dist load        # default: next test to the next free worker
pytest -n 4 --dist loadscope   # all tests in a class/module stay together
pytest -n 4 --dist loadfile    # all tests in a file stay together
pytest -n 4 --dist worksteal   # idle workers steal queued tests
```

`loadscope` and `loadfile` are the escape hatch when a module-scoped fixture is
genuinely expensive (one login, one seeded dataset) — they keep the tests that
share it in the same process. You trade some parallelism for far fewer setups.
`worksteal` helps when test durations vary wildly and one worker would otherwise
finish early and idle.

## 6. Debugging a parallel failure

Parallel output is interleaved and progress percentages jump around, which makes
failures harder to read. The workflow:

```bash
pytest -n 4                        # fails
pytest --lf                        # re-run just the failures, SERIALLY
```

- **Fails serially too** → a real bug. Fix it normally.
- **Passes serially** → an isolation problem, not an application bug. Something
  is shared.

To find *what* is shared:

```bash
pip install pytest-randomly
pytest -p randomly                 # random order, single process
```

If it fails in a random serial order, you've reproduced the coupling without
parallelism — much easier to debug. Fix it there, and the parallel failure goes
with it.

!!! warning "xdist and `-s` don't mix"
    `pytest -n 4 -s` gives you four processes writing to one terminal. Use
    logging with `log_cli=true` (module 06) instead, or debug the failing test
    serially.

!!! warning "`pdb` cannot attach to a worker"
    `--pdb` under `-n` will not give you a usable prompt. Reproduce serially
    first, then debug.

## 7. Is it actually faster?

Measure before you tune. `--durations` tells you where the time really goes:

```bash
pytest --durations=10
```

If two tests take 90 seconds each and the other 300 take 50 ms, parallelism buys
you almost nothing — fixing those two does. Common wins that beat adding workers:

- Log in once via API and inject the session cookie, instead of driving the login
  form in every UI test.
- Move assertions that don't need a browser down to the API layer (module 03).
- Replace a slow third-party call with a mock (module 04).

## Cheat sheet

| Need | Flag |
|---|---|
| N workers | `-n 4` |
| One per CPU | `-n auto` |
| Include hyperthreads | `-n logical` |
| Keep a module together | `--dist loadfile` |
| Keep a class together | `--dist loadscope` |
| Rebalance dynamically | `--dist worksteal` |
| Stop the whole run early | `--maxfail=1` (xdist honours it) |
| Which worker am I? | `worker_id` fixture |
| Re-run failures serially | `pytest --lf` (no `-n`) |
| Find slow tests | `--durations=10` |
| Expose hidden coupling | `pytest -p randomly` |

## Exercise

1. Write eight tests that each `time.sleep(0.5)`. Record the wall time at `-n 1`,
   `-n 2`, `-n 4`, and `-n auto`, and explain why the last step gains so little.
2. Write the module-level `user_id` anti-pattern from section 3. Confirm it
   passes serially and fails under `-n 4`, then fix it with a factory fixture
   from module 05.
3. Add a `scope="session"` fixture that prints "SETUP" and run with `-n 4`.
   Count how many times it prints, and explain the result in one sentence.
4. Use the `worker_id` fixture to write each worker's log to its own file.
   Confirm you get `gw0`–`gw3` files with no interleaved lines.
5. Take a suite that fails under `-n 4`, reproduce the failure serially with
   `pytest -p randomly`, and state which shared resource was responsible.
