# 01 · What Is Software Testing?

Software testing is the disciplined process of evaluating a product to find the
difference between **expected behaviour** and **actual behaviour**, and to give
stakeholders information about quality and risk. Note what that definition does
*not* say: it does not say "prove the software works." You can never prove that.
You can only reduce uncertainty about where it doesn't.

This module covers the vocabulary and mental model that everything else in this
course is built on.

## 1. Why testing exists

Every non-trivial system has more possible states than anyone can execute. A
single login form with an email field, a password field, a "remember me"
checkbox and a locale setting already has effectively infinite input
combinations. Testing is therefore an exercise in **sampling intelligently**:
choosing the small set of cases most likely to reveal a problem.

Three things follow from that, and they shape the whole profession:

1. **Testing is risk management, not certification.** "We tested it" is not a
   guarantee of correctness — it's a statement that a defined set of risks was
   examined.
2. **Bugs get more expensive the later they're found.** A requirement
   misunderstanding caught in a review costs a conversation. The same
   misunderstanding caught in production costs a hotfix, a rollback plan, a
   support queue, and possibly customer trust.
3. **Absence of evidence is not evidence of absence.** "No bugs found" means
   your tests didn't find any — it says as much about your tests as about the
   product.

!!! quote "Dijkstra's rule"
    "Testing shows the presence, not the absence of bugs." Internalise this
    early; it prevents both false confidence and the pointless goal of
    "100% tested."

## 2. Verification vs validation

Two words that get used interchangeably in casual conversation, and shouldn't be.

| | Verification | Validation |
|---|---|---|
| Question it answers | "Are we building the product **right**?" | "Are we building the **right** product?" |
| Compares against | Specification, design, standards | User needs, business intent |
| Typical activities | Reviews, walkthroughs, static analysis, unit tests | Functional testing, UAT, beta testing |
| Needs running code? | Not necessarily | Yes |
| Classic failure it catches | Field accepts 300 characters when the spec says 255 | Spec says 255, but real customer addresses need 400 |

A product can pass verification perfectly and still fail validation — it does
exactly what the document said, and the document was wrong.

## 3. QA vs QC vs testing

These three are nested, not synonymous. Getting them straight matters because
job titles and interview questions use them precisely.

| Term | Scope | Orientation | Owned by | Example |
|---|---|---|---|---|
| **QA** (Quality Assurance) | The whole *process* | Preventive — stop defects being created | Whole team / QA lead | Introducing definition-of-done, code review standards, CI gates |
| **QC** (Quality Control) | The *product* | Detective — find defects that exist | QA engineers | Executing a regression suite before release |
| **Testing** | A set of *activities* | Detective — a subset of QC | Testers, developers | Running 40 test cases against the checkout flow |

The one-line version: **QA is process-oriented and proactive; QC is
product-oriented and reactive; testing is how QC gets done.**

## 4. Manual vs automated testing

The most common false dichotomy in QA. They are not competitors — they answer
different questions.

| | Manual testing | Automated testing |
|---|---|---|
| Best at | Exploration, usability, ad-hoc investigation, one-off checks, new features in flux | Repetition, regression, data-heavy checks, speed, CI gating |
| Human judgement | Present — a tester notices "this looks wrong" even without a spec | Absent — a script only checks what it was told to check |
| Cost shape | Low setup, high per-run cost | High setup, near-zero per-run cost |
| Feedback speed | Minutes to hours | Seconds to minutes |
| Fails when | Repeated 200 times — humans get bored and miss things | The UI changes — scripts break and need maintenance |
| Finds | New, unexpected problems | Previously known problems that came back |

The practical rule used across the industry:

> **Automate what you'd otherwise repeat. Manually test what requires judgement.**

A test case that runs once, on one build, to answer one question is almost never
worth automating. A test case that will run on every commit for the next two
years almost always is.

!!! warning "The automation myth"
    "We'll automate 100% of testing" is a red flag in any test strategy.
    Automation cannot tell you a button is ugly, a workflow is confusing, an
    error message is condescending, or that a feature nobody asked for was
    built. Those are exactly the findings that make testers valuable.

## 5. The tester mindset

Technique can be taught in a week; mindset is what separates a good tester from
someone executing a checklist. The traits that matter:

- **Curiosity over compliance.** The step says "enter a valid email." A tester
  also asks: what about an email with a `+` alias? 300 characters? Unicode? The
  same email as an existing user, in different capitalisation?
- **Systematic scepticism.** Assume nothing works until observed. "The developer
  said it's fixed" is a hypothesis, not a result.
- **Empathy for the user.** The product will be used by tired people on bad
  Wi-Fi, on a small screen, in a hurry. Test in that world, not in the ideal one.
- **Comfort with being the bearer of bad news.** Reporting defects is the job.
  Report the *behaviour*, never the person — "the total is wrong" not "you
  broke the total."
- **Knowing when to stop.** There is always another test. Testing ends when the
  remaining risk is acceptable and agreed, not when you run out of ideas.

### The seven testing principles

A standard set worth memorising — they show up in ISTQB exams and in real
arguments about scope:

| # | Principle | What it means in practice |
|---|---|---|
| 1 | Testing shows presence of defects | You can never test "done"; you can only test "enough" |
| 2 | Exhaustive testing is impossible | Sample by risk, not by completeness |
| 3 | Early testing saves time and money | Review requirements before code exists |
| 4 | Defects cluster | ~80% of bugs live in ~20% of modules — go where the fire is |
| 5 | Pesticide paradox | The same tests stop finding bugs; refresh them periodically |
| 6 | Testing is context-dependent | A medical device and a marketing site need different rigour |
| 7 | Absence-of-errors fallacy | A bug-free product nobody wants is still a failure |

## 6. SDLC — where testing lives

The **Software Development Life Cycle** is the umbrella process for building
software. Testing is not a phase at the end of it; it touches every stage.

| SDLC stage | What happens | Tester's contribution |
|---|---|---|
| Requirements | Business needs captured | Review for ambiguity, testability, missing edge cases |
| Design | Architecture and UI designed | Identify risky areas; start planning test approach |
| Development | Code written | Unit tests by devs; testers prepare test cases and data |
| Testing | Builds validated | Execute, report defects, retest, regression |
| Deployment | Release to production | Smoke test the release; verify configuration |
| Maintenance | Fixes and enhancements | Regression testing; confirm fixes don't break neighbours |

Common SDLC models and their testing implication:

- **Waterfall** — sequential; testing is a distinct late phase. Defects found
  late are expensive. Still common in regulated/hardware-adjacent work.
- **V-Model** — every development stage has a matching test level (requirements
  ↔ acceptance testing, design ↔ system testing, and so on). It makes the
  verification/validation pairing explicit.
- **Agile/Scrum** — testing happens inside each sprint, alongside development.
  Testers join refinement, write cases against acceptance criteria, and
  regression is automated because it runs every sprint.
- **DevOps/CI-CD** — automated tests gate every commit. Manual testing
  concentrates on exploratory work and the things a pipeline can't judge.

## 7. STLC — the testing life cycle

Where SDLC governs building the product, the **Software Testing Life Cycle**
governs the testing work itself. It's the sequence you'll actually live in.

| Phase | Activities | Entry criteria | Exit criteria / deliverable |
|---|---|---|---|
| **1. Requirement analysis** | Study requirements, identify what's testable, list ambiguities | Requirements available | Testability report, list of questions, automation feasibility |
| **2. Test planning** | Scope, approach, effort estimate, tools, roles, schedule, risks | Requirements analysed | **Test plan** document, effort estimate |
| **3. Test case development** | Write test cases and scripts, prepare test data, build traceability matrix | Test plan approved | Test cases, test data, **RTM** |
| **4. Test environment setup** | Provision environments, install builds, configure test data | Environment spec ready | Environment ready + smoke test passed |
| **5. Test execution** | Run cases, log results, raise defects, retest fixes | Environment + cases ready | Test execution report, defect log |
| **6. Test closure** | Evaluate exit criteria, report coverage/defect metrics, capture lessons | Execution complete | **Test summary report**, lessons learned |

!!! info "Entry and exit criteria"
    Every STLC phase has both. They are the antidote to the classic dysfunction
    of testing starting on a build that doesn't install, or a release shipping
    because the date arrived rather than because the criteria were met. Write
    them down before the phase begins, when nobody is under pressure yet.

## 8. Static vs dynamic testing

One more axis, often skipped by self-taught testers, and the source of a lot of
cheap defect prevention.

- **Static testing** examines artefacts *without executing code*: requirement
  reviews, design walkthroughs, code reviews, linters, static analysers. It's
  the cheapest defect-finding available — catching an ambiguous requirement in a
  30-minute review costs almost nothing.
- **Dynamic testing** executes the software and observes behaviour. Everything
  from Module 3 onward is dynamic.

A team that only does dynamic testing is paying full price for every defect it
finds.

## Glossary

| Term | Meaning |
|---|---|
| **Error / mistake** | The human action that produced the problem (a developer's slip) |
| **Defect / bug / fault** | The flaw in the artefact resulting from an error |
| **Failure** | The observable wrong behaviour when the defect executes |
| **Test case** | A documented set of inputs, steps and expected results |
| **Test scenario** | A high-level "what to test" statement; yields multiple test cases |
| **Test suite** | A grouped collection of test cases run together |
| **Build** | A specific compiled version of the software handed to test |
| **Regression** | Re-testing existing functionality after a change |
| **Coverage** | A measure of how much (of code, requirements, risk) was tested |
| **RTM** | Requirements Traceability Matrix — maps requirements to test cases |
| **SUT / AUT** | System / Application Under Test |

A defect does not always cause a failure — a bug in a branch nobody executes sits
there silently. This is why coverage matters.

## Exercise

Pick any application you use daily — a banking app, an e-commerce site, a food
delivery service — and write up the following, in your own words:

1. **Three verification questions** and **three validation questions** you would
   ask about its login flow. Make the distinction obvious in each.
2. **Five test ideas** for its search feature that a purely automated approach
   would *not* catch, and explain why automation would miss each one.
3. Map the app's release process onto the **six STLC phases**. For any phase you
   can't observe from outside, state what you would need to ask the team.
4. Identify one part of the app where you'd expect **defect clustering**, and
   justify your reasoning (complexity, frequency of change, integration points).

Keep this write-up — you'll turn parts of it into formal test cases in Module 2
and into a real test plan in Module 10.
