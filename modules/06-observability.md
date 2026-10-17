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

*(Illustrative story, assembled from very common patterns.)*

Leo launched a waitlist for his AI writing tool. He posted it on a few forums, the signups rolled in, and the counter on his landing page climbed past 2,000. He was thrilled.

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

Leo lost about 85% of his launch signups, and he couldn't even email them to apologize, because he didn't have their addresses. The fix would have been four lines of error handling and a free error-tracking account.

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

"Use request IDs" is easy advice. Here's what it looks like wired up, in a small Express app (the same idea applies to Next.js middleware, FastAPI middleware, or any framework). This code runs as shown:

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
- **Alert on symptoms, not causes.** "Checkout error rate above 5% for 10 minutes" (users are hurting) beats "CPU above 80%" (maybe nothing is wrong).
- **Use durations.** A single failed request isn't an incident. Sustained failure for 5 to 10 minutes is.
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

**Next up:** [Module 7: Data Modeling & Persistence](07-data-modeling.md)
