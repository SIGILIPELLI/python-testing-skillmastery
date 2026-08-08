# 05 · Test Data Management & Factories

Most suites don't rot because the assertions were wrong. They rot because of the
data: a hard-coded `qa.buyer01@test.com` that two tests both try to register, a
staging record someone deleted, a test that passes alone and fails in a batch.
Test data is infrastructure, and it needs the same discipline as the code.

The goal of this module: every test creates exactly the data it needs, with
values that can't collide, and leaves the system as it found it.

## 1. Why literals in tests hurt

```python
def test_new_user_gets_welcome_email():
    user = create_user(
        email="qa.buyer01@test.com",
        full_name="Test User",
        country="IN",
        is_active=True,
        roles=[],
    )
    assert welcome_email_sent_to(user.email)
```

Two problems. First, the reader can't tell which of those five fields the test
*cares* about — is the assertion about `country`? about `is_active`? (It's about
none of them; only `email` matters.) Second, run this test twice and the second
run collides on a unique email constraint.

## 2. A factory fixture

The fix is a function that fills in sensible defaults and lets each test override
only the field it's testing.

```python
# conftest.py
import itertools
import pytest

_counter = itertools.count()


@pytest.fixture
def make_user():
    created = []

    def _make_user(**overrides):
        n = next(_counter)
        attrs = {
            "email": f"qa.user{n:03d}@example.test",
            "full_name": "Ada Lovelace",
            "country": "IN",
            "is_active": True,
        }
        attrs.update(overrides)
        user = create_user(**attrs)
        created.append(user)
        return user

    yield _make_user

    for user in created:            # teardown: clean up everything we made
        delete_user(user.id)
```

This is the **factory-as-fixture** pattern: the fixture yields a *callable*, not
a value. Now the test states only what it means:

```python
def test_new_user_gets_welcome_email(make_user):
    user = make_user()
    assert welcome_email_sent_to(user.email)


def test_inactive_user_cannot_log_in(make_user):
    user = make_user(is_active=False)      # the one field this test is about
    assert login(user.email, "any") is False
```

A test that needs two users calls the factory twice — something a plain
`@pytest.fixture def user()` cannot do, because a fixture value is cached per
test.

## 3. `factory_boy` and `Faker`

For anything larger than a handful of models, use the libraries.

```bash
pip install factory_boy Faker
```

```python
import factory
from faker import Faker
from dataclasses import dataclass, field

fake = Faker()
Faker.seed(20250808)


@dataclass
class User:
    email: str
    full_name: str
    country: str = "IN"
    is_active: bool = True
    roles: list = field(default_factory=list)


class UserFactory(factory.Factory):
    class Meta:
        model = User

    email = factory.Sequence(lambda n: f"qa.user{n:03d}@example.test")
    full_name = factory.Faker("name")
    country = "IN"
    is_active = True
```

```python
def test_factory_defaults():
    user = UserFactory()
    assert user.email.endswith("@example.test")
    assert user.is_active is True


def test_factory_override():
    user = UserFactory(country="DE", is_active=False)
    assert user.country == "DE"


def test_sequence_is_unique():
    a, b = UserFactory(), UserFactory()
    assert a.email != b.email


def test_batch():
    users = UserFactory.create_batch(3, country="US")
    assert len({u.email for u in users}) == 3
```

## 4. Real run

```text
User(email='qa.user000@example.test', full_name='Lawrence Bryan', country='IN', is_active=True, roles=[])
User(email='qa.user001@example.test', full_name='Mark Long', country='DE', is_active=False, roles=[])
qa.user002@example.test qa.user003@example.test
qa.user004@example.test Deanna Olsen US
qa.user005@example.test Troy Hanson US
qa.user006@example.test Linda Nelson US

4 passed in 0.04s
```

`Sequence` guarantees the emails never collide. `Faker` supplies names that look
like data instead of `Test User 1`, which matters more than it sounds: realistic
names surface encoding bugs, column-width bugs, and sort-order bugs that
`aaa`/`bbb` never will.

## 5. Seeding: reproducible randomness

`Faker.seed(20250808)` makes every run produce the *same* "random" values. That's
the difference between a test you can debug and one you can't.

| Approach | Finds new bugs | Reproducible | Verdict |
|---|---|---|---|
| Hard-coded literals | No | Yes | Fine for the field under test |
| Unseeded random | Sometimes | **No** | Flaky; failures vanish on re-run |
| Seeded random | Sometimes | Yes | Good default |
| Seeded, seed logged in output | Yes | Yes | Best — vary the seed nightly, log it |

If you *do* run with a varying seed, print it in the test output. A failure you
can't reproduce is a bug report nobody can act on.

## 6. Fixture scope and data leakage

Scope decides how long a fixture's value lives — and getting it wrong is one of
the top causes of order-dependent failures.

| Scope | Created | Use for |
|---|---|---|
| `function` (default) | Once per test | Anything mutable — records, carts, temp dirs |
| `class` | Once per test class | A logged-in state shared by a small group |
| `module` | Once per file | An expensive read-only dataset |
| `session` | Once per run | DB connection pool, HTTP session, auth token |

!!! danger "The classic scope bug"
    ```python
    @pytest.fixture(scope="session")     # ✗
    def cart():
        return Cart()

    def test_add_item(cart):
        cart.add("shoes")
        assert cart.count() == 1

    def test_empty_cart_total_is_zero(cart):
        assert cart.total() == 0         # fails — still holds "shoes"
    ```
    Run `test_empty_cart_total_is_zero` alone and it passes. Run the file and it
    fails. **Session-scope anything mutable and you have coupled every test that
    touches it.** Share expensive *connections* at session scope; create the
    *data* at function scope.

## 7. Cleaning up

Three strategies, in order of preference:

```python
# 1. Transaction rollback — fastest and most complete
@pytest.fixture
def db_session(connection):
    transaction = connection.begin()
    session = Session(bind=connection)
    yield session
    session.close()
    transaction.rollback()          # nothing the test wrote survives
```

```python
# 2. Track-and-delete — when you can't roll back (e.g. tests over a real API)
@pytest.fixture
def make_order(api):
    ids = []

    def _make(**kw):
        order = api.post("/orders", json=kw, timeout=10).json()
        ids.append(order["id"])
        return order

    yield _make

    for order_id in reversed(ids):          # reverse order respects FK deps
        api.delete(f"/orders/{order_id}", timeout=10)
```

```python
# 3. Unique namespacing — when deletion isn't possible at all
run_id = uuid.uuid4().hex[:8]
email = f"qa.{run_id}.{n}@example.test"     # collisions impossible; sweep later
```

Teardown after `yield` runs even when the test fails — but *not* if the fixture
raised before reaching `yield`. Keep setup before `yield` minimal for that
reason.

## 8. Where data should come from

| Source | Good for | Risk |
|---|---|---|
| Inline literals | The one field under test | Collisions, unclear intent |
| Factory fixtures | Almost everything | — |
| JSON/YAML files | Large fixed payloads, contract samples | Drifts from the real schema |
| Production dumps | Realistic volume | **PII exposure** — must be anonymised |
| Seeded shared staging DB | Cross-team demos | Anyone can mutate it; tests couple to it |

!!! warning "Never put real customer data in a test fixture"
    A copied-in production record puts real names, emails, and card fragments
    into your git history permanently. Anonymise at export time, not later.

## Cheat sheet

| Need | Pattern |
|---|---|
| Two of a thing in one test | Factory fixture yielding a callable |
| Guaranteed-unique field | `factory.Sequence` or a `uuid4()` prefix |
| Realistic values | `factory.Faker("name")`, seeded |
| Related object | `factory.SubFactory(UserFactory)` |
| Derived value | `factory.LazyAttribute(lambda o: f"{o.name}@x.test")` |
| Many at once | `Factory.create_batch(10)` |
| Cleanup | `yield`, then delete in reverse creation order |
| Expensive shared resource | `scope="session"` — connections only, never data |
| Temp files | pytest's built-in `tmp_path` fixture |

## Exercise

1. Convert a test that hard-codes five user fields into a `make_user` factory
   fixture. Confirm the test body now names only the field it's asserting on.
2. Write a test that needs *two* distinct users. Show why a plain value fixture
   can't do it, then do it with the factory.
3. Reproduce the section 6 scope bug on purpose: make a mutable fixture
   `scope="session"`, watch one test pass alone and fail in the file, then fix
   the scope and confirm both orders pass.
4. Build a `UserFactory` with `factory_boy` including a `Sequence` email, a
   `Faker` name, and a `LazyAttribute` username derived from the name. Generate
   a batch of 5 and assert every email is unique.
5. Write a factory fixture that creates a record over a real API and deletes it
   in teardown. Make the test fail deliberately (`assert False`) and confirm from
   the API that the cleanup still ran.
