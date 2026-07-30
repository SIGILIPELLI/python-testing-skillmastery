# 05 · Manual Testing in Practice

You can't test everything. This module covers the formal techniques that let you
choose a *small* set of test cases with a *high* chance of finding defects —
equivalence partitioning, boundary value analysis, decision tables, state
transition testing — plus exploratory testing, the skill that finds what your
formal cases never anticipated.

These are black-box techniques: they work from the specification, without seeing
the code.

## 1. Equivalence Partitioning (EP)

**Principle:** divide the input domain into groups where every value in a group
should be treated identically by the system. Test one value from each group.

If `25` and `26` go down exactly the same code path, testing both adds cost and
no information.

### Worked example — age field on an insurance application

> *Requirement: applicants must be 18–65 inclusive. Under 18 → "Not eligible —
> minimum age is 18." Over 65 → "Not eligible — maximum age is 65." Non-numeric
> input → "Enter a valid age."*

| Partition | Range / description | Valid? | One representative value |
|---|---|---|---|
| P1 | Below minimum | Invalid | `10` |
| P2 | Eligible range | **Valid** | `35` |
| P3 | Above maximum | Invalid | `72` |
| P4 | Zero / negative | Invalid | `-5` |
| P5 | Non-numeric text | Invalid | `abc` |
| P6 | Empty | Invalid | `` (blank) |
| P7 | Decimal | Invalid (probably — *ask*) | `25.5` |
| P8 | Whitespace-padded number | Valid? (*ask*) | `  35  ` |

Eight tests instead of the 100+ you'd get by trying every age. And note P7 and
P8: partitioning surfaces specification gaps as a side effect.

### Partitioning non-numeric inputs

EP is not only for numbers. Partition by *how the system treats* the value:

- **Country dropdown** → domestic / EU / rest of world / sanctioned country /
  none selected.
- **File upload** → allowed type, disallowed type, no extension, zero-byte file,
  file over the size limit, correct extension but corrupt contents.
- **Payment method** → credit card, debit card, wallet, expired card, card
  requiring 3-D Secure, card issued in a different currency.
- **User role** → anonymous, standard, premium, admin, suspended.

!!! warning "Partitions must be genuinely equivalent"
    If `35` and `55` hit different premium-calculation brackets, they are *not*
    in the same partition, however similar the input looks. Partition by
    **behaviour**, never by appearance.

## 2. Boundary Value Analysis (BVA)

**Principle:** defects cluster at the edges of partitions, because that's where
`<` gets written instead of `<=`. Test the boundary and its immediate neighbours.

For each boundary, test three values: **min−1, min, min+1** and **max−1, max,
max+1**. (The two-value variant tests just the pair straddling each boundary.)

### Worked example — the same 18–65 age field

| Value | Partition | Expected result | Why it's here |
|---|---|---|---|
| `17` | Below min | Rejected — "minimum age is 18" | min − 1 |
| **`18`** | Valid | **Accepted** | min — the classic off-by-one |
| `19` | Valid | Accepted | min + 1 |
| `64` | Valid | Accepted | max − 1 |
| **`65`** | Valid | **Accepted** | max — the classic off-by-one |
| `66` | Above max | Rejected — "maximum age is 65" | max + 1 |

Six cases. Bugs found by this exact pattern in the wild: an application rejecting
18-year-olds because the code says `age > 18` instead of `age >= 18`. It is the
single highest-yield technique in black-box testing.

### Worked example — a text field with a length limit

> *Requirement: the "Display name" field accepts 3 to 30 characters.*

| Length | Input | Expected |
|---|---|---|
| 2 | `ab` | Rejected — "at least 3 characters" |
| **3** | `abc` | **Accepted** |
| 4 | `abcd` | Accepted |
| 29 | 29 chars | Accepted |
| **30** | 30 chars | **Accepted** |
| 31 | 31 chars | Rejected — "at most 30 characters" |

And the boundary-adjacent questions worth asking: is the limit enforced on the
client, the server, or both? Does the field *truncate* at 30 rather than reject?
Do emoji count as one character or four? Does a trailing space count?

### Boundaries hide everywhere

| Domain | Boundaries worth testing |
|---|---|
| Numbers | 0, 1, −1, max int, floating-point rounding at `0.005` |
| Strings | empty, 1 char, exactly the limit, limit+1, only whitespace |
| Dates | 1st and last day of month, 29 Feb in a leap year, 28→1 Mar in a non-leap year, DST change day, year end, timezone midnight |
| Collections | empty list, 1 item, page-size boundary (e.g. 10, 11 items with 10-per-page), max items |
| Money | 0.00, 0.01, the free-shipping threshold exactly, currency rounding on 1/3 splits |
| Time | session timeout at exactly the limit, token expiry, request timeout |

## 3. Combining EP and BVA

The two techniques are complementary and used together: EP tells you *which*
groups exist; BVA tells you *where within each group* to test.

**Worked example — a shipping-cost rule.**

> *Orders under ₹500: ₹60 shipping. ₹500–₹1,999: ₹30 shipping. ₹2,000 and above:
> free shipping. Orders cannot exceed ₹1,00,000.*

Partitions (EP): `< 500` · `500–1999` · `2000–100000` · `> 100000` · invalid
(negative, zero, non-numeric).

Boundaries (BVA) at 500, 2000 and 100000:

| Order value | Expected shipping | Technique |
|---|---|---|
| `250` | ₹60 | EP — mid-partition |
| `499` | ₹60 | BVA — min−1 |
| **`500`** | **₹30** | BVA — the boundary |
| `501` | ₹30 | BVA — min+1 |
| `1200` | ₹30 | EP — mid-partition |
| `1999` | ₹30 | BVA — max−1 |
| **`2000`** | **Free** | BVA — the boundary |
| `2001` | Free | BVA — max+1 |
| `99999` | Free | BVA — max−1 |
| `100000` | Free | BVA — the boundary |
| `100001` | Rejected | BVA — max+1 |
| `0` | Rejected / empty cart | EP — invalid |
| `-100` | Rejected | EP — invalid |

Thirteen cases give near-complete confidence in a rule that has, in principle,
ten million valid inputs. That is what test design buys you.

## 4. Decision tables

When output depends on a **combination** of conditions, EP and BVA aren't enough
— you need to cover the combinations systematically. A decision table lays out
every combination of conditions and the action for each.

### Worked example — loan approval

> *Approve if: credit score ≥ 700 **and** employed **and** existing debt is
> below 40% of income. If the score is 650–699 but the applicant is employed
> with debt below 40%, refer to a manual review. Anything else is rejected.
> Anyone unemployed is rejected regardless of score.*

| Rule | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
|---|---|---|---|---|---|---|---|---|
| **Conditions** | | | | | | | | |
| Credit score ≥ 700 | Y | Y | Y | Y | N | N | N | N |
| Credit score 650–699 | N | N | N | N | Y | Y | Y | N |
| Employed | Y | Y | N | N | Y | Y | N | – |
| Debt < 40% of income | Y | N | Y | N | Y | N | – | – |
| **Actions** | | | | | | | | |
| Approve | ✔ | | | | | | | |
| Refer to manual review | | | | | ✔ | | | |
| Reject | | ✔ | ✔ | ✔ | | ✔ | ✔ | ✔ |

Eight rules → eight test cases, and each one has concrete data attached:

| TC | Score | Employed | Debt ratio | Expected |
|---|---|---|---|---|
| TC-LOAN-01 | 720 | Yes | 25% | Approved |
| TC-LOAN-02 | 720 | Yes | 55% | Rejected |
| TC-LOAN-03 | 720 | No | 25% | Rejected |
| TC-LOAN-04 | 720 | No | 55% | Rejected |
| TC-LOAN-05 | 670 | Yes | 25% | Manual review |
| TC-LOAN-06 | 670 | Yes | 55% | Rejected |
| TC-LOAN-07 | 670 | No | 25% | Rejected |
| TC-LOAN-08 | 610 | Yes | 25% | Rejected |

Combine with BVA on the score boundaries — 649, 650, 699, 700 — and you have a
rigorously covered rule.

!!! info "Collapsing the table"
    Three binary conditions give 2³ = 8 combinations; four give 16. When a
    condition is irrelevant to the outcome (marked `–` above — unemployed
    applicants are rejected whatever their debt ratio), collapse those columns
    into one rule. This keeps the table readable as conditions multiply.

## 5. State transition testing

Use it when the system's response depends on **what happened before** — orders,
accounts, subscriptions, sessions, anything with a status field.

### Worked example — account lockout

> *An account locks after 3 consecutive failed login attempts. A successful
> login resets the counter. A locked account unlocks after 30 minutes or via a
> password reset.*

States: `Active` · `Active (1 fail)` · `Active (2 fails)` · `Locked`.

| Current state | Event | Next state |
|---|---|---|
| Active | Successful login | Active (logged in) |
| Active | Failed login | Active (1 fail) |
| Active (1 fail) | Failed login | Active (2 fails) |
| Active (1 fail) | Successful login | Active (counter reset) |
| Active (2 fails) | Failed login | **Locked** |
| Active (2 fails) | Successful login | Active (counter reset) |
| Locked | Any login attempt | Locked — "Account locked" message |
| Locked | 30 minutes elapse | Active (counter reset) |
| Locked | Password reset completed | Active (counter reset) |

Now test the **invalid transitions** too — these are where the real bugs are:

- Does a successful login on the *third* attempt reset the counter, or does the
  lock happen first?
- Is the counter per-account or per-IP? Two people on the same office network
  shouldn't lock each other out.
- Does the counter reset survive a server restart, or does a deploy silently
  unlock everyone?
- Does the 30-minute timer restart if a locked account is attempted again?
- Can a password reset be triggered *while* locked?

State transition testing is the technique that finds the questions nobody wrote
down.

## 6. Error guessing

An explicitly experience-based technique: use what you know about how software
breaks. Not a substitute for systematic techniques — a supplement after them.

The standard list of things worth trying on almost any input:

- Empty, whitespace-only, and `null`
- Zero, negative numbers, very large numbers, leading zeros (`007`)
- Special characters: `' " < > & % \ / ; -- $ { }`
- Unicode, emoji, right-to-left text, combining accents
- Very long strings (10,000 characters)
- SQL fragments (`' OR '1'='1`) and script tags (`<script>alert(1)</script>`)
- Duplicate submission — double-click the submit button
- Back button after submit; refresh after submit; open the same form in two tabs
- Session expiry mid-form
- Network interruption mid-request
- Copy-paste with hidden formatting from Word
- Values in the *wrong* format that look right (`31/02/2026`, `2026-13-01`)

## 7. Exploratory testing

Scripted testing checks what you already thought of. **Exploratory testing** is
simultaneous learning, test design and execution — you use what you just learned
to decide what to try next. It is not "random clicking"; it's structured
improvisation.

### Session-based test management

The discipline that makes exploratory testing accountable:

1. **Write a charter** — a one-sentence mission for a timeboxed session:

   > *"Explore the coupon-application flow with mixed carts, using expired,
   > case-varied and stacked codes, to discover inconsistencies between the cart
   > total and the order confirmation."*

2. **Timebox it.** 60 or 90 minutes. Longer and focus decays.
3. **Take notes as you go** — what you tried, what you observed, questions
   raised, bugs found, and any *setup* time (which is data about testability).
4. **Debrief** — 10 minutes with a lead or peer: what did you find, what didn't
   you get to, what's the next charter?

### A session sheet

| Field | Content |
|---|---|
| Charter | Explore coupon application on mixed carts |
| Area | Cart / Checkout |
| Build | 2.14.0-rc3 |
| Duration | 90 min (60 test design/execution · 20 bug investigation · 10 setup) |
| Data used | `SAVE20`, `EXPIRED10`, `save20`, ` SAVE20 `, carts of 1/2/5 items |
| Bugs | BUG-118, BUG-121 |
| Issues | Staging seed data lacks sale-flagged items — had to create one manually |
| Questions | Should a code survive removing the item that qualified the cart? |
| Next charter | Coupon behaviour when the cart is modified after application |

### Heuristics to drive a session

Structured prompts stop exploration turning into aimless clicking:

- **Vary the sequence.** Apply the coupon *before* adding items, not after.
- **Interrupt.** Navigate away mid-flow, then return. Refresh. Use the back button.
- **Go to the extremes.** Zero items, one item, fifty items.
- **Break the assumption of one user.** Same account in two browsers, both
  changing the cart.
- **Follow the data.** Does what the cart shows match the order email, the order
  history page, and the admin view?
- **CRUD each entity.** Create, read, update, delete — and delete something
  another screen still references.
- **Test the "and then?"** — after the successful path, what state is left behind?

!!! tip "Exploratory + scripted is not either/or"
    A mature process runs scripted regression (predictable, automatable
    coverage) *and* exploratory sessions (finds the unknown unknowns).
    Organisations that drop exploratory testing because "everything is
    automated" ship bugs that no script was ever going to look for.

## 8. Choosing a technique

| Situation | Technique |
|---|---|
| Input field with a range or set of valid values | Equivalence partitioning + BVA |
| Any limit, threshold, or `<`/`<=` decision | BVA |
| Output depends on a combination of conditions | Decision table |
| Behaviour depends on prior events / status | State transition |
| A brand-new feature you barely understand | Exploratory session |
| A stable feature about to be released | Scripted regression |
| Time is short and risk is unclear | Risk-based prioritisation + exploratory |
| Requirement seems complete but feels wrong | Error guessing + exploratory |

## Exercise

For an online hotel-booking form (check-in date, check-out date, number of
guests 1–8, number of rooms 1–4, promo code, loyalty tier Bronze/Silver/Gold):

1. Write the **equivalence partitions** for *number of guests* and for *promo
   code*, with one representative value each and a note on which partitions the
   requirement fails to define.
2. Do a full **BVA** on both *number of guests* and *number of rooms*, plus a
   date-related BVA set covering: check-in today, check-in yesterday, check-out
   equal to check-in, check-out before check-in, a stay spanning 29 February
   2028, and a stay spanning a year boundary.
3. Build a **decision table** for this rule and derive one test case per rule
   with concrete data:

   > *Free breakfast is included for Gold members on any stay; for Silver
   > members on stays of 3 nights or more; and for anyone booking 2 or more
   > rooms. Bronze members on a 1-night single-room stay pay for breakfast.*

4. Draw the **state transitions** for a booking: `Draft → Confirmed → Checked-in
   → Checked-out`, plus `Cancelled` and `No-show`. List five **invalid
   transitions** you would explicitly test, and what should happen for each.
5. Write **three exploratory charters** for this booking flow, each timeboxed at
   60 minutes and each targeting a different risk. Then run one of them against
   any real booking site and complete a session sheet using the format in
   section 7.
