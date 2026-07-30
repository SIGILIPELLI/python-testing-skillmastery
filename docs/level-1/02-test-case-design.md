# 02 · Test Case Design & Documentation

A test case is the unit of work in testing. Written well, anyone on the team can
execute it and get the same result. Written badly, it's a note-to-self that
becomes worthless the moment you leave the project.

This module covers the anatomy of a test case, how to derive cases from
requirements, and how to prove coverage with a traceability matrix.

## 1. Scenario vs case vs script

These three sit at different altitudes and get confused constantly.

| Level | Answers | Granularity | Example |
|---|---|---|---|
| **Test scenario** | *What* to test | One line, no steps | "Verify user can log in" |
| **Test case** | *How* to test it, and what "pass" means | Steps + expected result | "Login with valid credentials", 5 steps, expected: dashboard loads |
| **Test script** | Automated execution of a case | Code | `test_login_valid_credentials()` in pytest |

One scenario typically explodes into 5–15 test cases. "Verify user can log in"
becomes: valid credentials, wrong password, unregistered email, empty fields,
locked account, expired password, case-insensitive email, trailing whitespace,
SQL-injection string, account with 2FA enabled…

That explosion is the actual work of test design.

## 2. The test case template

Use this structure. Every column earns its place.

| Field | Purpose | Example |
|---|---|---|
| **Test Case ID** | Unique, stable reference for reports and defects | `TC-LOGIN-004` |
| **Module / Feature** | Groups cases, enables filtered runs | `Authentication` |
| **Title** | One line, states the condition being tested | `Login fails with unregistered email` |
| **Priority** | Execution order under time pressure | `High` |
| **Type** | Functional / Negative / Boundary / UI / Security | `Negative` |
| **Preconditions** | State that must exist before step 1 | `App is deployed; no account exists for qa.unknown@test.com` |
| **Test Data** | Exact values used | `email: qa.unknown@test.com, password: Valid#123` |
| **Steps** | Numbered, one action each | See below |
| **Expected Result** | Observable, specific, verifiable | `Error "No account found for this email" appears below the email field; user remains on /login` |
| **Actual Result** | Filled at execution time | `As expected` |
| **Status** | Pass / Fail / Blocked / Skipped / Not Run | `Pass` |
| **Requirement ID** | Link back for traceability | `REQ-AUTH-02` |
| **Executed By / Date** | Audit trail | `B. Kumar, 2026-07-30` |

### A fully worked example

**TC-LOGIN-004 — Login fails with unregistered email**

| | |
|---|---|
| Module | Authentication |
| Priority | High |
| Type | Negative |
| Requirement | REQ-AUTH-02 |
| Preconditions | Application reachable at staging URL; no account exists for `qa.unknown@test.com`; browser cache cleared |
| Test Data | Email `qa.unknown@test.com`, Password `Valid#123` |

| Step | Action | Expected Result |
|---|---|---|
| 1 | Navigate to `https://staging.example.com/login` | Login page loads within 3 s; Email, Password fields and "Sign in" button are visible |
| 2 | Enter `qa.unknown@test.com` in the Email field | Value appears; no inline error yet |
| 3 | Enter `Valid#123` in the Password field | Value appears masked as dots |
| 4 | Click **Sign in** | Button shows loading state, then re-enables |
| 5 | Observe the page | Error message "No account found for this email" appears below the Email field; URL is still `/login`; password field is cleared; no session cookie is set |

Note what step 5 does: it states **multiple observable conditions**, including a
negative one (no session cookie). A vague expected result like "login fails" is
the single most common defect in test case writing — it lets two testers reach
opposite conclusions from the same screen.

## 3. Rules for writing steps

1. **One action per step.** "Enter email and password and click Sign in" is
   three steps. When it fails, you need to know *which* action failed.
2. **Use exact values, not descriptions.** "Enter a long name" → "Enter
   `Aaaaa…` (256 characters)". Reproducibility depends on it.
3. **Name UI elements as the user sees them.** "Click the **Sign in** button",
   not "click the submit control".
4. **Expected results must be observable.** "System processes the request" is
   not observable. "Order confirmation number appears in the format `ORD-#####`"
   is.
5. **Independence.** A case should not depend on another case having run first.
   If it truly does, say so in Preconditions and note the dependency.
6. **Include cleanup if the case leaves state behind.** A case that creates a
   user should say how to remove it, or the second run will fail for the wrong
   reason.
7. **No implementation detail in a manual case.** Referring to a database table
   or CSS class ties the case to a version of the system it will outlive.

!!! tip "The stranger test"
    Before you call a test case done, ask: could someone who has never seen this
    application execute it exactly as written and reach an unambiguous
    pass/fail? If not, it isn't finished.

## 4. Positive and negative cases

Every functional requirement needs both, and juniors systematically under-write
the negative side.

- **Positive (happy path):** the system does what it should with valid input.
- **Negative:** the system *refuses* correctly with invalid input, and refuses
  gracefully — a clear message, no crash, no data written, no stack trace leaked.

For a "Register with email and password" requirement:

| ID | Type | Title |
|---|---|---|
| TC-REG-001 | Positive | Register successfully with valid unique email and strong password |
| TC-REG-002 | Negative | Registration rejected when email format is invalid (`user@`, `user@@x.com`, `@x.com`) |
| TC-REG-003 | Negative | Registration rejected when email already registered |
| TC-REG-004 | Negative | Registration rejected when password below minimum length |
| TC-REG-005 | Negative | Registration rejected when required field left empty |
| TC-REG-006 | Boundary | Registration succeeds at exactly the minimum password length |
| TC-REG-007 | Boundary | Registration succeeds at exactly the maximum email length |
| TC-REG-008 | Security | Script tag in the name field is stored/rendered escaped, not executed |
| TC-REG-009 | UI | Password strength indicator updates as the user types |
| TC-REG-010 | Functional | Confirmation email is received within 2 minutes at the registered address |

Ten cases from one sentence of requirement. That ratio is normal.

## 5. Deriving cases from a requirement

Take a real, slightly under-specified requirement:

> **REQ-CART-07:** A customer can apply one promotional code per order. Codes
> are case-insensitive. An expired or invalid code shows an error. Codes cannot
> be applied to orders under ₹500.

Work through it clause by clause. Each clause is a source of cases, and each
*gap* is a question for the business analyst.

| Clause | Cases it generates | Question it raises |
|---|---|---|
| "one promotional code per order" | Apply one code (pass); apply a second code (behaviour?) | Does the second code replace the first, or is it rejected? |
| "case-insensitive" | `SAVE20`, `save20`, `SaVe20` all apply | Is leading/trailing whitespace trimmed? |
| "expired … shows an error" | Apply a code expired yesterday | What's the exact message? Is a code expiring *today* valid? |
| "invalid … shows an error" | Apply a nonsense code; apply an empty code | Same message for expired and invalid, or different? |
| "orders under ₹500" | ₹499 (reject), ₹500 (?), ₹501 (accept) | Is ₹500 exactly "under"? Is the threshold pre- or post-tax? |
| Unstated | Apply code, then remove an item so total drops below ₹500 | Does the discount get revoked? **This is the interesting one.** |

That last row is where testers earn their keep. The requirement never mentions
it, the developer probably didn't implement it, and it's a real bug waiting in
production. **Reading requirements for what they omit is a core skill.**

Log ambiguities as questions rather than guessing — an assumption baked into a
test case becomes a false failure or, worse, a missed defect.

## 6. Priority and severity in test design

At design time you assign **priority** to cases (severity comes later, per
defect — see Module 4).

| Priority | Meaning | Typical cases |
|---|---|---|
| **P1 / Critical** | Must run on every build; blocks release if failing | Login, checkout, payment, data integrity |
| **P2 / High** | Core features; run every regression cycle | Search, profile update, order history |
| **P3 / Medium** | Secondary flows; run per release | Preferences, filters, sorting |
| **P4 / Low** | Cosmetic or rare | Tooltip wording, footer links |

Priority is what you use when the release is tomorrow and you have time for 40
of your 200 cases. If you have never had to make that call, write your
priorities as if you will — because you will.

## 7. Requirements Traceability Matrix (RTM)

An RTM maps requirements to the test cases that verify them, in both directions.
Its purpose is to answer two questions that management will absolutely ask:

- **Forward:** "Is every requirement covered by at least one test?"
- **Backward:** "Does every test trace to a real requirement?" (Tests that
  don't are either scope creep or a sign of an undocumented requirement.)

### Example RTM

| Req ID | Requirement summary | Test Case IDs | Cases | Status | Defects |
|---|---|---|---|---|---|
| REQ-AUTH-01 | User logs in with valid credentials | TC-LOGIN-001, TC-LOGIN-002 | 2 | Pass | — |
| REQ-AUTH-02 | Invalid credentials are rejected with a message | TC-LOGIN-003, TC-LOGIN-004, TC-LOGIN-005 | 3 | 2 Pass, 1 Fail | BUG-118 |
| REQ-AUTH-03 | Account locks after 5 failed attempts | TC-LOGIN-006, TC-LOGIN-007 | 2 | Not Run | — |
| REQ-CART-07 | One promo code per order, min ₹500 | TC-CART-020 … TC-CART-027 | 8 | 6 Pass, 2 Blocked | BUG-121 |
| REQ-CART-08 | Cart persists across sessions | **none** | **0** | **Gap** | — |

The value of the RTM is that last row. `REQ-CART-08` has **zero coverage**, and
the matrix makes that impossible to miss in a status meeting. A coverage gap you
can see is a decision; a coverage gap you can't see is an incident.

Coverage percentage here: 4 of 5 requirements covered = **80% requirement
coverage**. Note this is *requirement* coverage, not code coverage — different
metric, different meaning (Module 3).

!!! info "Keep the RTM cheap"
    An RTM maintained by hand across 800 cases will rot. Most test management
    tools (TestRail, Zephyr, Xray, qTest, or even a well-structured
    spreadsheet) generate it from the links you already record in the
    Requirement ID field. Record the link once, at case-writing time.

## 8. Where test cases live

| Option | Good for | Watch out for |
|---|---|---|
| Spreadsheet (Excel / Google Sheets) | Small teams, fast start, zero cost | No execution history, merge conflicts, no defect linking |
| Test management tool (TestRail, Zephyr, Xray, qTest) | Teams, audit trails, RTM generation, run history | Licence cost; needs discipline to keep tidy |
| In the repo as Markdown/YAML | Version control, review via pull request, lives with the code | Reporting is manual unless you build it |
| Automated tests as documentation | Always current — the test *is* the spec | Only covers what's automated; unreadable to non-technical stakeholders |

Whichever you pick, the fields in section 2 are what you need. The tool is an
implementation detail; the discipline is not.

## Exercise

Using the search feature of any e-commerce site you can access:

1. Write **one test scenario** and derive **eight test cases** from it, using
   the full template from section 2 (all fields, numbered steps, specific
   expected results). Include at least: two positive, four negative, and two
   boundary cases.
2. For each case, assign a **priority** (P1–P4) and justify the two P1s in one
   sentence each.
3. Take this requirement and list **every ambiguity** you'd raise with the
   business analyst before writing cases:

   > *"Search results are sorted by relevance. Users can filter by price range
   > and rating. Searching for a term with no matches shows a suggestion."*

4. Build an **RTM** for the four requirements below against the cases you wrote,
   and identify any coverage gaps:
   `REQ-SRCH-01` keyword search returns matching products ·
   `REQ-SRCH-02` results sortable by price · `REQ-SRCH-03` no-match shows
   suggestions · `REQ-SRCH-04` search history saved for logged-in users.
