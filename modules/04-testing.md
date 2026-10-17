# Module 4: Testing Beyond "Ask Claude for Tests"

> You asked the AI to write tests. It wrote 47 of them. They all pass.
> Your app is still broken.
> Welcome to test theater.

"Add tests" is one of the most common prompts in vibe coding, and one of the least understood. The AI is great at producing things that *look* like tests: files named `*.test.ts`, lots of `expect(...)`, a satisfying wall of green checkmarks. Whether those tests would ever catch a real bug is a completely different question, and usually nobody asks it.

This module is about the difference between tests that make you *feel* safe and tests that actually *keep* you safe.

---

## Learning Objectives

By the end of this module you will be able to:

1. Recognize the main kinds of fake AI tests: tests that test the mock, tests that assert whatever the code currently does, and tests with no meaningful assertions.
2. Explain the test pyramid and when to use unit, integration, and end-to-end tests.
3. Write (or direct the AI to write) tests that check **behavior**, not implementation details.
4. Build a habit of edge cases and negative tests: empty input, bad input, wrong user, double submit.
5. Test authentication and permissions explicitly, the tests most vibe-coded apps are missing.
6. Set up CI so tests run on every push, automatically.
7. Use mutation-testing intuition ("if I broke this line, would a test fail?") to judge whether tests are worth anything.

---

## Vibe Fail: 100% Coverage, 0% Protection

*(Illustrative story. The pattern shows up in real codebases constantly.)*

Dev builds an invoicing app. Before launch, a friend asks, "Do you have tests?" Dev prompts: "Write comprehensive tests for the invoice module. Aim for 100% coverage."

The AI delivers. Coverage report: 100%. Dev screenshots it for Twitter.

Here's one of the tests:

```ts
it("calculates invoice total", async () => {
  const db = { getLineItems: vi.fn().mockResolvedValue([{ amount: 100 }, { amount: 50 }]) };
  const calc = { total: vi.fn().mockReturnValue(150) };

  const result = await buildInvoice(db, calc, "inv_1");

  expect(calc.total).toHaveBeenCalled();
  expect(result.total).toBe(150);
});
```

Read it carefully. The calculator is a mock that returns 150. The test checks that the result is 150. **It's testing the mock.** The real `total()` function could return `NaN`, add tax twice, or ignore discounts, and this test would stay green forever.

The tax calculation was wrong for every customer outside Dev's home state for two months. Coverage was 100% the whole time. Coverage measures which lines *ran* during tests, not whether anything *checked* that they did the right thing.

---

## Core Concepts

### 1. Why AI-generated tests are so often theater

When you ask an AI to "write tests for this code," you've asked it to look at the code and describe it. So it does. That leads to three classic failure modes:

**Failure mode A: Testing the mock.** Everything interesting is replaced with a fake that returns the expected answer, so the test only proves the fake works.

**Failure mode B: Asserting whatever the code does.** The AI reads the implementation, sees it returns `0.19` for a 10% discount (because of a bug), and writes `expect(discount).toBe(0.19)`. The bug is now *protected by a test.* Congratulations, you've cast your bug in concrete.

**Failure mode C: No meaningful assertion.**

```python
def test_create_user():
    user = create_user("a@b.com", "password123")
    assert user is not None   # this passes if create_user returns literally anything
```

**Failure mode D: Making the tests pass by changing the tests.** You ask the agent to fix a failing test. It "fixes" it by changing the expected value, adding `.skip`, or deleting the test. Some agents will also quietly loosen assertions. Always check *which side* of the test changed: the code or the expectation.

The root cause is the same in all four: **the tests were derived from the code instead of from the requirements.** A test is supposed to be an independent statement of what the code *should* do. If it's generated from what the code *does*, it can't disagree with the code, and a test that can't disagree is useless.

**Hard truth:** a test is only valuable if it can fail. If you've never seen a test fail, you don't know that it works.

### 2. The test pyramid

Not all tests are equal. The classic model is a pyramid:

```text
          /\
         /  \     End-to-end (E2E): few, slow, realistic
        /----\    "A real browser logs in and buys a poster"
       /      \
      /--------\  Integration: some, medium speed
     /          \ "The API route + real database return the right data"
    /------------\
   /              \ Unit: many, fast, focused
  /________________\ "calculateTotal() handles discounts correctly"
```

| Type | What it tests | Speed | Use it for |
|---|---|---|---|
| **Unit** | One function or module in isolation | Milliseconds | Business logic, calculations, validation, parsing |
| **Integration** | Several pieces together (API route + database, service + external API in test mode) | Seconds | Endpoints, queries, permissions, data persistence |
| **End-to-end** | The whole app through a real browser or client | Seconds to minutes | Critical user journeys: signup, login, checkout |

Why a pyramid? Unit tests are cheap and pinpoint problems precisely. E2E tests are realistic but slow and flaky, and when they fail you still have to figure out *where*. You want lots of the cheap ones and a handful of the expensive ones covering what matters most.

For a typical vibe-coded web app, a sane starting point:

- **Unit tests** for every function that does math, money, dates, permissions logic, or data transformation.
- **Integration tests** for every API route that reads or writes user data, using a real test database (not mocks).
- **3 to 10 E2E tests** for the journeys that would make you lose sleep if they broke: sign up, log in, the core paid action, and anything involving money.

**About mocks:** mock what you don't control and can't run locally (third-party APIs, email providers, payment processors, preferably via their official test modes). Don't mock your own business logic or your own database in integration tests. If you mock everything, you're testing a fantasy version of your app.

### 3. Test behavior, not implementation

A behavior test says: "Given this input, the outside world sees this result." An implementation test says: "Internally, function X called function Y with argument Z."

```ts
// BAD: tests implementation. Breaks if you refactor, passes if logic is wrong.
it("applies discount", () => {
  const spy = vi.spyOn(pricing, "multiply");
  checkout(cart, "SAVE10");
  expect(spy).toHaveBeenCalledWith(100, 0.9);
});
```

```ts
// GOOD: tests behavior. Survives refactors, fails if the answer is wrong.
it("SAVE10 takes 10% off the subtotal", () => {
  const cart = [{ priceCents: 5000, qty: 2 }]; // $100.00
  expect(calculateTotal(cart, "SAVE10")).toBe(9000); // $90.00
});

it("discount is applied only once even if code is submitted twice", () => {
  const cart = [{ priceCents: 5000, qty: 2 }];
  expect(calculateTotal(cart, "SAVE10", "SAVE10")).toBe(9000);
});
```

Notice where the expected value `9000` came from: **a human did the arithmetic from the requirement.** Not from running the code. That's the whole trick. The expected value must come from somewhere other than the code under test.

Good behavior tests:

- Have names that read like requirements ("expired coupons are rejected").
- Use realistic inputs.
- Assert on outputs, returned data, database state, HTTP status codes, or what the user sees.
- Survive a refactor that doesn't change behavior.

### 4. Edge cases and negative tests

Happy-path tests prove the app works when everyone behaves. Users don't behave. Attackers definitely don't behave. Your own code, three weeks from now, won't behave.

**The edge-case menu** (pick the relevant ones for each function):

| Category | Examples |
|---|---|
| Empty | `""`, `[]`, `{}`, `null`, `undefined`, `None`, zero items in cart |
| Boundaries | 0, 1, -1, max length, exactly at the limit, one past the limit |
| Wrong type | string where number expected, number as string `"10"` |
| Weird text | Emoji, accents, very long strings, whitespace-only, SQL/HTML characters |
| Time | Midnight, month end, Feb 29, daylight saving changes, different time zones |
| Money | Rounding (0.1 + 0.2), negative amounts, currency mismatch |
| Duplicates | Same request twice, double-click submit, same email registered twice |
| Missing things | Record doesn't exist, file not found, external API down |
| Concurrency | Two users editing the same record, two purchases of the last item |

**Negative tests** check that bad things are *refused*:

```python
def test_rejects_negative_quantity(client, auth_headers):
    res = client.post("/cart/items", json={"product_id": 1, "qty": -5}, headers=auth_headers)
    assert res.status_code == 422

def test_rejects_expired_coupon():
    with pytest.raises(CouponExpired):
        apply_coupon(cart_total_cents=10_000, code="SUMMER2023", today=date(2026, 1, 1))
```

A good rule of thumb: **for every happy-path test, write at least two unhappy-path tests.**

### 5. Testing auth and permissions (the tests you're missing)

This is the single highest-value category of test for a vibe-coded app, and the one AI almost never writes unless you ask. Most real data leaks in AI-built apps aren't clever hacks. They're "User A can see User B's stuff." (The vulnerability itself is covered in [Module 2](02-security-vulnerabilities.md). Here we make sure it stays fixed.)

For every endpoint or query that touches user data, test four cases:

1. **Not logged in** → 401 (or redirect to login).
2. **Logged in, owns the resource** → success.
3. **Logged in, does NOT own the resource** → 403 or 404, and no data leaked.
4. **Logged in, wrong role** (e.g., regular user hitting an admin action) → 403.

```ts
describe("GET /api/invoices/:id", () => {
  it("401 when not logged in", async () => {
    const res = await request(app).get(`/api/invoices/${aliceInvoice.id}`);
    expect(res.status).toBe(401);
  });

  it("200 for the owner", async () => {
    const res = await request(app).get(`/api/invoices/${aliceInvoice.id}`).set(authAs(alice));
    expect(res.status).toBe(200);
    expect(res.body.id).toBe(aliceInvoice.id);
  });

  it("404 for another user, with no data in the body", async () => {
    const res = await request(app).get(`/api/invoices/${aliceInvoice.id}`).set(authAs(bob));
    expect(res.status).toBe(404);
    expect(res.body).not.toHaveProperty("amount");
  });
});
```

If you use database-level security like Supabase Row Level Security, write tests that connect **as a regular user** (with the anon or user key, not the service role key) and try to read another user's rows. If those tests don't exist, you don't know RLS works. You just believe it does.

### 6. CI: tests that run without your memory

Tests you have to remember to run are tests you'll stop running. **Continuous Integration (CI)** runs your tests automatically on every push and every pull request. If tests fail, the change is flagged before it reaches production.

A minimal GitHub Actions workflow for a Node project:

```yaml
# .github/workflows/test.yml
name: test
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports: ["5432:5432"]
    env:
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/postgres
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm test
```

Then turn on **branch protection** so the main branch can't be merged into unless CI passes. Now the AI can't sneak a broken change past you, and neither can tired you.

CI rules:

- Tests run against a **test** database, never production.
- Secrets used in CI are test-mode keys stored as CI secrets, never committed.
- A red build blocks merging. "I'll fix it later" is how red builds become permanent wallpaper.
- Flaky tests get fixed or deleted. A test that randomly fails teaches everyone to ignore failures.

### 7. Mutation-testing intuition: would this test catch a bug?

Here's the most useful question in this module:

> **If I broke this line of code on purpose, would any test fail?**

That's mutation testing in one sentence. Tools like [Stryker](https://stryker-mutator.io/) (JavaScript/TypeScript, C#, Scala) and [mutmut](https://github.com/boxed/mutmut) (Python) automate it: they make small changes ("mutants") to your code, like flipping `>` to `>=`, changing `+` to `-`, or deleting a line, then run your tests. If tests still pass, the mutant "survived," which means your tests didn't notice the code was broken.

You don't need the tools to use the idea. Do it by hand:

1. Pick a function you care about (pricing, permissions, date logic).
2. Break it. Change `>=` to `>`. Remove the ownership check. Return early.
3. Run the tests.
4. If nothing fails, you've found a gap. Write a test that fails on the broken version, then revert the break.

This is also the best way to evaluate tests the AI wrote. Ask it to write tests, then break the code yourself and see if they notice. Theater tests won't.

### 8. Worked example: from weak tests to strong tests

Let's do the whole thing once, start to finish, on a small function. The requirements (written by a human, in English, *before* any code):

> - US orders under $50.00 pay $5.00 shipping.
> - US orders of $50.00 **or more** ship free.
> - International orders always pay $15.00.
> - An empty cart ($0) ships for $0.
> - Country codes are two letters, case-insensitive. Anything else is an error.
> - Subtotals are whole cents, never negative. Anything else is an error.

**Step 1: the AI's first implementation** has a classic off-by-one: `subtotalCents > 5000` instead of `>=`. A $50.00 order gets charged shipping.

**Step 2: the AI's weak tests.** Asked to "write tests for shippingCents," it reads the code and produces:

```js
// The AI's tests: derived from the code, not the requirements
it("returns shipping for US orders", () => {
  expect(shippingCents(5000, "US")).toBe(500); // enshrines the off-by-one bug
});
it("returns a number", () => {
  expect(typeof shippingCents(100, "CA")).toBe("number"); // passes for any number at all
});
```

Both green. One protects the bug; the other would pass if the function returned `-7`. No edge cases, no bad input, no boundary.

**Step 3: strong tests, written from the requirements.** Here's the corrected function and tests that would have caught the bug on the first run (Vitest; Jest is nearly identical, just drop the import line):

```js
// shipping.js
export const FREE_SHIPPING_THRESHOLD_CENTS = 5000; // $50.00
export const US_FLAT_RATE_CENTS = 500;             // $5.00
export const INTERNATIONAL_FLAT_RATE_CENTS = 1500; // $15.00

export function shippingCents(subtotalCents, country) {
  if (!Number.isInteger(subtotalCents) || subtotalCents < 0) {
    throw new RangeError("subtotalCents must be a non-negative integer");
  }
  if (typeof country !== "string" || !/^[A-Za-z]{2}$/.test(country)) {
    throw new TypeError("country must be a 2-letter code");
  }
  if (subtotalCents === 0) return 0; // empty cart ships nothing
  if (country.toUpperCase() === "US") {
    return subtotalCents >= FREE_SHIPPING_THRESHOLD_CENTS ? 0 : US_FLAT_RATE_CENTS;
  }
  return INTERNATIONAL_FLAT_RATE_CENTS;
}
```

```js
// shipping.test.js
import { describe, it, expect } from "vitest";
import { shippingCents } from "./shipping.js";

describe("shippingCents", () => {
  // Happy paths: expected values come from the requirements, not from running the code
  it("charges $5 for US orders under $50", () => {
    expect(shippingCents(4999, "US")).toBe(500);
  });

  it("is free for US orders of exactly $50 (boundary)", () => {
    expect(shippingCents(5000, "US")).toBe(0);
  });

  it("is free for US orders over $50", () => {
    expect(shippingCents(12000, "US")).toBe(0);
  });

  it("charges $15 for international orders, even large ones", () => {
    expect(shippingCents(4999, "CA")).toBe(1500);
    expect(shippingCents(50000, "DE")).toBe(1500);
  });

  // Edge cases
  it("charges nothing for an empty cart", () => {
    expect(shippingCents(0, "US")).toBe(0);
    expect(shippingCents(0, "FR")).toBe(0);
  });

  it("accepts lowercase country codes", () => {
    expect(shippingCents(1000, "us")).toBe(500);
  });

  // Negative tests: bad input is refused, loudly
  it.each([-1, 10.5, NaN, "5000", null])("rejects invalid subtotal %s", (bad) => {
    expect(() => shippingCents(bad, "US")).toThrow(RangeError);
  });

  it.each(["", "USA", "1A", undefined])("rejects invalid country %s", (bad) => {
    expect(() => shippingCents(1000, bad)).toThrow(TypeError);
  });
});
```

The same thing in Python with pytest. `parametrize` turns a table of cases into individual tests, which is the cleanest way to cover boundaries:

```python
# shipping.py
FREE_SHIPPING_THRESHOLD_CENTS = 5000  # $50.00
US_FLAT_RATE_CENTS = 500              # $5.00
INTERNATIONAL_FLAT_RATE_CENTS = 1500  # $15.00


def shipping_cents(subtotal_cents: int, country: str) -> int:
    if type(subtotal_cents) is not int or subtotal_cents < 0:  # rejects bools and floats too
        raise ValueError("subtotal_cents must be a non-negative integer")
    if not isinstance(country, str) or len(country) != 2 or not country.isalpha():
        raise ValueError("country must be a 2-letter code")
    if subtotal_cents == 0:
        return 0
    if country.upper() == "US":
        return 0 if subtotal_cents >= FREE_SHIPPING_THRESHOLD_CENTS else US_FLAT_RATE_CENTS
    return INTERNATIONAL_FLAT_RATE_CENTS
```

```python
# test_shipping.py
import pytest
from shipping import shipping_cents


@pytest.mark.parametrize(
    "subtotal, country, expected",
    [
        (4999, "US", 500),    # just under the threshold
        (5000, "US", 0),      # exactly at the threshold: free (boundary!)
        (12000, "US", 0),     # well over
        (1000, "us", 500),    # lowercase accepted
        (4999, "CA", 1500),   # international flat rate
        (50000, "DE", 1500),  # international never free
        (0, "US", 0),         # empty cart
        (0, "FR", 0),
    ],
)
def test_shipping_rules(subtotal, country, expected):
    assert shipping_cents(subtotal, country) == expected


@pytest.mark.parametrize("bad_subtotal", [-1, 10.5, "5000", None, True])
def test_rejects_invalid_subtotal(bad_subtotal):
    with pytest.raises(ValueError):
        shipping_cents(bad_subtotal, "US")


@pytest.mark.parametrize("bad_country", ["", "USA", "1A", None])
def test_rejects_invalid_country(bad_country):
    with pytest.raises(ValueError):
        shipping_cents(1000, bad_country)
```

Notice what changed from the weak version:

- Every expected value traces back to a line of the requirements.
- The **boundary** (exactly $50.00) has its own test. Off-by-one bugs live on boundaries.
- Bad input has tests that prove it's **refused**, not silently accepted.
- The Python version rejects `True` explicitly, because in Python `True` is technically an `int` (and equals 1). That's exactly the kind of edge case that only shows up when you go looking for it.
- Run the buggy `>` version against these tests and the boundary test fails immediately. That's the point.

### 9. Testing an API endpoint, with auth and fixtures

Unit tests cover the function. Integration tests cover the **endpoint**: routing, auth, validation, data access, and status codes together. Here's a complete, runnable example in Python (FastAPI + pytest). The app uses an in-memory store so you can run it anywhere; in your project, the fixtures would reset a real **test** database instead.

```python
# app.py — a tiny invoices API (in-memory store so the example runs anywhere)
from fastapi import Depends, FastAPI, Header, HTTPException

app = FastAPI()
INVOICES: dict[str, dict] = {}   # invoice_id -> {"id", "owner_id", "amount_cents"}
TOKENS: dict[str, str] = {}      # token -> user_id (stand-in for real session auth)


def current_user_id(authorization: str | None = Header(default=None)) -> str:
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Not logged in")
    user_id = TOKENS.get(authorization.removeprefix("Bearer "))
    if user_id is None:
        raise HTTPException(status_code=401, detail="Not logged in")
    return user_id


@app.get("/invoices/{invoice_id}")
def get_invoice(invoice_id: str, user_id: str = Depends(current_user_id)):
    invoice = INVOICES.get(invoice_id)
    if invoice is None or invoice["owner_id"] != user_id:
        raise HTTPException(status_code=404, detail="Not found")  # 404, not 403: don't confirm it exists
    return invoice
```

**Fixtures** are pytest's way of giving each test fresh, known data. **Factories** (`make_user`, `make_invoice`) build realistic records with sensible defaults, so each test only spells out what matters to it:

```python
# conftest.py — fixtures: fresh, known data for every test
import pytest
from fastapi.testclient import TestClient
import app as app_module


@pytest.fixture(autouse=True)
def clean_store():
    app_module.INVOICES.clear()
    app_module.TOKENS.clear()
    yield


@pytest.fixture
def client():
    return TestClient(app_module.app)


def make_user(user_id: str) -> dict:
    """Factory: creates a user with a login token and returns auth headers."""
    token = f"test-token-{user_id}"
    app_module.TOKENS[token] = user_id
    return {"id": user_id, "headers": {"Authorization": f"Bearer {token}"}}


def make_invoice(owner_id: str, amount_cents: int = 12_500) -> dict:
    invoice = {"id": f"inv_{owner_id}_{len(app_module.INVOICES) + 1}",
               "owner_id": owner_id, "amount_cents": amount_cents}
    app_module.INVOICES[invoice["id"]] = invoice
    return invoice


@pytest.fixture
def alice():
    return make_user("alice")


@pytest.fixture
def bob():
    return make_user("bob")


@pytest.fixture
def alice_invoice(alice):
    return make_invoice(owner_id=alice["id"])
```

And the tests themselves read like the permission requirements:

```python
# test_invoices_api.py — the four auth cases, against the real app
def test_401_when_not_logged_in(client, alice_invoice):
    res = client.get(f"/invoices/{alice_invoice['id']}")
    assert res.status_code == 401


def test_401_with_a_garbage_token(client, alice_invoice):
    res = client.get(f"/invoices/{alice_invoice['id']}", headers={"Authorization": "Bearer nope"})
    assert res.status_code == 401


def test_owner_can_read_their_invoice(client, alice, alice_invoice):
    res = client.get(f"/invoices/{alice_invoice['id']}", headers=alice["headers"])
    assert res.status_code == 200
    assert res.json()["amount_cents"] == 12_500


def test_other_user_gets_404_and_no_data(client, bob, alice_invoice):
    res = client.get(f"/invoices/{alice_invoice['id']}", headers=bob["headers"])
    assert res.status_code == 404
    assert "amount_cents" not in res.text


def test_missing_invoice_is_404(client, alice):
    res = client.get("/invoices/does-not-exist", headers=alice["headers"])
    assert res.status_code == 404
```

The JavaScript equivalent uses `beforeEach` to reset and seed the test database, and a request library such as Supertest (shown in section 5). The pattern is identical: **reset → seed known data with factories → act as a specific user → assert status and body.**

**Test data rules:**

- Every test starts from a known state. No test depends on data left behind by another test, or on the order tests run in.
- Use factories with defaults instead of giant shared fixtures. When a test needs "an expired coupon," it says `make_coupon(expires_at=yesterday)`, and everything else gets a sane default.
- Never point tests at production, and never copy real customer data into test fixtures. Use obviously fake data (`alice@example.com`; the `example.com` domain is reserved for exactly this).
- Keep at least **two users** in every permission-related test. One-user test suites can't catch "User A sees User B's data."

### 10. Flaky tests: the ones that pass on Tuesdays

A **flaky test** sometimes passes and sometimes fails without any code change. Flaky tests are worse than no tests, because they train everyone (including your AI agent) to rerun until green and ignore red.

| Cause | Symptom | Fix |
|---|---|---|
| Real time / dates | Fails near midnight, month-end, or DST changes | Freeze time (fake timers) or pass "now" in as a parameter |
| Randomness | Fails one run in fifty | Seed the random generator, or inject the random value |
| Shared state | Passes alone, fails in the full suite (or vice versa) | Reset state in `beforeEach`/fixtures; no globals between tests |
| Order dependence | Fails when tests run in a different order or in parallel | Each test creates its own data |
| Real network calls | Fails when a third-party API is slow or down | Use the provider's test mode or a fake at the boundary |
| Timing / async | `sleep(500)` "fixes" it, mostly | Await the actual condition (the promise, the element appearing), never a fixed delay |

Time is the most common culprit. Here's a function that depends on the clock, and a test that controls the clock instead of hoping:

```js
// trial.js
const TRIAL_LENGTH_MS = 14 * 24 * 60 * 60 * 1000;

export function isTrialExpired(signupAt) {
  return Date.now() - signupAt.getTime() > TRIAL_LENGTH_MS;
}
```

```js
// trial.test.js
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { isTrialExpired } from "./trial.js";

describe("isTrialExpired", () => {
  beforeEach(() => {
    vi.useFakeTimers();
    vi.setSystemTime(new Date("2026-03-15T12:00:00Z")); // "now" is fixed for every test
  });
  afterEach(() => {
    vi.useRealTimers();
  });

  it("is not expired exactly 14 days after signup", () => {
    expect(isTrialExpired(new Date("2026-03-01T12:00:00Z"))).toBe(false);
  });

  it("is expired one millisecond after 14 days", () => {
    expect(isTrialExpired(new Date("2026-03-01T11:59:59.999Z"))).toBe(true);
  });
});
```

In Python, the cleanest fix is to make "now" a parameter (`def is_trial_expired(signup_at, now)`) and pass a fixed datetime in tests; libraries like `freezegun` or `time-machine` can freeze the clock when you can't change the signature.

**Policy:** when a test flakes, fix it within a day or delete it and write a better one. Never let "just rerun it" become a habit, and never let the AI "fix" a flaky test by adding retries or longer sleeps.

### 11. What to test first in an existing, untested app

You have an app with zero tests and limited time. Don't ask the AI for "comprehensive tests" (you'll get 200 theater tests). Prioritize by **what would hurt most if it broke**:

1. **Authorization on user data.** The four auth cases for every route that reads or writes another person's data. Highest risk, highest value.
2. **Money.** Prices, totals, discounts, credits, refunds, plan limits. Unit tests with boundaries.
3. **The critical journey.** One E2E test each for signup, login, and your core paid or value-delivering action.
4. **Anything that has broken before.** Every bug you fix from now on gets a regression test that fails before the fix and passes after. This alone builds a valuable suite within months.
5. **Data transformations.** Imports, exports, parsing, date logic, anything that turns one shape of data into another.
6. **Everything else**, opportunistically: when you touch a file, leave it with a test it didn't have before.

A realistic first week: 4 auth tests for your most sensitive route, 10 unit tests for your pricing or core rule, 1 E2E test for signup-to-core-action, CI running all of them. That's maybe a day of focused work, and it covers the failures that end businesses.

---

## What Good Looks Like

**A healthy test suite for a small app:**

- Tests are named like requirements, and a non-engineer could read the list of test names and understand what the app promises.
- Expected values are computed from requirements, not copied from code output.
- Business logic (money, dates, permissions, validation) has thorough unit tests with edge cases.
- Every data-touching API route has integration tests, including the four auth cases.
- A handful of E2E tests cover signup, login, and the core paid flow.
- Tests run in CI on every push; main is protected.
- Breaking a critical line on purpose makes at least one test fail.
- Mocks are only used for third-party services you can't run locally.

**Test review checklist (for AI-written tests):**

- [ ] Does each test have an assertion that could actually fail?
- [ ] Where did the expected values come from: the requirement, or the code's current output?
- [ ] Is the thing being tested mocked? (If so, the test is about the mock.)
- [ ] Are there negative tests (bad input, wrong user, missing data)?
- [ ] Are auth cases covered: logged out, owner, non-owner, wrong role?
- [ ] Did the diff change any existing expected values, add `.skip`, or delete tests? Why?
- [ ] If I break the code on purpose, does a test fail?
- [ ] Do the tests run in CI?

---

## Prompt Templates

**1. Tests from requirements, not from code**

```text
Write tests for [function/endpoint] based on these REQUIREMENTS, not on the
current implementation:
- [requirement 1]
- [requirement 2]
- [requirement 3]
Compute expected values from the requirements and show your arithmetic in
comments. If the current implementation fails a test, DO NOT change the test
to match. Report the failure to me instead.
```

**2. Edge cases and negatives**

```text
For [function], list at least 10 edge cases and negative cases (empty input,
boundaries, wrong types, duplicates, missing records, time zones, rounding,
concurrency where relevant). For each, state the correct expected behavior.
Then write tests for them. At least two unhappy-path tests per happy-path test.
```

**3. Auth and permission matrix**

```text
For every API route in [folder] that reads or writes user data, write
integration tests covering: (1) not logged in, (2) logged in as owner,
(3) logged in as a different user, (4) logged in with insufficient role.
Use a real test database with two seeded users. Do not mock the auth or
database layer. Assert on status codes AND that no other user's data appears
in the response body.
```

**4. No mocking my own logic**

```text
Only mock external services we don't control ([Stripe/SendGrid/etc.], using
their official test modes or fakes). Do not mock our own modules, services,
or database in integration tests. If a test needs data, seed it in a test DB.
```

**5. Fix the code, not the test**

```text
These tests are failing: [paste output]. Fix the APPLICATION CODE so they pass.
Do not modify test files, expected values, or add skip/only. If you believe a
test itself is wrong, stop and explain why instead of changing it.
```

**6. Mutation check**

```text
Here is [function] and its tests. Suggest 10 small mutations to the function
(flip comparisons, change constants, remove a condition, return early). For
each, tell me whether the existing tests would catch it. For any that would
survive, write a test that kills it.
```

**7. CI setup**

```text
Create a GitHub Actions workflow that runs lint, typecheck, and tests on every
push and pull request, using a Postgres service container for the test
database. Use only test-mode secrets referenced from GitHub Actions secrets.
Do not hard-code any credentials.
```

---

## Exercise

**Part A: Audit the theater (20 minutes)**

1. Open the test folder of a project you built with AI. (If there's no test folder, that's your answer, skip to Part B.)
2. Pick five tests. For each one, answer: what is mocked? Where did the expected value come from? Could this test fail if the logic were wrong?
3. Label each test: **real**, **theater**, or **partial**.

**Part B: The sabotage drill (20 minutes)**

1. Pick your most important function (money, permissions, or core business rule).
2. Break it three different ways, one at a time: flip a comparison, remove a check, change a constant.
3. Run tests after each break. Write down which breaks survived.
4. Use Prompt Template 1 or 6 to write tests that catch the survivors. Confirm they fail on the broken code and pass on the fixed code.

**Part C: The auth matrix (30 minutes)**

1. Pick one API route that returns user-specific data.
2. Write (or prompt for) the four auth tests: logged out, owner, other user, wrong role.
3. Run them. If the "other user" test passes when it should fail, stop everything and go read [Module 2](02-security-vulnerabilities.md).

**Part D: CI (15 minutes)**

Add a CI workflow that runs your tests on push. Make one commit that breaks a test and confirm the build goes red. Then fix it.

---

## Key Takeaways

- "Ask the AI for tests" produces test-shaped objects. Whether they protect you depends on whether they can fail.
- The four kinds of theater: testing the mock, asserting whatever the code does, no real assertion, and "fixing" tests by changing expectations.
- Expected values must come from **requirements**, not from running the code.
- Use the test pyramid: many fast unit tests, some integration tests with a real test database, a few E2E tests for critical journeys.
- Test behavior (inputs and outputs), not implementation (which internal function got called).
- For every happy-path test, write at least two unhappy-path tests.
- Auth and permission tests are the highest-value tests in most vibe-coded apps: logged out, owner, other user, wrong role.
- Run tests in CI on every push, and protect your main branch.
- The killer question: "If I broke this line, would a test fail?" If not, you don't have a test. You have a decoration.
- Coverage tells you what ran. It doesn't tell you what was checked.

---

**Next up:** [Module 5: Architecture for Prompt Engineers](05-architecture.md)
