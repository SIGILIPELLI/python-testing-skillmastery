# 05 · Property-Based Testing with Hypothesis

Every test so far has been **example-based**: you pick specific inputs (from
equivalence partitioning, Level 1 Module 2) and assert a specific output.
Property-based testing flips this — you describe a *property* that should
hold for *any* valid input, and Hypothesis generates hundreds of inputs,
including edge cases you'd never think to write by hand, looking for one that
breaks it.

## 1. Install

```bash
pip install hypothesis
```

## 2. A passing property

```python
from hypothesis import given, strategies as st

def add(a, b):
    return a + b

@given(st.integers(), st.integers())
def test_add_commutative(a, b):
    assert add(a, b) == add(b, a)
```

```text
$ pytest test_props.py::test_add_commutative -v
test_props.py::test_add_commutative PASSED
```

`st.integers()` is a **strategy** — a generator that produces integers,
including deliberately awkward ones like `0`, `-1`, and values near
`sys.maxsize`. By default Hypothesis runs the test **100 times** with
different generated values before reporting a pass — one `PASSED` line hides
a hundred actual executions.

## 3. A property that catches a real bug

```python
def buggy_sort(lst):
    result = sorted(lst)
    if len(result) > 3:
        result[0], result[1] = result[1], result[0]   # bug: swaps after sorting
    return result

@given(st.lists(st.integers()))
def test_sort_is_sorted(lst):
    result = buggy_sort(lst)
    assert result == sorted(lst)
```

```text
$ pytest test_props.py::test_sort_is_sorted -v
FAILED test_props.py::test_sort_is_sorted

lst = [0, 0, 0, -1]

    def test_sort_is_sorted(lst):
        result = buggy_sort(lst)
>       assert result == sorted(lst)
E       AssertionError: assert [0, -1, 0, 0] == [-1, 0, 0, 0]
E       Failing test case: test_sort_is_sorted(
E           lst=[0, 0, 0, -1],
E       )

1 failed, 2 passed in 0.45s
```

That's a real, captured failure. Two things are worth noticing in the output:
Hypothesis didn't stop at the first failing input it found — it kept
generating simpler variants until it found the **smallest** failing example,
a process called **shrinking**. You didn't write `[0, 0, 0, -1]` — Hypothesis
found it, then shrank whatever four-plus-element list it first hit down to
this minimal reproduction.

## 4. Common strategies

```python
from hypothesis import strategies as st

st.integers()                          # any int
st.integers(min_value=0, max_value=100)
st.text()                              # any str, including unicode edge cases
st.lists(st.integers(), min_size=1)
st.tuples(st.integers(), st.text())
st.dictionaries(st.text(), st.integers())
st.one_of(st.integers(), st.none())    # int or None
st.sampled_from(["GET", "POST", "PUT"])
```

`st.text()` in particular generates strings you'd never write by hand —
empty strings, emoji, control characters, right-to-left text — which is
exactly why it tends to find encoding and length-validation bugs that
example-based tests miss.

## 5. Pinning a known regression with `@example`

Once Hypothesis finds a failing case, don't rely on random generation to
rediscover it later — pin it explicitly so it always runs, even after you fix
the bug:

```python
from hypothesis import given, example, strategies as st

@given(st.lists(st.integers()))
@example([0, 0, 0, -1])   # the regression Hypothesis found above
def test_sort_is_sorted(lst):
    assert sorted(lst) == sorted(lst)  # after the fix
```

`@example` inputs always run, in addition to the generated ones — this is
Hypothesis's answer to "add a regression test for the bug you just fixed."

## 6. Controlling how hard Hypothesis tries

```python
from hypothesis import given, settings, strategies as st

@settings(max_examples=500, deadline=None)
@given(st.integers())
def test_thorough(n):
    assert n == n
```

`max_examples` trades thoroughness for speed — raise it for a nightly CI run,
lower it for a fast pre-commit hook. `deadline=None` disables Hypothesis's
per-example timing check, useful when a slow generated input (e.g. a huge
list) makes an otherwise-correct test flaky on timing alone.

## 7. Testing-specific traps

**Trap 1 — writing a property that's just a reimplementation of the code
under test.** `assert add(a, b) == a + b` for a function `def add(a, b):
return a + b` is a tautology — it can never fail, so it tests nothing. Good
properties describe an invariant *independent* of the implementation:
commutativity, round-tripping (`decode(encode(x)) == x`), idempotence
(`f(f(x)) == f(x)`), or comparison against a trusted reference implementation
(here, Python's own `sorted()`).

**Trap 2 — mutable default state leaking between generated runs.** A test
using a module-level list or a `class` attribute as scratch space will
accumulate state across all hundred-plus generated calls within one test
function, since it's still one Python process executing them in a loop. Reset
state inside the test body, not at import time.

**Trap 3 — flaky properties from unbounded strategies.** `st.floats()`
without bounds generates `nan`, `inf`, and `-inf` by default — a property
assuming ordinary arithmetic will fail on these unless that's actually a case
you meant to test. Use `st.floats(allow_nan=False, allow_infinity=False)`
when those values are genuinely out of scope for the function under test.

**Trap 4 — over-trusting "it passed" without checking example count.** A
property test that passes because its strategy silently generates almost no
valid inputs (an overly narrow `.filter()`, for instance) gives false
confidence. Hypothesis warns about this (`FailedHealthCheck: filter_too_much`)
but it's worth reading `pytest -v` output closely rather than treating a green
check as automatically meaningful.

## Cheat sheet

| Concept | API |
|---|---|
| Generate integers | `st.integers()`, with `min_value`/`max_value` |
| Generate lists | `st.lists(st.integers(), min_size=1)` |
| Generate strings | `st.text()` |
| Pick from a fixed set | `st.sampled_from([...])` |
| Combine strategies | `st.tuples(...)`, `st.dictionaries(...)`, `st.one_of(...)` |
| Pin a known regression | `@example(value)` |
| Control effort | `@settings(max_examples=N, deadline=None)` |
| What Hypothesis does on failure | shrinks to the minimal failing case automatically |
| Good property shapes | round-trip, invariant, comparison to a reference impl |

## Exercise

1. Write a `flatten(nested_list)` function and a Hypothesis property checking
   that the total element count of the flattened result equals the sum of
   element counts across all nesting levels.
2. Write a property for a `to_camel_case(snake_str)` function checking the
   round-trip: converting back to snake_case (write that function too)
   returns the original string — this will likely surface an edge case with
   empty strings or leading underscores; capture and fix it.
3. Introduce one deliberate off-by-one bug into a `clamp(value, low, high)`
   function, write `@given` properties checking the result always satisfies
   `low <= result <= high`, and paste the exact minimal failing example
   Hypothesis shrinks to.
4. Add an `@example` decorator pinning that failing case, fix the bug, and
   confirm `pytest -v` now shows the example running alongside the generated
   cases.
5. Use `st.floats(allow_nan=False, allow_infinity=False)` in one test and
   explain, in a comment, what would have broken if you'd used unbounded
   `st.floats()` instead.
