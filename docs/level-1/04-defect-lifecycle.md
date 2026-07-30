# 04 · Defect Lifecycle & Bug Reporting

Finding a bug is half the job. The other half is communicating it so precisely
that a developer can reproduce and fix it without a single follow-up question.
A tester's reputation is built almost entirely on the quality of their defect
reports.

## 1. The defect lifecycle

Every defect moves through a defined set of states. The exact names vary by tool
and team, but the shape is universal.

```text
        NEW
         │  (triage)
         ▼
      ASSIGNED ──────────────► DEFERRED
         │                     REJECTED
         │ (dev fixes)         DUPLICATE
         ▼                     NOT A BUG / WORKS AS DESIGNED
       FIXED
         │  (tester retests)
         ├──── fails ────► REOPENED ──► (back to ASSIGNED)
         │
         └──── passes ───► VERIFIED ──► CLOSED
```

| Status | Meaning | Who moves it | Next |
|---|---|---|---|
| **New** | Just logged, not yet reviewed | Tester | Assigned / Rejected / Duplicate |
| **Assigned** | Triaged and given to a developer | Lead / triage meeting | Fixed / Deferred |
| **Open / In Progress** | Developer is working on it | Developer | Fixed |
| **Fixed** | Code change done, awaiting verification | Developer | Verified / Reopened |
| **Pending Retest** | Fix deployed to test environment | Release / dev | Verified / Reopened |
| **Verified** | Tester confirmed the fix works | Tester | Closed |
| **Reopened** | Fix didn't work or broke something | Tester | Assigned |
| **Closed** | Done, no further action | Tester / lead | — |
| **Deferred** | Real bug, won't fix this release | Product owner | Backlog |
| **Rejected** | Not accepted as a defect | Developer / PO | Closed |
| **Duplicate** | Already reported | Triage | Closed, linked to original |
| **Not a Bug / As Designed** | Behaviour is intentional | Developer / PO | Closed |
| **Cannot Reproduce** | Developer couldn't make it happen | Developer | Back to tester for more detail |

### Rules that keep the lifecycle honest

- **Only the tester who raised it (or QA generally) closes a defect.** A
  developer marking their own bug "Closed" removes the verification step.
- **"Cannot Reproduce" is a request for information, not a verdict.** Respond
  with environment details, a video, exact data, logs — not an argument.
- **Reopen rather than raise a new bug** when a fix is incomplete. It preserves
  the history and keeps the reopen-rate metric honest.
- **Deferred needs an owner and a target.** "Deferred" without a release target
  is how bugs disappear.

## 2. Severity vs priority

The single most-asked interview question in QA, and genuinely useful in practice.

| | Severity | Priority |
|---|---|---|
| Measures | **Technical impact** on the system | **Urgency** of fixing it |
| Set by | Tester | Product owner / business |
| Question | "How badly does this break things?" | "How soon must this be fixed?" |
| Changes over time? | Rarely | Often — a release date changes priority |

They are **independent axes**, which is exactly why all four combinations exist:

| | | Example |
|---|---|---|
| **High severity, high priority** | Breaks core function, everyone hits it | Payment fails for all users at checkout |
| **High severity, low priority** | Breaks badly, almost nobody hits it | App crashes when the profile photo is a 400 MB TIFF |
| **Low severity, high priority** | Cosmetic, but highly visible or embarrassing | Company name misspelt on the homepage; competitor's logo on the login screen |
| **Low severity, low priority** | Minor, rarely seen | Tooltip on an admin-only settings page has a typo |

### Severity scale

| Severity | Definition | Example |
|---|---|---|
| **S1 · Critical / Blocker** | System unusable, data loss, no workaround; blocks testing | Application won't start; orders are silently lost |
| **S2 · Major** | Major function broken, workaround is painful or absent | Coupon never applies; search returns no results for valid terms |
| **S3 · Minor** | Function works but incorrectly; acceptable workaround exists | Sort by price is ascending when descending is selected |
| **S4 · Trivial / Cosmetic** | No functional impact | Misaligned label, inconsistent button shade |

### Priority scale

| Priority | Definition | Expectation |
|---|---|---|
| **P1 · Urgent** | Fix immediately, possibly hotfix | Blocks release or is live in production |
| **P2 · High** | Fix in the current sprint/release | Ship-blocking for this release |
| **P3 · Medium** | Fix in an upcoming release | Backlog with a target |
| **P4 · Low** | Fix when convenient | May never be fixed, and that's a decision |

!!! tip "Set severity honestly"
    Inflating severity to get attention destroys the signal. If everything is
    S1, triage stops working and genuinely critical bugs queue behind cosmetic
    ones. Use priority — which the business controls — as the escalation lever.

## 3. The bug report template

| Field | Purpose |
|---|---|
| **ID** | Auto-generated (e.g. `BUG-118`) |
| **Summary / Title** | One line: *what* fails, *where*, *under what condition* |
| **Severity** | S1–S4, technical impact |
| **Priority** | P1–P4, business urgency |
| **Environment** | Build number, OS, browser+version, device, URL, account/role |
| **Preconditions** | State required before step 1 |
| **Test Data** | Exact values used |
| **Steps to Reproduce** | Numbered, minimal, complete |
| **Expected Result** | What should happen, and the source (requirement ID, spec, mockup) |
| **Actual Result** | What actually happened, precisely |
| **Reproducibility** | Always / Intermittent (n of m attempts) / Once |
| **Attachments** | Screenshot, screen recording, console log, HAR file, server log excerpt |
| **Related** | Requirement ID, failing test case ID, related/duplicate bugs |
| **Reported by / date** | Audit trail |

### Writing the title

The title is read a hundred times more often than the body — in triage, in
standups, in release notes. Format that works:

> **[Area] What fails when what condition**

| Bad title | Why it fails | Better title |
|---|---|---|
| "Login broken" | Which login? What breaks? For whom? | `[Auth] Login fails with 500 error when email contains a "+" alias` |
| "Doesn't work" | Contains no information at all | `[Cart] Promo code SAVE20 is not applied to orders containing sale items` |
| "Bug in checkout page" | Names a location, not a behaviour | `[Checkout] Order total excludes shipping when the delivery address is changed after the coupon is applied` |
| "App crash" | No condition, no scope | `[Profile] App crashes on Android 14 when uploading a photo larger than 20 MB` |

## 4. A complete worked example

---

**BUG-118 — [Cart] Promo code SAVE20 is not applied to orders containing sale items**

| Field | Value |
|---|---|
| **Severity** | S2 · Major |
| **Priority** | P1 · Urgent (campaign launches Monday) |
| **Status** | New |
| **Environment** | Staging build `2.14.0-rc3` · Chrome 127.0.6533.89 · macOS 15.5 · `https://staging.shop.example.com` · Account `qa.buyer01@test.com` (standard customer role) |
| **Reproducibility** | Always — 5 of 5 attempts |
| **Related** | REQ-CART-07 · TC-CART-024 |

**Preconditions**

- Promo code `SAVE20` is active, valid through 2026-08-31, 20% off, minimum order ₹500.
- Product `SKU-99213 "Trail Runner Shoes"` is flagged as a sale item, list price ₹2,400, sale price ₹1,800.
- Cart is empty.

**Steps to Reproduce**

1. Log in as `qa.buyer01@test.com`.
2. Navigate to `/product/SKU-99213`.
3. Click **Add to cart**. Cart badge shows `1`.
4. Click the cart icon, then **View cart**. Subtotal shows `₹1,800.00`.
5. In the **Promo code** field, enter `SAVE20`.
6. Click **Apply**.

**Expected Result**

A 20% discount of ₹360.00 is applied. The cart shows:
`Subtotal ₹1,800.00 · Discount (SAVE20) −₹360.00 · Total ₹1,440.00`,
and the confirmation "Promo code applied" appears.
(Source: REQ-CART-07 — the requirement states one code per order with a ₹500
minimum, and does not exclude sale items.)

**Actual Result**

The error `"This code cannot be applied to your cart"` appears in red below the
promo field. Subtotal and total remain `₹1,800.00`. No discount row is added.
The browser console shows:

```text
POST /api/v2/cart/promo 422 (Unprocessable Entity)
{"error":"PROMO_INELIGIBLE_ITEM","sku":"SKU-99213"}
```

**Additional observations** (narrowing the fault for the developer)

- The same code applies correctly to a cart containing only non-sale items
  (verified with `SKU-88104`, ₹1,200 → ₹960).
- In a **mixed cart** (one sale item + one non-sale item, subtotal ₹3,000) the
  code is also rejected — so the presence of *any* sale item blocks the code,
  rather than the discount simply being restricted to eligible lines.
- Not reproducible on production build `2.13.2` — the code applies correctly
  there. **This is a regression introduced in 2.14.0.**

**Attachments** — `cart-promo-error.png`, `console-422.har`, `screen-recording.mp4`

---

Notice what makes this report good: it isolates the trigger (sale items), it
distinguishes regression from a never-worked feature, it cites the requirement
for the expected result, it includes the API error the developer will grep for,
and it eliminates two alternative explanations before the developer has to.

## 5. Reproducibility and intermittent bugs

"Happens sometimes" is the hardest report to act on. Make it actionable:

- **Quantify it.** "3 of 10 attempts" is data; "sometimes" is not.
- **Log everything around the failure**: exact timestamp (with timezone), user
  ID, session ID, request ID. This lets a developer find the server-side trace.
- **Vary one factor at a time** to find the trigger: fresh vs existing session,
  fast vs slow clicking, cold vs warm cache, first vs subsequent run, different
  data volumes.
- **Look for the usual suspects**: race conditions (does slowing down make it
  disappear?), caching, time/timezone boundaries, leftover state from a previous
  test, concurrency between two users.
- **Never suppress it.** An intermittent bug in test is a certainty in
  production, where the volume is a thousand times higher.

## 6. Common bug-reporting mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Reporting a symptom, not the trigger | Developer can't reproduce | Narrow it: which input, which state, which role |
| Two bugs in one report | Half gets fixed, the ticket closes | One defect per report, always |
| Missing environment | Fixed on the wrong build | Always include build, browser, OS, account |
| No expected result | Becomes an argument about intent | Cite the requirement, spec or mockup |
| Blaming a person | Poisons the relationship | Describe behaviour; systems have bugs, people don't "have" bugs |
| Steps that assume context | Fails at step 1 for anyone else | Start from a known state — logged out, cart empty |
| Screenshot only, no text | Unsearchable, unreadable in reports | Text first; screenshot as evidence |
| Filing without checking for duplicates | Triage overhead, split discussion | Search the tracker before logging |

## 7. Jira basics for testers

Jira is the most common defect tracker. The parts a tester touches daily:

**Issue types.** `Bug` (defect), `Story` (a user-facing requirement), `Task`
(work item), `Sub-task`, `Epic` (a group of stories). Test-related types depend
on your setup — some teams add `Test` and `Test Execution` via Xray or Zephyr.

**Key fields on a Bug.** Summary, Description (put the template above here),
Priority, Severity (usually a custom field), Affects Version/s, Fix Version/s,
Environment, Components, Labels, Assignee, Reporter, Sprint, Linked Issues.

**Workflow.** Jira's default bug workflow maps to section 1: `Open → In
Progress → Resolved → Closed`, with `Reopened` looping back. Teams customise
heavily; learn *your* board's transitions and who is allowed to make them.

**Linking is how traceability works.** Link the bug to the story it breaks
(`blocks` / `is blocked by`), to the duplicate (`duplicates`), and to related
defects (`relates to`). A bug with no link is an orphan in every report.

**JQL — the query language**, and the four queries every tester should have
saved:

```text
project = SHOP AND type = Bug AND status not in (Closed, Resolved) AND reporter = currentUser() ORDER BY priority DESC
project = SHOP AND type = Bug AND status = Resolved AND "Fix Version/s" = "2.14.0"
project = SHOP AND type = Bug AND priority in (Highest, High) AND status != Closed ORDER BY created ASC
project = SHOP AND type = Bug AND status changed to Reopened AFTER -14d
```

Those are, in order: my open bugs; what I need to retest; the urgent open list;
and the reopen rate — the last of which is the single best measure of fix
quality on a team.

**Screenshots and attachments.** Attach directly to the issue rather than
pasting into a comment thread; comments get collapsed and evidence gets lost.

## 8. Defect metrics worth tracking

| Metric | Formula | Tells you |
|---|---|---|
| **Defect density** | Defects ÷ size (KLOC or story points) | Which modules are risky |
| **Defect removal efficiency (DRE)** | Defects found before release ÷ (before + after) × 100 | How effective testing was |
| **Reopen rate** | Reopened ÷ total fixed × 100 | Fix quality, and whether retest is rigorous |
| **Defect age** | Time from New to Closed | Triage and fix responsiveness |
| **Defect leakage** | Defects found in production ÷ total × 100 | What your process is missing |
| **Rejection rate** | Rejected ÷ total logged × 100 | Report quality — high means testers are guessing |

A DRE of 95% means 5 of every 100 defects reached customers. Track the trend,
not the absolute number, and never let a metric become a target that someone can
game by re-labelling tickets.

## Exercise

1. Take any real application and find **three genuine defects** (cosmetic ones
   count). Write a full bug report for each using the template in section 3 —
   complete environment details, numbered minimal steps, precise expected vs
   actual. Assign both severity and priority, and **justify each pairing in one
   sentence**.
2. Construct **one example of each of the four severity/priority combinations**
   from a hypothetical banking app, and explain the reasoning.
3. Rewrite these four titles properly:
   - "search not working"
   - "Page slow"
   - "error msg"
   - "Cannot save — urgent!!!"
4. A developer marks your defect **Cannot Reproduce**. Write the comment you'd
   add — list the specific extra information you'd supply and the two questions
   you'd ask them, without escalating or blaming.
5. Your team's reopen rate is 30% and rejection rate is 25%. Diagnose the two
   likely root causes for each number, and propose one concrete process change
   for each.
