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
