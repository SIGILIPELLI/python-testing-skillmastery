# 06 · Performance Testing with Locust

Every test so far checks *correctness* — does the right thing happen? Locust
checks *capacity* — how many users, requests, or transactions can the system
handle before it slows down or falls over? It's written in Python, using the
same mental model as the rest of this course: define user behavior as code,
run it, read structured output.

## 1. Install

```bash
pip install locust
```

## 2. A locustfile

```python
# locustfile.py
from locust import HttpUser, task, between

class WebsiteUser(HttpUser):
    wait_time = between(0.1, 0.3)   # pause between tasks, per simulated user

    @task(3)                        # 3x more likely to run than view_page
    def view_homepage(self):
        self.client.get("/")

    @task(1)
    def view_page(self):
        self.client.get("/page")
```

`HttpUser` models one simulated user; Locust spawns many of them concurrently,
each independently picking a `@task` (weighted by the number passed to the
decorator) and waiting `wait_time` between actions — a much closer simulation
of real traffic than firing requests in a tight loop.

## 3. Running it headlessly

```bash
locust -f locustfile.py --host https://example.com \
       --headless -u 3 -r 1 --run-time 5s --csv=results
```

```text
[INFO] Run time limit set to 5 seconds
[INFO] Ramping to 3 users at a rate of 1.00 per second
[INFO] --run-time limit reached, shutting down

Type     Name          # reqs  # fails |   Avg   Min   Max   Med | req/s failures/s
GET      /                  9   0(0%)  |    27    15    66    19 |  5.32       0.00
GET      /page              5  5(100%) |    18    16    19    19 |  2.96       2.96
Aggregated                 14  5(35.71%)|    23    15    66    19 |  8.28       2.96

Error report
# occurrences  Error
5              GET /page: HTTPError('404 Client Error: Not Found for url: /page')
```

That's a real run against `https://example.com` — 3 users, ramped at 1/second,
for 5 seconds. `/page` genuinely doesn't exist on that domain, so Locust
reports exactly what it should: a 100% failure rate on that endpoint, isolated
from the 0% failure rate on `/`. This is the same signal a real load test
gives you against your own app: which specific endpoint degrades or breaks
under concurrent load, not just an aggregate number.

`-u 3 -r 1` means "ramp up to 3 concurrent users, adding 1 per second."
`--headless` skips Locust's web UI (`locust -f locustfile.py --host ...`
without `--headless` starts a browser dashboard on `localhost:8089` instead)
— use headless for CI, the web UI for exploratory runs where you want to
watch response times live and adjust user count on the fly.

## 4. Reading the numbers that matter

- **req/s** — throughput. Compare this across runs with increasing `-u` to
  find where throughput stops scaling with added users — that's your
  system's practical ceiling.
- **Avg / Med / percentiles** — median (p50) tells you the typical
  experience; p95/p99 tell you what your *worst-served* users experience,
  which matters more for SLAs than the average does.
- **failures/s** — a spike here under increasing load is usually the first
  sign of the system exhausting a resource (connection pool, thread pool,
  database connections) rather than CPU.

## 5. Simulating a realistic session, not isolated GETs

```python
from locust import HttpUser, task, between

class ShopperUser(HttpUser):
    wait_time = between(1, 3)

    def on_start(self):
        # runs once per simulated user, before any @task
        response = self.client.post("/login", json={"user": "loadtest", "pw": "x"})
        self.token = response.json().get("token", "")

    @task
    def browse_and_checkout(self):
        headers = {"Authorization": f"Bearer {self.token}"}
        self.client.get("/products", headers=headers)
        self.client.post("/cart", json={"product_id": 1}, headers=headers)
        self.client.post("/checkout", headers=headers)
```

`on_start` runs once when a simulated user "arrives" — the natural place to
log in and capture an auth token, exactly like a `conftest.py` fixture that
sets up state before a test body runs.

## 6. Testing-specific traps

**Trap 1 — treating 0 failures as success without checking response
content.** Locust marks a request "successful" based on HTTP status code by
default. An endpoint returning `200 OK` with an error message in the JSON
body looks fine to Locust unless you explicitly validate it:

```python
with self.client.get("/api/data", catch_response=True) as response:
    if "error" in response.json():
        response.failure("Got 200 but body contained an error")
```

**Trap 2 — load-testing against a shared or production environment by
accident.** `--host` takes whatever URL you give it — pointing a load test at
a shared staging environment other teams depend on, or worse, production, can
cause a real outage. Always confirm `--host` before running, and never load
test a system without the team that owns it knowing in advance.

**Trap 3 — client-side bottleneck, not server-side.** If req/s plateaus and
CPU on the client machine running Locust is pegged at 100%, you're measuring
Locust's own limits, not your server's. Locust supports distributing load
across multiple worker processes/machines (`--worker`/`--master`) for exactly
this reason.

**Trap 4 — `wait_time` too aggressive or too lax.** `between(0.1, 0.3)` above
simulates a very impatient user; real users pause several seconds between
actions. An unrealistically low `wait_time` inflates throughput numbers to
something that doesn't reflect real traffic patterns.

## Cheat sheet

| Concept | Locust API |
|---|---|
| A simulated user | `class MyUser(HttpUser):` |
| Time between actions | `wait_time = between(min, max)` |
| An action, weighted | `@task(weight)` |
| Per-user setup (e.g. login) | `def on_start(self):` |
| Make a request | `self.client.get/post(...)` |
| Validate beyond status code | `catch_response=True` + `response.failure(...)` |
| Run without the web UI | `--headless -u N -r RATE --run-time Ns` |
| Machine-readable output | `--csv=prefix` produces `prefix_stats.csv` etc. |
| Distribute load | `--master` / `--worker` |

## Exercise

1. Write a locustfile with two `@task`s at different weights against a
   public test API of your choice (or `https://example.com`), and run it
   headlessly for 10 seconds at `-u 5 -r 1`.
2. Add `catch_response=True` to one task and fail the request explicitly if
   the response takes longer than 1 second
   (`if response.elapsed.total_seconds() > 1: response.failure(...)`).
3. Re-run at `-u 20` and compare req/s and the failures/s column against the
   `-u 5` run — write two sentences on what changed and why.
4. Add an `on_start` method that performs a fake "login" request before the
   weighted tasks run, and confirm (via `--csv`) that it appears as its own
   row in the stats output.
5. Deliberately point `--host` at a URL with a path that doesn't exist (like
   `/page` above) for one task, and paste the exact Error report line Locust
   generates — this is what a real load test surfaces when a load-tested
   deploy has a broken route.
