# Module 6: Error Handling & Observability

> Your app has two states: "working" and "you don't know."
> Without observability, those look exactly the same from where you're sitting.

In your terminal, errors are loud. A red stack trace scrolls by, you paste it into the chat, the AI fixes it. Lovely.

In production, errors are silent. They happen on someone else's phone, at 3 a.m., in a time zone you've never visited. The user sees a spinner that never stops, closes the tab, and never comes back. You find out three weeks later from a one-star review, or never.

**Observability** is the ability to answer "what is my app doing right now, and why?" without guessing. **Error handling** is making sure that when things go wrong (and they will), they go wrong *visibly*, *safely*, and *recoverably*. AI-generated code is notoriously bad at both, because the AI's goal is to make the error disappear from *your* screen, which is not the same as making it go away.

---

## Learning Objectives

By the end of this module you will be able to:

1. Spot and fix error handling that swallows errors (empty catches, log-and-continue, fake success).
2. Separate user-facing errors (friendly, safe) from internal errors (detailed, private).
3. Replace `console.log` debugging with structured logging.
4. Keep secrets and personal data out of logs.
5. Set up error tracking (e.g., Sentry) so you hear about exceptions before your users tell you.
6. Choose a few meaningful metrics and alerts instead of none (or a hundred).
7. Add a health check endpoint and know what it should check.
8. Debug production issues from evidence instead of vibes.

---

## Vibe Fail: The Signup Form That Thanked Everyone
*Illustrative composite.*

Omar launched a waitlist for his AI writing tool. He posted it on a few forums, the signups rolled in, and the counter on his landing page climbed past 2,000. He was thrilled.

A month later he went to email the waitlist. The database had 312 rows.

Here's the signup handler the AI wrote:

```js
async function handleSignup(email) {
  try {
    await db.insert("waitlist", { email });
  } catch (e) {
    // ignore duplicate errors
  }
  incrementCounter();
  return { message: "Thanks! You're on the list." };
}
```

The comment said "ignore duplicate errors." The code ignored **all** errors. A few days after launch, the database's free tier connection limit started rejecting inserts during traffic spikes. Every failure was silently swallowed, the counter still went up, and every user still saw "Thanks!" Nothing was logged. No alert fired. No error tracker existed.

Omar lost about 85% of his launch signups, and he couldn't even email them to apologize, because he didn't have their addresses. The fix would have been four lines of error handling and a free error-tracking account.

---

## Core Concepts

### 1. The many ways to swallow an error

The AI's favorite trick for "fixing" a crash is to wrap it in `try/catch` and move on. Learn to recognize the family:

```js
// 1. The black hole
try { await save(data); } catch (e) {}

// 2. The diary entry (logs to a console nobody reads in production)
try { await save(data); } catch (e) { console.log(e); }

// 3. The liar
try { await save(data); return { ok: true }; } catch (e) { return { ok: true }; }

// 4. The default that hides the problem
const user = await getUser(id).catch(() => null); // was it "not found" or "DB is down"?

// 5. The promise nobody awaited
sendWelcomeEmail(user); // no await, no .catch: fails silently, or crashes the process later
```

```python
# 6. Python's version: the bare except
try:
    process_payment(order)
except:
    pass   # also catches KeyboardInterrupt, typos (NameError), everything
```

**The principle:** only catch an error if you can **do something useful** with it: retry, fall back, add context, or translate it into a clear response. Otherwise let it propagate to a place that can handle it (a global error handler) where it gets logged and tracked.

```js
// GOOD: catch the specific thing you expect, let everything else surface
async function handleSignup(email) {
  try {
    await db.insert("waitlist", { email });
  } catch (err) {
    if (err.code === "23505") {               // Postgres unique violation: genuinely a duplicate
      return { message: "You're already on the list!" };
    }
    logger.error({ err, route: "signup" }, "waitlist insert failed");
    throw err;                                // global handler returns a 500 + error tracker records it
  }
  return { message: "Thanks! You're on the list." };
}
```

```python
# GOOD: specific exceptions, context, re-raise
try:
    process_payment(order)
except CardDeclined as e:
    return PaymentResult(ok=False, user_message="Your card was declined.")
except Exception:
    logger.exception("payment failed", extra={"order_id": order.id})
    raise
```

### 2. User-facing vs. internal errors

Every error has two audiences, and they need completely different messages.

| | User-facing error | Internal error |
|---|---|---|
| **Audience** | Your customer | You (future, tired you) |
| **Goal** | Tell them what happened and what to do | Tell you exactly what broke and why |
| **Content** | Plain language, next step, maybe a reference ID | Stack trace, input context, IDs, timing |
| **Where it goes** | The screen / API response | Logs and error tracker |
| **Must never include** | Stack traces, SQL, file paths, internal IDs, secrets | Passwords, tokens, full card numbers, unnecessary personal data |

```ts
// BAD: leaks internals to the user (and to attackers doing reconnaissance)
return Response.json({ error: err.stack }, { status: 500 });
// "error": "PostgresError: relation \"users_v2\" does not exist at /app/src/db.ts:42..."

// GOOD: generic message + correlation ID; details go to logs
const errorId = crypto.randomUUID();
logger.error({ err, errorId, userId: user?.id, route: "/api/orders" }, "order creation failed");
return Response.json(
  { error: "Something went wrong creating your order. Please try again.", errorId },
  { status: 500 }
);
```

The `errorId` is a quiet superpower: when a user emails "it broke," they can give you the ID and you can find the exact log entry in seconds.

Also: **use the right status codes.** 400/422 for bad input, 401 for not logged in, 403 for not allowed, 404 for not found, 409 for conflicts, 429 for rate limited, 500 for "our fault." Returning 200 with `{ error: ... }` breaks monitoring, retries, and every tool that assumes HTTP means what it says.

### 3. Structured logging (not `console.log("here 2")`)

`console.log` debugging works on your laptop. In production you'll have thousands of log lines per minute from many requests at once, and "here 2" won't help.

**Structured logs** are machine-readable (usually JSON) with consistent fields, so you can search and filter them:

```text
BAD:  User 123 checkout failed!!!
GOOD: {"level":"error","time":"2026-10-17T22:41:07Z","msg":"checkout failed","userId":"u_123","orderId":"o_987","requestId":"req_abc","durationMs":2311,"err":{"type":"StripeCardError","message":"card_declined"}}
```

Now you can ask your log tool: "show me all `checkout failed` errors in the last hour, grouped by `err.type`." Try that with "failed!!!".

Use a real logger: **pino** or **winston** in Node, the built-in **logging** module (or **structlog**) in Python. Most hosting platforms (Vercel, Render, Fly.io, Railway, AWS, etc.) collect stdout logs; structured JSON makes them searchable.

**Logging levels:**

- `debug`: detailed info for development; usually off in production.
- `info`: normal, notable events (user signed up, order placed, job finished).
- `warn`: something odd but handled (retrying a failed API call, deprecated usage).
- `error`: something failed and needs attention.

**Always include:** a request ID (generate one per request and attach it to every log line for that request), the user or account ID (an internal ID, not their email), the operation, and timing for slow things.

```js
import pino from "pino";
export const logger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  redact: ["req.headers.authorization", "req.headers.cookie", "*.password", "*.token", "*.apiKey"],
});

// per request
const log = logger.child({ requestId, userId: user?.id, route: "POST /api/orders" });
log.info({ itemCount: items.length }, "creating order");
```

### 4. Never log secrets or PII

Logs are often the least protected data you have. They get shipped to third-party services, retained for months, viewed by contractors, pasted into chats (including AI chats) while debugging. Treat them as semi-public.

**Never log:**

- Passwords (even failed attempts; users often type their password into the username field)
- API keys, access tokens, session cookies, `Authorization` headers
- Full credit card numbers, CVVs, bank details
- Government IDs, health information
- Full request bodies from auth, payment, or profile endpoints

**Be careful with:** emails, names, phone numbers, IP addresses, addresses. Depending on where your users live, these are regulated personal data (GDPR, CCPA, and friends). Log internal IDs instead and look up the person when you need to.

The classic AI-generated mistake:

```js
console.log("Incoming request:", req.headers, req.body); // logs cookies, tokens, passwords
console.log("Stripe config:", process.env);               // logs EVERY secret you have
```

Use your logger's **redaction** feature (shown above with pino), log specific fields instead of whole objects, and search your codebase for `console.log(req`, `print(request`, and `process.env` in log statements. Secrets handling in general is covered in [Module 2](02-security-vulnerabilities.md).

### 5. Error tracking: hear about it before your users tweet about it

Logs are where you *look*. Error tracking is what *tells you to look*.

An error tracker (e.g., **Sentry**, or alternatives like Rollbar, Bugsnag, Honeybadger, or the built-in options on some platforms) captures every unhandled exception from your frontend and backend, groups duplicates together, records the stack trace, browser, release version, and the user's recent actions ("breadcrumbs"), and notifies you.

Setup is usually 10 minutes:

1. Create a project in the tracker.
2. Install the SDK for your framework (most have a wizard for Next.js, React, Express, Django, FastAPI, etc.).
3. Set the DSN (the project key) as an environment variable.
4. Upload source maps for frontend code so stack traces point to your real files, not minified gibberish.
5. Tag events with environment (`production` vs `staging`) and release (git SHA).
6. Configure it to **scrub** sensitive data (most do some by default; check the settings).
7. Trigger a test error and confirm it arrives.

Then route new-issue notifications somewhere you'll actually see them (email, Slack, phone push). An error tracker nobody checks is just a very organized black hole.

**Important:** a global error handler only sees errors that *escape*. If your code swallows them (section 1), the tracker never knows. Fix the swallowing first.

### 6. Metrics: a few numbers that tell you if things are OK

Logs tell you what happened in one request. Metrics tell you how the whole system is doing over time. You don't need a hundred dashboards. Start with the **four golden signals** popularized by Google's SRE book ([Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)):

| Signal | Question | Example |
|---|---|---|
| **Latency** | How long do requests take? | p95 response time for `/api/checkout` |
| **Traffic** | How much demand is there? | Requests per minute, signups per hour |
| **Errors** | How many requests fail? | % of 5xx responses |
| **Saturation** | How full is the system? | DB connections used vs. limit, memory, queue length |

Add a couple of **business metrics** too, because they catch problems technical metrics miss: signups per hour, successful payments per hour, emails sent. If successful payments drop to zero on a Tuesday afternoon while error rates look normal, something is wrong, maybe in a way that doesn't throw errors at all.

Most hosting platforms and databases give you basic metrics for free. Use what's there before adding tools.

### 7. Alerts: wake me up only when it matters

An alert is a metric or error condition that notifies a human. The two ways to get alerts wrong:

- **No alerts:** you find out from customers.
- **Too many alerts:** you get 200 notifications a day, start ignoring them, and miss the real one. (This is called alert fatigue, and it's how real outages go unnoticed.)

Good alerts are:

- **About symptoms users feel:** error rate above 5% for 5 minutes, checkout latency above 3 seconds, zero successful payments in 30 minutes.
- **Actionable:** if you get the alert, there's something you can do.
- **Routed by urgency:** "site is down" goes to your phone; "disk 70% full" goes to email.
- **Tested:** trigger them once to confirm they actually arrive.

**A starter alert set for a small app:**

1. Uptime check fails (external service pings your health endpoint).
2. New error type appears in the error tracker.
3. 5xx error rate above a threshold for 5+ minutes.
4. A key business metric drops to zero for an unusual length of time.
5. Spend or usage crosses a budget threshold (more in [Module 8](08-deployment.md)).

### 8. Health checks

A health check is an endpoint like `/health` or `/api/health` that reports whether your app is alive and able to do its job. Hosting platforms, load balancers, and uptime monitors call it.

```ts
// app/api/health/route.ts
export async function GET() {
  const checks: Record<string, "ok" | "fail"> = {};
  try { await db.execute("SELECT 1"); checks.database = "ok"; }
  catch { checks.database = "fail"; }

  const healthy = Object.values(checks).every((v) => v === "ok");
  return Response.json(
    { status: healthy ? "ok" : "degraded", checks, version: process.env.GIT_SHA },
    { status: healthy ? 200 : 503 }
  );
}
```

Guidelines:

- **Liveness** ("is the process running?") should be cheap. **Readiness** ("can it serve traffic?") checks critical dependencies like the database.
- Don't expose sensitive details (connection strings, internal hostnames, versions of every library).
- Don't make it expensive. It might be called every few seconds.
- Point a free external uptime monitor at it so you know when the whole thing is down, not just when individual requests fail.

### 9. Debugging production without guessing

The vibe-coder debugging loop: see a vague complaint → paste a guess into the AI → deploy the AI's guess → hope. Repeat until something changes. This is how one bug becomes four.

**The evidence-first loop:**

1. **Reproduce or locate.** Get the error ID, time, user ID, or URL. Find the event in the error tracker or logs.
2. **Read the actual error.** The full message and stack trace, not just the first line.
3. **Gather context.** What request was it? What input? Which release? Did it start after a specific deploy? Is it one user or everyone?
4. **Form a hypothesis** that explains *all* the evidence, not just some of it.
5. **Verify the hypothesis** before fixing: reproduce locally or in staging with the same input, or add a targeted log line.
6. **Fix, with a test** that fails before the fix and passes after ([Module 4](04-testing.md)).
7. **Confirm in production:** the error stops appearing in the tracker.

Using AI in this loop is great, *if you give it evidence.* "It's broken, fix it" produces guesses. "Here is the stack trace, the request payload with secrets removed, the relevant code, and the commit where it started; what are three hypotheses and how would I verify each?" produces debugging.

Hard truth: if you can't explain *why* the fix works, you haven't fixed the bug. You've moved it.

### 10. Request IDs, end to end

"Use request IDs" is easy advice. Here's what it looks like wired up, in a small Express app (the same idea applies to Next.js middleware, FastAPI middleware, or any framework):

```js
// server.js — request IDs flowing through every log line
import express from "express";
import pino from "pino";
import pinoHttp from "pino-http";
import { randomUUID } from "node:crypto";

const logger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  redact: ["req.headers.authorization", "req.headers.cookie"],
});

const app = express();
app.use(express.json());

app.use(
  pinoHttp({
    logger,
    // Reuse an incoming ID (from a proxy or the frontend) or create one
    // (incoming IDs are user input: accept only short, simple values)
    genReqId: (req, res) => {
      const incoming = req.headers["x-request-id"];
      const id = typeof incoming === "string" && /^[\w-]{1,64}$/.test(incoming) ? incoming : randomUUID();
      res.setHeader("x-request-id", id); // the client can show/report it
      return id;
    },
  })
);

app.post("/api/orders", async (req, res, next) => {
  try {
    req.log.info({ itemCount: req.body.items?.length ?? 0 }, "creating order");
    const order = await createOrder(req.body, req.log, req.id);
    req.log.info({ orderId: order.id }, "order created");
    res.status(201).json(order);
  } catch (err) {
    next(err);
  }
});

async function createOrder(body, log, requestId) {
  log.debug("reserving inventory");
  // Pass the ID to downstream services so their logs can be joined to yours
  // await fetch(INVENTORY_URL, { headers: { "x-request-id": requestId }, ... })
  if (!body.items?.length) throw Object.assign(new Error("Order has no items"), { status: 422 });
  return { id: "ord_" + randomUUID().slice(0, 8), items: body.items };
}

// Global error handler: full details to logs, safe message + request ID to the user
app.use((err, req, res, _next) => {
  const status = err.status ?? 500;
  req.log.error({ err }, "request failed");
  res.status(status).json({
    error: status === 500 ? "Something went wrong. Please try again." : err.message,
    requestId: req.id,
  });
});

app.listen(3000);
```

Send one bad request with `x-request-id: req_7f3a` and the logs look like this (trimmed; pino-http also attaches request details to each line by default, which you can slim down with custom serializers):

```text
{"level":30,"time":...,"req":{"id":"req_7f3a","method":"POST","url":"/api/orders"},"itemCount":0,"msg":"creating order"}
{"level":50,"time":...,"req":{"id":"req_7f3a",...},"err":{"type":"Error","message":"Order has no items","stack":"...","status":422},"msg":"request failed"}
{"level":30,"time":...,"req":{"id":"req_7f3a",...},"res":{"statusCode":422},"responseTime":4,"msg":"request completed"}
```

And the user gets `{"error":"Order has no items","requestId":"req_7f3a"}`. Now the full chain works:

1. The user (or your support form) reports `req_7f3a`.
2. You search your logs for `req_7f3a` and get every line from that request, in order: what came in, what happened, what failed, how long it took.
3. If you passed the ID to downstream services (the commented-out `fetch` header), their logs join up too.
4. If you also tag your error tracker with it (most SDKs let you set a tag such as `request_id` per request), the exception and the logs point at each other.

Two details that matter: incoming IDs are **user input**, so validate them (the regex above) before echoing them into logs and responses; and in serverless platforms, many already generate a request ID for you. Use theirs if it's exposed, so your logs line up with the platform's.

### 11. A sample log-line schema

Consistency is what makes logs searchable. Pick field names once, write them down (in `AGENTS.md` too, so the AI uses them), and stick to them:

| Field | Example | Notes |
|---|---|---|
| `time` | `2026-10-17T22:41:07.312Z` | UTC, ISO 8601 (or epoch ms, consistently) |
| `level` | `info` / `warn` / `error` | Use the levels from section 3 |
| `msg` | `"order created"` | Short, constant, past tense. Put variables in fields, not in the message |
| `requestId` | `req_7f3a` | On every line of a request |
| `userId` / `accountId` | `u_123` | Internal IDs only, never emails or names |
| `route` | `POST /api/orders` | The route pattern, not the full URL with IDs and query strings |
| `durationMs` | `231` | For requests and slow operations |
| `status` | `422` | HTTP status, for request-completed lines |
| `err` | `{type, message, stack}` | Error objects, serialized by the logger |
| `release` | `a1b2c3d` | Git SHA of the deployed code, so you can tie errors to a deploy |
| `env` | `production` | Never mix environments in one search without knowing |
| *domain fields* | `orderId`, `invoiceId`, `jobId` | Whatever lets you find "the thing" later |

The rule about `msg` is the one people break most. `"order o_987 created for u_123"` can't be grouped or counted; `{"msg":"order created","orderId":"o_987","userId":"u_123"}` can. "How many orders were created per hour yesterday?" becomes a query instead of a regex adventure.

### 12. Alert fatigue: fewer, better alerts

The failure mode after "no alerts" is "so many alerts that none of them mean anything." Signs you're there: you've muted the alert channel; you glance at alerts and think "that one always fires"; real problems were reported by customers even though an alert technically fired.

How to keep alerts meaningful:

- **Every alert must have an action.** If the honest response is "huh, okay," it's not an alert. Turn it into a dashboard item or a weekly report.
- **Alert on symptoms, not causes.** "Checkout error rate above 5% for 5 minutes" (users are hurting) beats "CPU above 80%" (maybe nothing is wrong).
- **Use durations.** A single failed request isn't an incident. Five minutes of sustained failure is.
- **Two severities are enough for a small app.** *Page* (wake me up: site down, payments failing, data at risk) and *ticket* (look at it today: disk filling, a new low-volume error type). Route them differently.
- **Group and deduplicate.** Error trackers group identical exceptions; make sure you're notified on *new* issues and *regressions*, not on every occurrence.
- **Review monthly.** For each alert that fired: was it real, did someone act, was it fast enough? Delete or tune the noisy ones. Add one for any incident that no alert caught.
- **Write a one-line runbook into the alert.** "Checkout errors high → check error tracker for newest release; roll back if it started after a deploy (see playbook)." At 3 a.m., you won't remember.

### 13. A minimal observability stack for solo builders

You don't need an enterprise observability platform. A sensible starting stack for a solo builder or small team, most of which is free or cheap at small scale (check current pricing; free tiers change):

| Need | Start with | Notes |
|---|---|---|
| Logs | Your hosting platform's built-in logs (most collect stdout) + a structured logger (pino, structlog) | Upgrade to a log service when you need longer retention or better search |
| Errors | Sentry or a similar error tracker, frontend + backend | Usually the highest-value 15 minutes in this module |
| Uptime | An external uptime monitor (UptimeRobot, Better Stack, or similar) hitting `/health` every 1-5 minutes | Must run *outside* your infrastructure |
| Metrics | Your platform's and database provider's built-in dashboards | Request counts, latency, error rates, DB connections |
| Business signals | A daily query or tiny dashboard: signups, orders, payments | A scheduled job that emails you yesterday's numbers works fine |
| Costs | Billing alerts on every provider ([Module 8](08-deployment.md)) | A cost spike is often your first sign of a bug or abuse |
| Notifications | One channel you actually watch (phone push for pages, email for tickets) | Not a Slack channel you muted in March |

Setup order if you're starting from nothing: error tracker → structured logging with request IDs → health endpoint + uptime monitor → billing alerts → one business-metric check. That's an afternoon, and it moves you from "you don't know" to "you'll know."

### 14. Worked case: debugging from evidence

*(Illustrative case, built from very common patterns.)*

**Symptom:** Tuesday 2:10 p.m., two customers email "checkout is broken." A third says it worked fine.

**The vibe approach** would be pasting "checkout is broken, fix it" into the AI, which would guess (maybe the Stripe key? maybe CORS?), change three things, and deploy. Instead:

**1. Locate.** The error tracker shows a new issue since 1:52 p.m.: `TypeError: Cannot read properties of undefined (reading 'priceCents')` in `calculateTotal`, release `9c41e2a`, 37 events, 19 users.

**2. Read the whole error.** The stack trace goes `POST /api/checkout → orderService.createOrder → pricing.calculateTotal`, failing at `item.product.priceCents`.

**3. Gather context.** Release `9c41e2a` was deployed at 1:48 p.m. Filtering logs by `route: POST /api/checkout` and `level: error`, every failure has `itemCount` ≥ 1 and includes product `p_221`. Successful checkouts in the same period don't include `p_221`. Looking at `git show 9c41e2a`: an AI-written change that "optimized" the product query to exclude inactive products. Product `p_221` was marked inactive that morning but is still in some customers' carts.

**4. Hypothesis that explains *all* the evidence:** the new query returns no product for inactive items still sitting in carts, so `item.product` is `undefined`. It explains the timing (deploy at 1:48), the subset of users (only carts containing `p_221`), and the exact error.

**5. Verify before fixing.** In staging, add an inactive product to a cart and check out. Same `TypeError`. Confirmed.

**6. Mitigate, then fix.** Roll back to the previous release to stop the bleeding ([Module 8](08-deployment.md)); errors stop within minutes. Then write a failing test first ("checkout with an inactive product in the cart returns a clear 409 'item no longer available' and does not crash"), fix the code to handle missing products explicitly, and ship through CI.

**7. Confirm.** After deploy, the issue stays quiet in the error tracker; checkout success rate returns to normal. Reply to the three customers.

Total time: about 40 minutes, most of it spent *reading*, not changing. Notice that the AI was useful at step 3 ("explain what this commit changed in the product query") and step 6 ("write a test for this scenario"), as a tool working from evidence instead of an oracle guessing in the dark.

### 15. Designing error responses: problem details (RFC 9457)

Section 2 said "generic message plus an error ID." Here's how to make that consistent across a whole API, so your frontend, your logs, and any API customers all know what an error looks like. There's a standard for it: [RFC 9457, Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html) (2023, replacing RFC 7807). An error response uses the media type `application/problem+json` and these fields:

- `type`: a URL identifying the kind of problem (it can point to a docs page), or `about:blank` for a plain HTTP error
- `title`: a short, human-readable summary of that kind of problem
- `status`: the HTTP status code
- `detail`: an explanation specific to this occurrence
- `instance`: which request or resource it happened on
- plus any extra fields you need (`errors` for field validation, `errorId` for support)

Compare what AI-generated APIs typically return across five endpoints:

```text
{ "error": "bad email" }                   ← 400
{ "message": "Forbidden", "ok": false }    ← 200 (!)
{ "err": { "code": "P2002", "meta": ... } } ← 500, Prisma internals leaked
"Something went wrong"                     ← 500, plain text
{ "success": false, "reason": "upgrade" }  ← 402
```

Five shapes means the frontend needs five parsing branches, and one of them hides a failure behind a 200. Here's one central handler that makes every error look the same, in Express:

```js
// problems.js — every error response follows RFC 9457 "problem details"
import { randomUUID } from "node:crypto";

const TYPE_BASE = "https://example.com/problems/"; // each type URL can be a docs page

export class HttpProblem extends Error {
  constructor(status, slug, title, detail, extra = {}) {
    super(detail);
    Object.assign(this, { status, slug, title, detail, extra });
  }
}

export const problem = {
  validation: (errors) =>
    new HttpProblem(422, "validation-error", "Your request has invalid fields.",
      "Fix the fields listed in errors and try again.", { errors }),
  notFound: (what) =>
    new HttpProblem(404, "not-found", "Not found.", `${what} doesn't exist or you can't access it.`),
  outOfStock: (productIds) =>
    new HttpProblem(409, "out-of-stock", "Some items are no longer available.",
      "Remove the unavailable items from your cart and try again.", { productIds }),
};

export function problemHandler(logger) {
  // Express recognizes error handlers by their four arguments
  return (err, req, res, next) => {
    if (res.headersSent) return next(err);
    const known = err instanceof HttpProblem;
    const status = known ? err.status : 500;
    const errorId = randomUUID();

    if (status >= 500) logger.error({ err, errorId, reqId: req.id, path: req.originalUrl }, "unhandled error");
    else logger.warn({ errorId, reqId: req.id, status, type: err.slug }, "request rejected");

    res.status(status).type("application/problem+json").json({
      type: known ? TYPE_BASE + err.slug : "about:blank",
      title: known ? err.title : "Internal Server Error",
      status,
      detail: known ? err.detail
        : "Something went wrong on our side. Try again, and contact support with the error ID if it keeps happening.",
      instance: req.originalUrl,
      errorId,
      ...(known ? err.extra : {}),
    });
  };
}
```

```js
// app.js
import express from "express";
import pino from "pino";
import { problem, problemHandler } from "./problems.js";

const logger = pino();
const app = express();
app.use(express.json());

const products = new Map([["p_100", { id: "p_100", inStock: true }], ["p_221", { id: "p_221", inStock: false }]]);

app.post("/api/orders", async (req, res) => {
  const { items } = req.body ?? {};
  if (!Array.isArray(items) || items.length === 0) {
    throw problem.validation([{ field: "items", message: "Add at least one item." }]);
  }
  const unavailable = items.filter((i) => !products.get(i.productId)?.inStock).map((i) => i.productId);
  if (unavailable.length) throw problem.outOfStock(unavailable);
  res.status(201).json({ orderId: "ord_1" });
});

app.get("/api/boom", async () => {
  throw new Error("connect ECONNREFUSED 10.0.0.5:5432"); // a real internal error: never shown to users
});

app.use(problemHandler(logger)); // after all routes; Express 5 forwards async errors here
app.listen(3000);
```

What the client actually receives:

```text
$ curl -i -X POST localhost:3000/api/orders -H 'Content-Type: application/json' \
    -d '{"items":[{"productId":"p_100"},{"productId":"p_221"}]}'
HTTP/1.1 409 Conflict
Content-Type: application/problem+json; charset=utf-8

{"type":"https://example.com/problems/out-of-stock","title":"Some items are no longer available.",
 "status":409,"detail":"Remove the unavailable items from your cart and try again.",
 "instance":"/api/orders","errorId":"86f3c5e8-...","productIds":["p_221"]}

$ curl localhost:3000/api/boom
{"type":"about:blank","title":"Internal Server Error","status":500,
 "detail":"Something went wrong on our side. Try again, and contact support with the error ID if it keeps happening.",
 "instance":"/api/boom","errorId":"3edb948e-..."}
```

Notice what the 500 does *not* contain: `ECONNREFUSED 10.0.0.5:5432`. That string (your database's private IP and port) went to the logs with the same `errorId`, where you can find it. The user gets something useful (the 409 even tells the frontend exactly which product to highlight), and an attacker gets nothing.

### 16. Wiring up Sentry for real

Section 5 described what an error tracker does. Here's the actual setup for the two most common vibe-coded stacks, following [Sentry's current docs](https://docs.sentry.io/platforms/javascript/). (Other trackers look similar.)

**Express (Node).** Sentry needs to load *before* your app code, so it can hook into Express and HTTP. Put the setup in its own file:

```js
// instrument.js — loaded before anything else so Sentry can hook into Express and HTTP
import * as Sentry from "@sentry/node";

Sentry.init({
  dsn: process.env.SENTRY_DSN,               // from env vars, different per environment
  environment: process.env.APP_ENV ?? "development",
  release: process.env.GIT_SHA,              // which deploy introduced the error
  tracesSampleRate: 0.1,                     // sample 10% of requests for performance data
  sendDefaultPii: false,                     // don't attach IPs, cookies, or user data automatically
  beforeSend(event) {
    // Last line of defense: strip anything that looks like a credential
    if (event.request?.headers) {
      delete event.request.headers.authorization;
      delete event.request.headers.cookie;
    }
    return event;
  },
});
```

Start the app with `node --import ./instrument.js app.js` (that's the documented way for ES modules; with CommonJS, `require("./instrument")` on the first line of your app instead). Then register Sentry's error handler after your routes:

```js
// app.js — start with: node --import ./instrument.js app.js
import * as Sentry from "@sentry/node";
import express from "express";

const app = express();

app.get("/api/projects", async (req, res) => {
  res.json([]);
});

// After all routes, before any other error middleware:
Sentry.setupExpressErrorHandler(app);

// Your own handler still decides what the user sees (e.g. the problem+json handler above)
app.use((err, req, res, next) => {
  res.status(500).type("application/problem+json").json({
    type: "about:blank", title: "Internal Server Error", status: 500,
    errorId: res.sentry, // Sentry's event ID, so support can find the exact error
  });
});

app.listen(3000);
```

By default, Sentry's Express integration reports 5xx errors and skips 3xx and 4xx responses, which is what you want: a user typing a wrong coupon code isn't an incident. See the [Express guide](https://docs.sentry.io/platforms/javascript/guides/express/).

**Next.js.** Run the wizard: `npx @sentry/wizard@latest -i nextjs`. It creates the client config (`instrumentation-client.ts`), server and edge configs, an `instrumentation.ts` file that loads them, and wraps `next.config` with `withSentryConfig` so source maps get uploaded and stack traces point at your real code. Read the files it generates rather than trusting them blindly; check the [Next.js guide](https://docs.sentry.io/platforms/javascript/guides/nextjs/) for the current list, because the file layout has changed across versions.

**The three checks after setup:**

1. **Throw a test error in each environment** and confirm it shows up, tagged with the right environment and release.
2. **Look at one event's full payload.** Search it for tokens, emails, request bodies. If you see a password field, fix your scrubbing before real users arrive.
3. **Set up one alert rule** that reaches you (section 7), such as "new issue in production" sent to your phone. Default settings that email a folder you never open don't count.

### 17. Querying your logs: answers in seconds

Structured logs (section 3) pay off when something breaks and you need answers *fast*. Here's a slice of JSON logs, one object per line, the way pino and most loggers write them:

```text
{"level":30,"time":"2026-10-17T14:02:11.120Z","reqId":"req_7a1","route":"/api/projects","status":200,"durationMs":84,"userId":"u_12","msg":"request completed"}
{"level":30,"time":"2026-10-17T14:02:11.990Z","reqId":"req_7a3","route":"/api/checkout","msg":"retrying payment intent","attempt":2}
{"level":50,"time":"2026-10-17T14:02:13.002Z","reqId":"req_7a3","route":"/api/checkout","status":500,"durationMs":3012,"userId":"u_44","errorId":"e_91c","err":{"type":"Error","message":"stripe timeout"},"msg":"checkout failed"}
{"level":30,"time":"2026-10-17T14:02:13.950Z","reqId":"req_7a4","route":"/api/projects","status":200,"durationMs":2450,"userId":"u_12","msg":"request completed"}
{"level":50,"time":"2026-10-17T14:02:17.640Z","reqId":"req_7a7","route":"/api/projects","status":500,"durationMs":120,"userId":"u_77","errorId":"e_b13","err":{"type":"TypeError","message":"Cannot read properties of undefined (reading 'name')"},"msg":"list projects failed"}
```

On your laptop or a server, [`jq`](https://jqlang.org/) answers questions in one line each.

**"What's failing right now?"** All errors, trimmed to the useful fields:

```bash
jq -c 'select(.level >= 50) | {time, route, errorId, error: .err.message}' app.log
```

**"A customer emailed error ID `e_91c`."** Find the error, take its `reqId`, then follow that request's whole story:

```bash
jq -c 'select(.errorId == "e_91c") | .reqId' app.log    # → "req_7a3"
jq -c 'select(.reqId == "req_7a3")' app.log              # the retry, then the failure
```

**"Is it one bug or many?"** Group errors by route and message:

```bash
jq -r 'select(.level >= 50) | "\(.route)  \(.err.message)"' app.log | sort | uniq -c | sort -rn
```

**"What's slow?"**

```bash
jq -c 'select(.durationMs > 2000) | {time, route, durationMs}' app.log
```

**"Error rate per route"** (the number behind your alert):

```bash
jq -c -s 'map(select(.status)) | group_by(.route)
  | map({route: .[0].route, requests: length, errors: map(select(.status >= 500)) | length})' app.log
```

Hosted log tools (your platform's log viewer, Better Stack, Axiom, Datadog, Grafana Loki, and others) have their own query languages, but they express the same questions: *filter by level, filter by request ID, group by route, find the slow ones.* For example, in a Loki-style query, `{app="api"} | json | level >= 50` filters JSON logs by level. Learn the four questions; the syntax is a quick search away. None of it works if your logs are `console.log("here 2")`.

### 18. A gallery of silent failures

The scariest failures don't throw errors anyone sees. They happen in code with no user watching: background jobs, webhooks, and scheduled tasks. *(Each story is an illustrative composite of very common patterns.)*

#### The background job that failed quietly for three weeks

*The story:* Order receipts are sent by a background job. The email provider rotated its API keys; the job's `try/catch` logged "send failed" at debug level and moved on. Three weeks later, a customer's accountant asks why there are no receipts. 1,800 customers never got one, and some of them disputed the charges with their bank because they didn't recognize the charge. Each dispute costs a fee on top of the refund.

*The opposite failure:* a job with infinite retries and no backoff hammers a broken API 50 times a second, gets your account rate-limited, and blocks every other job in the queue.

*The fix:* bounded retries with backoff, then a **dead-letter** state that alerts a human. With [BullMQ](https://docs.bullmq.io/), for example:

```js
await receiptQueue.add("send-receipt", { orderId }, {
  attempts: 5,                                       // not 0, not infinite
  backoff: { type: "exponential", delay: 10_000 },   // 10s, 20s, 40s, 80s...
});

worker.on("failed", (job, err) => {
  if (job && job.attemptsMade >= (job.opts.attempts ?? 1)) {
    logger.error({ err, jobId: job.id, orderId: job.data.orderId }, "receipt permanently failed");
    Sentry.captureException(err, { tags: { queue: "receipts" } }); // a human hears about it
  }
});
```

Also add a metric on queue depth (jobs waiting). A queue growing for an hour is a symptom users haven't noticed yet.

#### The webhook that said "OK" and did nothing

*The story:* The Stripe webhook handler returns `200` first, *then* updates the database. When the database update throws, Stripe already got its 200 and never retries. The customer paid, and their account still says Free. They email support, angrily, on a Saturday.

*The opposite story:* the handler processes the event, then crashes before responding. Stripe retries (in live mode, for up to three days, per [Stripe's webhook docs](https://docs.stripe.com/webhooks)), and every retry grants another month of credits, because the handler doesn't check whether it already processed that event.

*The fix:* verify the signature (see [Module 2](02-security-vulnerabilities.md) on webhook verification), process *before* responding, return a non-2xx when processing fails so the sender retries, and make processing **idempotent** by recording event IDs:

```js
app.post("/webhooks/stripe", express.raw({ type: "application/json" }), async (req, res) => {
  let event;
  try {
    event = stripe.webhooks.constructEvent(
      req.body, req.headers["stripe-signature"], process.env.STRIPE_WEBHOOK_SECRET);
  } catch {
    return res.status(400).send("Invalid signature");
  }

  // Claim the event ID. A duplicate delivery inserts nothing and is acknowledged.
  const claimed = await db.query(
    "INSERT INTO processed_webhooks (event_id) VALUES ($1) ON CONFLICT DO NOTHING", [event.id]);
  if (claimed.rowCount === 0) return res.sendStatus(200);

  try {
    await handleStripeEvent(event);       // update the subscription, grant credits, etc.
    res.sendStatus(200);
  } catch (err) {
    await db.query("DELETE FROM processed_webhooks WHERE event_id = $1", [event.id]); // let the retry try again
    logger.error({ err, eventId: event.id, type: event.type }, "webhook processing failed");
    res.sendStatus(500);                  // Stripe will retry
  }
});
```

(If your handler's database writes can share one transaction with the `processed_webhooks` insert, that's even cleaner: both commit or neither does.)

#### The cron job that simply stopped

*The story:* A nightly job emails customers their usage report and cleans up expired trials. During a platform migration, the cron schedule doesn't get copied over. Nothing errors, because nothing runs. Six weeks later, finance notices that 300 expired trials are still using Pro features for free.

Error trackers can't catch this: **the absence of a run produces no error.** The fix is a **heartbeat** (also called a dead man's switch): the job pings a monitoring URL when it finishes successfully, and the monitor alerts you when the ping *doesn't* arrive on schedule. Services like [Healthchecks.io](https://healthchecks.io/), Cronitor, Better Stack, and Sentry's cron monitoring all do this:

```js
async function nightlyJob() {
  await expireTrials();
  await sendUsageReports();
  await fetch(process.env.NIGHTLY_JOB_HEARTBEAT_URL); // only reached if everything above succeeded
}
```

Configure the monitor with the expected schedule (daily at 02:00) and a grace period (an hour). Now "the job didn't run" becomes a message on your phone the next morning instead of a finance discovery six weeks later.

**The pattern across all three:** for anything that runs without a user watching, ask "if this stopped working tonight, how would I find out?" If the answer is "a customer would tell me eventually," add the alert now.

---

## What Good Looks Like

- No empty `catch` blocks. Every catch either handles a specific case or adds context and rethrows.
- A global error handler on the server and an error boundary on the frontend, both reporting to an error tracker.
- Users see friendly messages with an error ID; you see full details in logs.
- Logs are structured JSON with request IDs, never containing secrets or unnecessary PII.
- Error tracking is set up for frontend and backend, tagged by environment and release, with notifications you actually read.
- A health check endpoint and an external uptime monitor.
- A handful of meaningful alerts on user-facing symptoms and key business metrics.
- When something breaks, you start with evidence, not guesses.

**Observability checklist:**

- [ ] Search the codebase for empty or log-only `catch` blocks and bare `except:`. Fixed?
- [ ] Unhandled promise rejections and background job failures are caught and reported?
- [ ] API responses use correct status codes?
- [ ] User-facing errors contain no stack traces, SQL, or file paths?
- [ ] Logger configured with levels, JSON output, request IDs, and redaction?
- [ ] No secrets, tokens, passwords, or full request bodies in logs?
- [ ] Error tracker installed on frontend and backend, test error received?
- [ ] Health endpoint exists and an uptime monitor calls it?
- [ ] At least 3 alerts configured and tested?

---

## Prompt Templates

**1. Hunt for swallowed errors**

```text
Find every try/catch, .catch(), and except block in this codebase. For each,
classify it as: (a) handles a specific expected error correctly, (b) swallows
errors silently, (c) logs and continues as if nothing happened, or (d) returns
a fake success. For (b), (c), and (d), propose a fix that either handles the
specific case or adds context and rethrows to the global handler.
Also list any promises that are not awaited or have no error handling.
```

**2. User-facing vs. internal errors**

```text
Implement consistent error handling for our API: a typed AppError class with
an HTTP status, a safe user-facing message, and an internal error code; a
global error handler that logs full details (with a generated errorId and
requestId) and returns only { error: safeMessage, errorId } to the client.
Never return stack traces, SQL, or internal paths in responses. Use correct
HTTP status codes (400/401/403/404/409/422/429/500).
```

**3. Structured logging with redaction**

```text
Replace console.log/print statements in server code with a structured logger
([pino/structlog]). Configure JSON output, log levels from an env var, a
per-request child logger with requestId and userId, and redaction for
authorization headers, cookies, passwords, tokens, and API keys. Do not log
full request bodies on auth, payment, or profile routes. List every log
statement you removed or changed.
```

**4. Secrets-in-logs audit**

```text
Search the codebase for any log or print statement that could output secrets
or personal data: whole request/response objects, headers, process.env or
os.environ, user objects, tokens, passwords, card data, emails. List each with
file and line and a safer replacement that logs only IDs and non-sensitive
fields.
```

**5. Error tracking setup**

```text
Set up [Sentry] for both the frontend and backend of this [framework] app.
Read the DSN from an environment variable, tag events with environment and
release (git SHA), enable source map upload for the frontend, enable PII
scrubbing, and add a temporary /api/debug-error route (disabled in production
by env flag) so I can verify events arrive. Show me exactly what to configure
in the dashboard.
```

**6. Evidence-based debugging**

```text
Here is a production error. Do not propose a fix yet.
- Error and full stack trace: [paste]
- Request (secrets removed): [paste]
- Relevant code: [paste or file paths]
- When it started / recent deploys: [info]
Give me 3 hypotheses that explain ALL of this evidence, and for each, a
specific way to verify or rule it out. Then we'll fix the confirmed one with
a regression test.
```

---

## Exercise

**Part A: The swallow hunt (20 minutes)**

Search your project for error-swallowing patterns:

```bash
# Rough searches; review each hit by hand
rg -n "catch\s*\(\w*\)\s*\{\s*\}" src
rg -n "catch\s*\(" src
rg -n "except:\s*$|except Exception:\s*$" .
rg -n "\.catch\(\(\)\s*=>" src
```

For each hit, classify it with the categories from Prompt Template 1. Fix the three worst ones today.

**Part B: Break it on purpose (20 minutes)**

1. Install an error tracker on your project (free tiers are enough).
2. Temporarily break something in staging or locally (wrong DB password, a thrown error in a route).
3. Answer: Did the user see a friendly message? Did the error show up in the tracker? Could you find the log line by error ID? How long did it take you to know something was wrong?

**Part C: The log audit (15 minutes)**

Run Prompt Template 4. Fix anything that logs secrets or full request bodies. If you've ever shipped secrets to a log service, rotate those secrets (see [Module 2](02-security-vulnerabilities.md)).

**Part D: Minimum viable monitoring (20 minutes)**

Add a `/health` endpoint that checks the database. Point a free uptime monitor at it. Set up one alert for errors and one for a key business metric. Trigger each once to make sure the notification reaches you.

---

## Key Takeaways

- An error you swallow is an error you'll discover from a customer, a review, or your bank balance.
- Only catch errors you can handle. Otherwise add context and let them reach a global handler.
- Users get friendly messages and an error ID. You get the stack trace. Never mix them up.
- Use correct HTTP status codes. "200 OK: error" breaks everything downstream.
- Structured logs with request IDs turn "something broke" into "this broke, for this user, at this time, because of this."
- Logs are semi-public. Never log secrets, tokens, passwords, or full request bodies; redact by default.
- Error tracking tells you to look. Set it up on frontend and backend, and read the notifications.
- Track a few metrics (latency, traffic, errors, saturation, plus business numbers) and alert on symptoms users feel.
- A health check plus an uptime monitor means you know when you're down before Twitter does.
- Debug with evidence. Give the AI the stack trace and context, ask for hypotheses, verify before fixing, and add a regression test.

---

**Next up:** [Module 7: Data Modeling & Persistence](07-data-modeling.md) · [Back to course outline](../course-outline.md)
