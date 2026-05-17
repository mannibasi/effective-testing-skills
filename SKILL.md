---
name: effective-testing
description: >
  Apply when writing a new automated test, reviewing test code in a PR,
  diagnosing a flaky or failing test, or auditing a test suite for quality,
  coverage balance, or maintenance burden. Covers all layers: unit, API,
  integration, UI, and end-to-end. Use this skill to guide decisions about
  what to test, how to structure tests, which layer to test at, and whether
  existing tests are worth keeping.
---

## Quick reference

Use the section that matches what you're doing right now:

- **Writing a new test** → start with [Which layer?](#which-layer), then [Writing the test](#writing-the-test)
- **Reviewing a PR** → [Reviewing a test](#reviewing-a-test)
- **Fixing a flaky test** → [Diagnosing flakiness](#diagnosing-flakiness)
- **Auditing a suite** → [Auditing a suite](#auditing-a-suite)

---

## Which layer?

Choosing the right layer is the most consequential decision when writing a
test. The wrong layer makes tests slow, brittle, or unable to catch the bugs
that matter.

### Think in terms of size, not just type

Classify tests by the *resources they consume*, not just the label:

| Size | Runs in | Allowed to | Typical speed |
|---|---|---|---|
| **Small** | Single process | No I/O, no network, no sleep | Milliseconds |
| **Medium** | Single machine | Localhost network, threads, filesystem | Seconds |
| **Large** | Anywhere | External services, real network | Minutes |

The goal is to write the *smallest test that can meaningfully catch the
failure*. Small tests are fast and deterministic. Large tests are slow and
more prone to flakiness. Both are necessary — but large tests should be used
sparingly.

### Where to test what

| What you're verifying | Start here |
|---|---|
| Pure logic, calculations, edge cases | Small (unit) |
| Business rules in a service or function | Small or medium |
| Interaction between two components (e.g. service + DB) | Medium (integration) |
| A contract between services or APIs | Medium |
| A critical user journey end-to-end | Large (E2E) |
| Visual rendering / layout | Visual regression (separate tool) |

### The ROI view

The classic testing pyramid says "mostly unit tests." A more ROI-focused view
says **integration tests often give the best return** — they test real
interactions without the full cost and brittleness of E2E. A practical
breakdown for most codebases:

- **Small/unit** — fast logic checks, edge cases, pure functions. Many of these.
- **Medium/integration** — the bulk of your suite. Test real component
  interactions with real or in-process dependencies where practical.
- **Large/E2E** — reserved for critical user journeys. As few as needed to
  cover what lower layers can't.

Don't test business rules through the UI when you can test them at the API.
Don't test API behaviour when a direct function call works. Each layer up adds
cost and fragility.

### Common mis-levelling

- Verifying a discount calculation via Playwright → write a unit test instead
- Verifying an API contract using mocked HTTP → write an API test against a
  real endpoint
- Covering every edge case in E2E → push edge cases down to unit tests, keep
  one happy-path E2E

---

## Writing the test

### Structure: Arrange -> Act -> Assert

Every test follows this shape, clearly separated:

```python
def test_checkout_fails_when_card_is_expired():
    # Arrange
    user = create_user(email=unique_email())
    cart = add_item_to_cart(user, product_id="SKU-001")
    card = CreditCard(number="4111111111111111", expiry="01/20")

    # Act
    result = checkout(cart, payment=card)

    # Assert
    assert result.status == "failed", "Expected failure for expired card"
    assert result.error_code == "card_expired"
```

### Test one thing

Each test answers one question. If you cannot describe a test's purpose in a
single sentence, split it. Tests covering multiple behaviours produce ambiguous
failures — when it goes red, you won't know which behaviour broke.

### Name it clearly

```
[subject]_[scenario]_[expected_outcome]

checkout_with_expired_card_returns_failure
order_without_address_returns_422
dashboard_loads_for_authenticated_user
```

### Assert on outcomes, not steps

Assert on the end state the user cares about, not implementation details.
Outcome assertions survive refactoring and UI changes; step assertions don't.

```python
# Fragile — tests an implementation step
assert submit_button_was_clicked == True

# Durable — tests the outcome
assert order.status == "confirmed"
assert page.is_visible("#confirmation-banner")
```

### Make failure output self-explanatory

When a test fails, the output should tell the next engineer exactly what went
wrong without needing to re-run it or dig through logs.

```python
# Bad — tells you nothing useful
AssertionError: expected 201 but got 500

# Good — log context before asserting
logger.info(f"POST /orders payload: {payload}")
logger.info(f"Response {response.status}: {response.body[:500]}")
assert response.status == 201, (
    f"Order creation failed. "
    f"Status: {response.status}, body: {response.body}, payload: {payload}"
)
```

For UI tests, capture screenshots automatically on failure:

```python
@pytest.fixture(autouse=True)
def capture_on_failure(page, request):
    yield
    if request.node.rep_call.failed:
        page.screenshot(path=f"screenshots/{request.node.name}.png")
```

Don't log passwords, tokens, or PII in failure output.

### Own your data

Every test creates its own data and cleans up after itself. Never rely on
data another test created, or on pre-existing state in a shared database.

```python
# Unique data per run
def unique_email():
    return f"test+{uuid.uuid4().hex[:8]}@example.com"

# Rollback pattern for database tests
@pytest.fixture(autouse=True)
def db_transaction(db):
    with db.transaction() as txn:
        yield txn
        txn.rollback()
```

### Stay programmatic

The test must run from a CI trigger with zero human involvement.

- All config and credentials injected via environment variables
- Validate required env vars at startup; fail fast with a clear error
- Use condition-based waits, never `sleep N`

```python
# Bad
time.sleep(3)
assert page.find("#success-banner")

# Good
page.wait_for_selector("#success-banner", timeout=10_000)
assert page.is_visible("#success-banner")
```

### Use stable selectors (UI tests)

| Stability | Example |
|---|---|
| Best | `data-testid="submit-btn"` |
| Good | ARIA: `role="button" name="Submit"` |
| OK | Semantic: `<button>Submit</button>` |
| Fragile | CSS class: `.btn-primary` |
| Avoid | XPath position: `//div[3]/button[1]` |

---

## Reviewing a test

Work through these questions when reviewing test code in a PR:

**Right layer?**
- Is this the smallest test that could catch this failure?
- Could a unit or API test cover this instead of a UI/E2E test?
- Is it testing a real behaviour or an implementation detail?

**Trustworthy?**
- Does it assert on a meaningful outcome (not just status 200, not just
  "no exception")?
- Mutation check: if the feature broke, would this test fail? If not, the
  assertion is wrong.
- Does the failure message explain *what* failed, not just *that* it failed?

**Stable?**
- Any `sleep` calls? Replace with condition-based waits.
- Any shared data or shared accounts? Each test must own its data.
- Any hardcoded URLs or credentials? Inject via env vars.
- Does it depend on another test running first? Make it self-contained.

**Readable?**
- Can you understand what it tests in under 2 minutes?
- Does the name describe the scenario and expected outcome?
- Is Arrange / Act / Assert clearly separated?

**Unique?**
- Does it cover a behaviour not already tested elsewhere?
- If very similar tests exist, do they each protect against a *different*
  failure mode? If not, the redundant one should be removed.

---

## Diagnosing flakiness

A flaky test is a bug. Don't re-run until it passes — find the root cause.

**Step 1: Reproduce reliably**

```bash
for i in {1..10}; do
  run_test $TEST_NAME && echo "Pass $i" || echo "FAIL $i"
done
```

**Step 2: Match symptom to cause**

| Symptom | Likely cause | Fix |
|---|---|---|
| Passes locally, fails in CI | Timing / fewer resources | Condition-based waits; longer timeouts |
| Passes alone, fails in parallel | Shared state | Each test owns its data and accounts |
| Fails after one specific other test | Ordering dependency | Make the test self-contained |
| Intermittent with no pattern | Race condition | Find and eliminate the async dependency |
| Element not found | Fragile selector or timing | Stable `data-testid`; wait for presence |
| DB constraint error | Shared test data | Generate unique data per run |

**Step 3: Verify the fix**

Run 10 consecutive passes before closing — 3 is not enough for genuinely
flaky tests.

```bash
for i in {1..10}; do
  run_test $TEST_NAME && echo "Pass $i" || { echo "FAIL $i"; break; }
done
```

---

## Auditing a suite

Use when reviewing a test suite for overall health.

### Coverage balance

Is the suite shaped roughly like a trophy/pyramid, or inverted?

- Heavy E2E, thin integration -> push business logic tests down to the API layer
- Heavy unit, no integration -> you may be testing implementation, not behaviour
- No static analysis (linting, types) -> add it; catches whole classes of bugs for free

### Trust check

For each test (or group of tests) ask:

> *If this test fails, will the team investigate and fix the defect?*

- **Yes** -> it's earning its place.
- **No** -> it's noise. Find out why (too low risk, always flaky, wrong layer)
  and fix or delete it.

A smaller, trusted suite beats a large, ignored one. Deleting tests is a
legitimate engineering activity.

### Redundancy check

Group tests by the behaviour they protect against. If two tests would fail
for exactly the same reason, remove or merge the weaker one. Redundant tests
add maintenance cost without adding safety.

### Maintenance cost signal

After any UI or API change, count:
- Failures caused by a genuine bug -> good, the suite is doing its job
- Failures caused by brittle selectors or hardcoded values -> the suite is
  too fragile; fix those before adding new tests

If more than 10% of post-change failures are false positives, invest in
stabilising locators and data patterns first.

### Reporting check

- Do results go somewhere the whole team can see without SSH access?
- Are failures actionable — enough context to diagnose without re-running?
- Are flaky tests tracked separately from genuine failures?
- Is duration monitored? A suite that doubles in runtime over six months is
  a problem worth catching early.
