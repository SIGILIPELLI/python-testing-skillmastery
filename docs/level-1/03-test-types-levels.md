# 03 · Test Types & Levels

"Testing" is not one activity. Ask a team what they test and you'll get answers
that operate at different altitudes and answer different questions. This module
gives you the two axes that organise all of it: **levels** (how much of the
system is under test) and **types** (what quality attribute you're examining).

## 1. Test levels — how much of the system

Levels are stacked. Each one assumes the level below it did its job.

| Level | Unit under test | Who usually writes it | Speed | Environment | Finds |
|---|---|---|---|---|---|
| **Unit** | One function/class in isolation | Developers | Milliseconds | Local / CI | Logic errors, edge cases in a single component |
| **Integration** | Two or more components talking | Developers + QA | Seconds | CI with test doubles or real deps | Interface mismatches, wrong data contracts |
| **System** | The whole assembled application | QA | Seconds to minutes | Dedicated test environment | End-to-end flow failures, config problems |
| **Acceptance (UAT)** | The whole application, from business perspective | Business users / product owner | Minutes to hours | Staging / pre-prod | "It works, but it's not what we asked for" |

### Unit testing

Tests one thing with its dependencies replaced by stubs or mocks. If it touches
a database, a network, or the filesystem, it's usually not a unit test.

Characteristics: fast, deterministic, numerous (hundreds to thousands), owned by
whoever wrote the code. This is what you'll write with pytest in Module 7.

### Integration testing

Verifies the *seams*. Two components can each be perfectly correct and still
fail together — one sends a date as `DD/MM/YYYY`, the other parses `MM/DD/YYYY`;
one returns `null`, the other doesn't expect it.

Two approaches:

- **Big bang** — assemble everything and test. Cheap to organise, miserable to
  debug, because a failure could originate anywhere.
- **Incremental** — add one component at a time. *Top-down* uses **stubs** to
  stand in for not-yet-integrated lower components; *bottom-up* uses **drivers**
  to invoke lower components before the upper ones exist. Slower to set up, far
  faster to diagnose.

### System testing

The first level where you test the product as the user will meet it: real
application, real configuration, realistic data. Both functional and
non-functional types (section 3) are exercised here.

### Acceptance testing / UAT

Run by or on behalf of the business to decide whether to accept the release.
Crucially, UAT tests against **business need**, not the specification — which is
why it catches "you built exactly what the document said, and the document was
wrong." Related forms:

- **Alpha testing** — by internal staff, in a controlled environment.
- **Beta testing** — by real customers, in their own environment.
- **Contract / regulatory acceptance** — verifying contractual or legal criteria.
- **Operational acceptance (OAT)** — can operations actually run this thing?
  Backup, restore, failover, monitoring, runbooks.

!!! info "The V-Model view"
    Each development stage on the left has a matching test level on the right:
    requirements ↔ acceptance testing, system design ↔ system testing, detailed
    design ↔ integration testing, code ↔ unit testing. It's a useful mental
    model even on Agile teams: it reminds you *which artefact* each level is
    validating against.

## 2. The test pyramid

Levels have an ideal proportion, usually drawn as a pyramid:

```text
            /\
           /  \        End-to-end / UI tests   (few, slow, brittle, high confidence)
          /----\
         /      \      Integration / API tests (some, moderate speed)
        /--------\
       /          \    Unit tests              (many, fast, cheap, precise)
      /____________\
```

The rationale is economic. A unit test that fails points at one function. An
end-to-end test that fails could mean anything from a genuine bug to a slow
network to a changed CSS class. Push checks as far down as they can meaningfully
go, and reserve the top of the pyramid for a small set of critical user journeys.

!!! warning "The ice-cream cone"
    The common anti-pattern is the inverted pyramid: hundreds of UI tests, few
    unit tests. Symptoms are a suite that takes hours, fails randomly, and that
    everyone has learned to re-run rather than trust. If you inherit one, the
    fix is not "add more UI tests."

## 3. Functional vs non-functional testing

The second axis: **what quality attribute** are you examining?

| | Functional | Non-functional |
|---|---|---|
| Question | *What* does the system do? | *How well* does it do it? |
| Derived from | Requirements, user stories | Quality attributes, SLAs, standards |
| Pass criteria | Behaviour matches expectation | Measured value meets a threshold |
| Example | "Transfer moves ₹500 from A to B" | "Transfer completes in under 2 s at 500 concurrent users" |

### Functional types

| Type | Verifies |
|---|---|
| **Smoke** | The build is stable enough to test at all |
| **Sanity** | A specific fix or change works, narrowly |
| **Regression** | Existing functionality still works after a change |
| **Integration** | Components work together |
| **UI/GUI** | Screens render and behave correctly |
| **Localisation / i18n** | Language, currency, date formats, RTL layout |
| **Compatibility** | Browsers, OS versions, devices, screen sizes |

### Non-functional types

| Type | Verifies | Typical tool |
|---|---|---|
| **Performance** | Response time, throughput under expected load | Locust, JMeter, k6 |
| **Load** | Behaviour at expected peak load | Locust |
| **Stress** | Behaviour *beyond* capacity, and how it recovers | Locust |
| **Soak / endurance** | Stability over hours — memory leaks, connection exhaustion | Locust |
| **Spike** | Sudden traffic surge handling | Locust |
| **Scalability** | Does adding resources actually add capacity? | Cloud load tooling |
| **Security** | Auth, authorisation, injection, data exposure | OWASP ZAP, Burp |
| **Usability** | Can a real user complete the task without help? | Moderated sessions |
| **Accessibility** | WCAG conformance, screen readers, keyboard nav | axe, Lighthouse |
| **Reliability / availability** | Uptime, failover, recovery | Chaos tooling |
| **Compliance** | GDPR, HIPAA, PCI-DSS, regional requirements | Audit checklists |

You'll meet performance testing with Locust in Level 3 and security basics in
Level 4.

## 4. Smoke vs sanity vs regression

These three get mixed up in nearly every interview, so be precise.

| | Smoke | Sanity | Regression |
|---|---|---|---|
| **Purpose** | Is the build testable? | Did this specific fix work? | Did anything else break? |
| **Scope** | Wide, very shallow | Narrow, deeper | Wide and deep |
| **When** | Immediately on receiving a build | After a bug fix or small change | After any change, before release |
| **Documented?** | Yes, a fixed short checklist | Often ad-hoc, unscripted | Yes, a maintained suite |
| **Duration** | 5–15 minutes | 15–30 minutes | Hours (manual) / minutes (automated) |
| **On failure** | **Reject the build** — don't waste a testing cycle | Reopen the defect | Log a regression defect, usually high priority |
| **Automate?** | Always | Rarely | Always, as far as possible |

A concrete example. A build arrives claiming to fix "discount not applied to
sale items."

- **Smoke:** app loads, login works, product page renders, item adds to cart,
  checkout page opens, payment page opens. Six checks, ten minutes. If checkout
  is broken, reject the build — there's no point testing the discount fix in an
  app that can't check out.
- **Sanity:** apply a discount to a sale item; try two sale items; try a sale
  item plus a full-price item. Narrow, focused on the change.
- **Regression:** the full cart and checkout suite — discounts on non-sale items,
  promo codes, tax calculation, shipping thresholds, currency, order history —
  because a change to discount logic can plausibly break any of them.

!!! tip "The one-sentence distinctions"
    Smoke = *"is it worth testing?"* · Sanity = *"is the fix right?"* ·
    Regression = *"is everything else still right?"*

## 5. Retesting vs regression testing

Another pair that gets conflated:

| | Retesting (confirmation testing) | Regression testing |
|---|---|---|
| Target | The exact failed test case | Other, previously passing cases |
| Trigger | A defect is marked fixed | Any code change |
| Uses the defect's steps? | Yes, exactly | No |
| Can it be automated? | Yes, but often run manually once | Yes — the prime automation candidate |
| Skippable? | Never | Only by explicit risk acceptance |

Both happen after a fix. Retest first (does the fix work?), then regress (did the
fix break a neighbour?).

## 6. Black box, white box, grey box

A third axis: how much internal knowledge do you use?

| Approach | Knowledge of internals | Who | Techniques |
|---|---|---|---|
| **Black box** | None — tests behaviour through the interface | QA, business users | Equivalence partitioning, boundary value analysis, decision tables, state transition (Module 5) |
| **White box** | Full — tests code paths | Developers | Statement, branch, path, condition coverage |
| **Grey box** | Partial — knows architecture, DB schema, API contracts | Experienced QA | Integration and API testing, DB verification |

White-box **coverage criteria**, worth knowing by name:

- **Statement coverage** — every line executed at least once. Weakest.
- **Branch/decision coverage** — every `if` taken both ways.
- **Condition coverage** — every boolean sub-expression evaluated both ways.
- **Path coverage** — every route through the function. Strongest, and
  combinatorially impossible on anything real.

100% statement coverage is a much weaker guarantee than it sounds: code with an
untested `else` branch can still hit 100% statement coverage if the `if` body
contains all the statements. Coverage tells you what was *executed*, never what
was *verified*.

## 7. Choosing what to run

You will never run everything. A workable default for a two-week sprint release:

| Trigger | What runs | Where |
|---|---|---|
| Every commit | Unit tests + linting | CI, automated, < 5 min |
| Every merge to main | Unit + integration + API tests | CI, automated, < 15 min |
| Every deployment to test | Smoke suite | CI, automated, < 5 min |
| Every bug fix | Retest + sanity | Manual or automated |
| Nightly | Full automated regression | CI, automated, any duration |
| Before release | Full regression + exploratory + non-functional spot checks | Mixed |
| After release | Production smoke | Automated against prod |

## Summary table

| Axis | Options |
|---|---|
| **Levels** | Unit → Integration → System → Acceptance |
| **Functional types** | Smoke, sanity, regression, integration, UI, localisation, compatibility |
| **Non-functional types** | Performance, load, stress, soak, security, usability, accessibility, reliability, compliance |
| **Knowledge** | Black box, grey box, white box |
| **Execution** | Manual, automated |
| **Static vs dynamic** | Reviews & analysis vs running the software |

## Exercise

For an online food-delivery application (browse restaurants, add items to cart,
apply coupon, pay, track order):

1. Write **one test case at each of the four levels** for the "apply coupon"
   feature. Make it obvious from the case itself which level it belongs to —
   what's mocked, what's real, who runs it.
2. Design a **10-item smoke suite** for this application. Justify why each item
   is in it, and name two things you deliberately left out despite being
   important.
3. The team fixes "order tracking map does not refresh." Write the **sanity
   checks** (3–5 items) and then list **eight regression areas** you'd cover,
   ranked by risk, with a one-line justification for the top three.
4. List **five non-functional requirements** for this app, each with a
   **measurable threshold** (e.g. "search results render in under 1.5 s at the
   95th percentile with 2,000 concurrent users"). Vague thresholds don't count.
5. The suite currently has 15 unit tests, 4 API tests and 120 UI tests. Diagnose
   the shape, predict three symptoms the team is probably experiencing, and
   propose a rebalancing plan.
