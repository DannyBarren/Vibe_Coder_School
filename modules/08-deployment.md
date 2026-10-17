# Module 8: Deployment, Environments & Production Reality

> "It works on my machine."
> Great. We'll ship your machine to every customer.

Clicking "Deploy" has never been easier. Connect a repo, press a button, and your app is live on the internet with a URL and a certificate in under a minute. That's genuinely amazing.

It's also a trap. The ease of deploying hides everything that makes production different from your laptop: real users, real data, real money, real attackers, real bills, and no Ctrl+Z. Production is where "mostly works" meets "the whole internet," and the internet is not gentle.

This module covers the operational hygiene that keeps a one-click deploy from turning into a one-click disaster: separate environments, configuration and secrets per environment, keeping AI agents away from production, CI/CD and preview deploys, rollbacks, migrations during deploys, cost and rate-limit surprises, and what to actually do when things go wrong.

---

## Learning Objectives

By the end of this module you will be able to:

1. Explain why you need separate dev, staging, and production environments, and what belongs in each.
2. Manage environment variables per environment without leaking secrets.
3. Set up guardrails so AI agents never touch production credentials or data.
4. Use CI/CD and preview deploys so every change is tested before it's live.
5. Roll back a bad deploy in minutes, and know which things a rollback *can't* undo.
6. Run database migrations during deploys without downtime.
7. Prevent cost surprises from runaway serverless functions, AI API calls, and abuse.
8. Use rate limits to protect your app and your wallet.
9. Follow a simple incident playbook when production breaks.

---

## Vibe Fail: The $4,000 Weekend

*(Illustrative story; the ingredients are all real and common.)*

Jordan shipped an AI-powered "summarize any PDF" tool on Friday afternoon. Serverless functions, a popular LLM API, a database, all on generous free or pay-as-you-go tiers. The deploy took 40 seconds. Jordan went camping.

Here's what happened while Jordan was out of signal:

- The summarize endpoint had **no authentication and no rate limit.** Someone found it and started using it as a free LLM proxy, sending thousands of requests an hour.
- A retry loop the AI wrote ("retry on failure, up to forever") meant every timeout from the LLM API triggered more calls.
- The same `.env` file was used locally and in production, so the LLM key was the developer key with **no spending cap.**
- The serverless platform happily scaled to meet demand. That's its job.
- No billing alerts were set up. No error alerts either.

Monday morning: a bill of about $4,000 from the LLM provider, a few hundred more from the hosting platform, and an email from the provider about suspicious usage. The app had nine real users.

None of this was a hack in any clever sense. It was an open door, an infinite loop, no limits, and nobody watching.

---

## Core Concepts

### 1. Environments: dev, staging, production

An **environment** is a complete, separate copy of your app's running setup: code, configuration, database, and connected services.

| Environment | Purpose | Data | Who uses it |
|---|---|---|---|
| **Development (local)** | Building and experimenting | Fake/seed data | You (and your AI agent) |
| **Preview** | One per branch/PR, to review a change | Fake or copied-and-scrubbed data | You, reviewers |
| **Staging** | Final rehearsal; as close to production as possible | Realistic but not real (or anonymized) | You, testers |
| **Production** | Real users | Real data, real money | Customers |

The key rule: **environments don't share databases, secrets, or third-party accounts that move real money or send real messages.** Development uses test-mode Stripe keys, a sandbox email provider, and a local or dev database. Production uses live keys that only production can access.

Why it matters: in July 2025, an AI agent on Replit deleted a user's production database during a declared code freeze. One contributing factor Replit acknowledged was that, at the time, development and production shared a database, so agent actions during development could hit live data. Replit then rolled out separate development and production databases by default ([Replit blog](https://replit.com/blog/doubling-down-on-our-commitment-to-secure-vibe-coding); [The Register](https://www.theregister.com/software/2025/07/21/vibe-coding-service-replit-deleted-production-database/719783)). As the user himself put it, "you can't not separate preview and staging and production cleanly."

For a small app, you don't need four environments on day one. You do need **at least two: local and production, completely separated**, and you'll want staging as soon as real money or real data is involved.

### 2. Environment variables per environment

Configuration that differs between environments (database URLs, API keys, feature flags, log levels) belongs in **environment variables**, not in code.

```text
.env.example          # committed: lists every variable with fake values
.env.local            # NOT committed: your local values
(hosting dashboard)   # production and staging values, set per environment
```

```bash
# .env.example  (commit this one)
DATABASE_URL=postgres://user:pass@localhost:5432/myapp_dev
STRIPE_SECRET_KEY=sk_test_replace_me
OPENAI_API_KEY=replace_me
NEXT_PUBLIC_APP_URL=http://localhost:3000
LOG_LEVEL=debug
```

Rules:

- `.env` files with real values are in `.gitignore`. Always. Check now.
- Each environment has **its own** values: different database, different keys, different webhooks.
- **Prefixes matter.** In many frameworks, variables starting with `NEXT_PUBLIC_`, `VITE_`, `PUBLIC_`, or `REACT_APP_` are bundled into browser code and visible to anyone. Never put secrets in them.
- Validate config at startup (e.g., with a Zod schema) so the app fails immediately with a clear message if something's missing, instead of failing mysteriously on the first payment.
- Read env vars in one config module ([Module 5](05-architecture.md)'s single source of truth).

Secrets management in depth (rotation, scanning for leaks, vaults) is covered in [Module 2](02-security-vulnerabilities.md).

### 3. Never let the agent touch production

This deserves its own section because AI agents changed the risk profile. An agent with terminal access and production credentials can run migrations, delete data, rotate keys, or "clean up" resources, quickly, confidently, and sometimes while telling you it didn't.

**Agent guardrails:**

- **No production credentials on your development machine**, or at least not in any file or environment the agent can read. Production secrets live in your hosting platform's secret store and CI secrets.
- Agents work against **local or dev databases only.** If your tool supports it, configure dev/prod separation explicitly.
- **Deny-list destructive commands** in your agent's settings where supported: `DROP`, `TRUNCATE`, `rm -rf`, `git push --force`, `terraform destroy`, cloud CLI delete commands, production deploy commands.
- Require **human approval** for terminal commands that touch infrastructure, databases, or git history. Turn off "auto-run everything" for anything beyond your local sandbox.
- Production changes happen through **CI/CD and migrations**, reviewed and logged, not through an agent's terminal.
- Use **least-privilege** credentials: a read-only database user for debugging/analytics, scoped API keys, separate cloud accounts or projects for prod.
- Write these rules into your `AGENTS.md`/rules file ([Module 5](05-architecture.md)), but **don't rely on instructions alone.** Agents have ignored explicit instructions. Make it technically impossible, not just politely discouraged.

**Hard truth:** a freeze enforced by a sentence in a chat window is not a freeze. Access control is a freeze.

### 4. CI/CD: the path to production

**CI (Continuous Integration)** runs checks on every push: install, lint, typecheck, test, build ([Module 4](04-testing.md)). **CD (Continuous Deployment/Delivery)** automatically deploys code that passes those checks.

A healthy pipeline for a small app:

```text
push to feature branch
  → CI: lint, typecheck, unit + integration tests, build
  → preview deploy (unique URL, test-mode services)
  → you review the preview + the diff
merge to main (only if CI is green; branch protection on)
  → CI again
  → run database migrations (backward-compatible)
  → deploy to staging → smoke tests
  → deploy to production → smoke tests + watch errors/metrics
```

Principles:

- **Production is only changed by the pipeline.** No manual uploads, no "quick fix" directly on the server, no editing in the hosting dashboard's code editor.
- **Every deploy is traceable** to a commit SHA. You should be able to answer "what code is running in production right now?"
- **Small, frequent deploys** beat big, rare ones. A deploy with 3 changes is easy to debug. A deploy with 300 changes is a mystery novel.
- **Smoke tests after deploy:** a few automated checks that the homepage loads, login works, and the health endpoint is green ([Module 6](06-observability.md)).

### 5. Preview deploys

Many platforms (Vercel, Netlify, Render, Cloudflare Pages, and others) can create a **preview deployment** for every pull request: a live, unique URL running that branch's code.

They're excellent for reviewing AI changes in a real environment before merging. Make sure previews:

- Use **test-mode** keys and a non-production database (or a branch database if your provider supports database branching).
- Are **not publicly indexed** and, ideally, protected by password or team login. Preview URLs leak, and they often run with weaker settings.
- Don't send real emails or charge real cards.

### 6. Rollbacks: your emergency exit

A **rollback** returns production to the last known-good version. With most modern platforms it's one click or one command, because each deploy is an immutable build you can re-promote.

Know how to do it **before** you need it:

- Where is the rollback button or command on your platform?
- How long does it take?
- Who can do it?
- Have you tried it once on staging?

**What a rollback can't undo:**

- **Database changes.** If the bad deploy ran a migration that dropped a column, rolling back the code brings back code that expects a column that no longer exists. This is why migrations must be backward-compatible (next section) and why you need tested backups ([Module 7](07-data-modeling.md)).
- **Data written by buggy code** (wrong prices saved, corrupted records).
- **External side effects:** emails sent, charges made, webhooks fired, posts published.

**Feature flags** are the other half of the exit strategy: ship risky code turned off, turn it on for yourself, then a few users, then everyone. If it misbehaves, turn it off without a deploy.

And for anything that reaches many users at once, **roll out gradually.** On July 19, 2024, a faulty CrowdStrike content update crashed Windows machines worldwide; in its root-cause analysis, CrowdStrike committed to staged deployment of such updates, with canary testing, monitoring, and rollback ([CrowdStrike RCA](https://www.crowdstrike.com/wp-content/uploads/2024/08/Channel-File-291-Incident-Root-Cause-Analysis-08.06.2024.pdf); [CISA alert](https://www.cisa.gov/news-events/alerts/2024/07/19/widespread-it-outage-due-crowdstrike-update)). Your app is smaller. The lesson is the same: don't send a change to 100% of users before it's survived 1%.

### 7. Database migrations during deploy

During a deploy, there's a window where **old code and new code run at the same time** (old instances draining, new ones starting), and both talk to the same database. Your schema must work for both.

**Safe order of operations:**

1. **Migrate first, additively.** Add new columns (nullable or with defaults), new tables, new indexes. Old code ignores them.
2. **Deploy new code** that uses the new schema.
3. **Clean up later.** Remove old columns in a *future* deploy, once no running code uses them (the expand/contract pattern from [Module 7](07-data-modeling.md)).

**Danger list during deploys:**

- Renaming or dropping columns that the currently running code uses.
- Adding a `NOT NULL` column without a default to a table with existing rows.
- Long-running migrations that lock big tables (adding indexes on large Postgres tables should use `CREATE INDEX CONCURRENTLY`).
- Running migrations by hand from a laptop "real quick."

Knight Capital is the extreme cautionary tale of a deploy gone wrong: in 2012, new code was deployed to only seven of eight servers, with no second technician reviewing the deployment and no written procedure requiring it. The eighth server ran old code, which went haywire for about 45 minutes and lost the firm more than $460 million ([SEC order](https://www.sec.gov/files/litigation/admin/2013/34-70694.pdf)). Automated, reviewed, verified deploys are not bureaucracy. They are the cheap version of that lesson.

### 8. Cost surprises: the bill is a production incident too

Serverless and usage-based pricing are wonderful until they aren't. They scale to meet demand, including demand from bots, abuse, infinite loops, and your own bugs.

**Common causes of runaway bills:**

- Unauthenticated endpoints that call paid APIs (LLMs, image generation, SMS, maps).
- Retry loops without limits or backoff.
- Recursive triggers: a function writes to storage, which triggers the function, which writes to storage...
- `useEffect` loops or polling that hammer your API from every open browser tab.
- N+1 queries and missing indexes on usage-billed databases ([Module 7](07-data-modeling.md)).
- Logging everything at debug level to a paid log service.
- Leaked API keys used by strangers ([Module 2](02-security-vulnerabilities.md)).
- Large file egress (serving big images/videos directly instead of via a CDN).

**Defenses:**

- **Billing alerts and budgets** on every provider: hosting, database, LLM APIs, email/SMS, cloud. Set alerts at 50%, 80%, and 100% of what you expect to spend.
- **Hard spending caps** wherever the provider supports them (many LLM providers let you set monthly limits per project or key).
- **Separate API keys per environment and per app**, each with its own limit, so one leak or bug doesn't drain everything.
- **Authentication in front of anything that costs money per call.**
- **Retries with limits and exponential backoff**, never infinite.
- **Timeouts** on outbound calls and function execution.
- **Concurrency limits** on serverless functions where supported.

```ts
// BAD: retries forever, instantly
async function summarize(text) {
  while (true) {
    try { return await llm.complete(text); } catch { /* try again! */ }
  }
}

// GOOD: bounded retries, backoff, timeout, and only for retryable errors
async function summarize(text: string) {
  for (let attempt = 1; attempt <= 3; attempt++) {
    try {
      return await llm.complete(text, { timeoutMs: 20_000 });
    } catch (err) {
      if (!isRetryable(err) || attempt === 3) throw err;
      await sleep(2 ** attempt * 500); // 1s, 2s
    }
  }
}
```

### 9. Rate limits: protect your app and your wallet

A **rate limit** caps how many requests a user, IP, or API key can make in a window of time. It protects you from abuse, brute-force login attempts, scrapers, runaway clients, and your own bugs.

Where to apply them:

- **Login, signup, password reset, and verification codes** (strict, e.g., a handful per minute per IP and per account).
- **Anything that costs money per call** (LLM, SMS, email, image generation): per user, per day, tied to their plan.
- **Public APIs and forms** (contact forms, waitlists, search).
- **Webhook endpoints** (verify signatures too).

Implementation options include your platform's built-in firewall/rate-limit rules, an API gateway, or a library backed by a shared store like Redis (in-memory counters don't work across multiple serverless instances). Return **HTTP 429** with a `Retry-After` header when limits are hit.

Also respect **other people's** rate limits: your LLM, email, and payment providers will throttle you. Handle 429s from them with backoff, not by hammering harder.

### 10. The incident playbook

Something is broken in production. Customers are affected. Here's what to do, in order. Write a version of this down **now**, while you're calm.

**1. Detect and declare.** Alert fires, user reports, or you notice. Say it out loud (or in your team channel): "We have an incident." Start a timestamped notes doc.

**2. Assess impact.** Who's affected? All users or some? Is data being lost or exposed? Is money being lost? This decides urgency.

**3. Stop the bleeding (mitigate before you fix).**
- Recent deploy? **Roll back.** Don't debug forward under pressure.
- New feature? **Turn off the feature flag.**
- Cost runaway or abuse? **Disable the endpoint, revoke/rotate the key, tighten the rate limit.**
- Data being corrupted? **Stop the writes** (maintenance mode, disable the job).
- Security exposure? Revoke credentials, close access, and follow [Module 2](02-security-vulnerabilities.md).

**4. Communicate.** Tell affected users what's happening in plain language, even if it's just "We're aware of an issue with checkout and are working on it." Silence is worse than bad news.

**5. Diagnose with evidence.** Error tracker, logs, metrics, recent deploys ([Module 6](06-observability.md)). Use AI with real evidence, not vibes.

**6. Fix properly.** Through the normal pipeline, with a regression test ([Module 4](04-testing.md)). Not a hotfix typed directly into production.

**7. Recover data if needed.** Restore from backups into a separate database, compare, and carefully repair ([Module 7](07-data-modeling.md)). Never restore over production in a panic.

**8. Write a blameless postmortem.** What happened, timeline, impact, root cause, what went well, what didn't, and specific action items with owners. GitLab's 2017 database postmortem ([link](https://about.gitlab.com/blog/postmortem-of-database-outage-of-january-31/)) is a great public example of the format.

**Incident don'ts:**

- Don't let an AI agent "fix production" autonomously during an incident.
- Don't make five changes at once; you won't know which one helped.
- Don't delete logs or evidence.
- Don't hide it from affected users, especially if their data was involved; depending on where you and they are, you may have legal notification obligations.

---

## What Good Looks Like

- Local and production are fully separated (databases, keys, accounts); staging exists once real money or data is involved.
- Every environment variable is listed in `.env.example`; real values live only in local `.env` files (gitignored) and the hosting platform's secret store.
- No secrets in browser-exposed variables.
- AI agents cannot reach production credentials or data, by technical restriction, not just instruction.
- Production changes only through CI/CD: tests pass, preview reviewed, migrations run, deploy, smoke test.
- You know how to roll back, and you've practiced it.
- Migrations are backward-compatible; destructive cleanups happen in later deploys.
- Billing alerts and spending caps on every paid service; paid endpoints require auth and are rate-limited.
- A written incident playbook exists, and you'd actually follow it.

**Pre-launch production checklist:**

- [ ] Separate production database and production-only secrets?
- [ ] `.env` files gitignored; no secrets in repo history or client bundle?
- [ ] Agent tools have no access to production credentials?
- [ ] CI runs tests on every push; main branch protected?
- [ ] Preview deploys use test-mode services and aren't public?
- [ ] Rollback procedure known and tested?
- [ ] Migrations run via pipeline and are backward-compatible?
- [ ] Backups enabled and a restore tested ([Module 7](07-data-modeling.md))?
- [ ] Error tracking, health check, uptime monitor, and alerts in place ([Module 6](06-observability.md))?
- [ ] Billing alerts and spending caps on hosting, database, and every paid API?
- [ ] Rate limits on auth endpoints and anything that costs money?
- [ ] Incident playbook written down?

---

## Prompt Templates

**1. Environment separation audit**

```text
Audit this project's configuration. List every environment variable it uses,
where each is read, whether it's a secret, and whether it could end up in
browser code (NEXT_PUBLIC_/VITE_/PUBLIC_ prefixes or client imports). Generate
a .env.example with fake values, confirm .env files are gitignored, and add a
startup config validation (Zod or equivalent) that fails fast with a clear
message if a required variable is missing. Do not print any real secret
values.
```

**2. Agent guardrails**

```text
Draft a section for our AGENTS.md and a list of agent tool settings that:
restrict you to local/dev databases, forbid destructive commands (DROP,
TRUNCATE, rm -rf, force-push, production deploys, cloud resource deletion)
without my explicit approval, and forbid reading or using production
credentials. Also list technical controls I should set up (separate accounts,
least-privilege DB users, secret storage) so these rules don't depend on you
following instructions.
```

**3. CI/CD pipeline**

```text
Create a CI/CD setup for this [framework] app on [platform]: on every PR run
lint, typecheck, tests, and build, and create a preview deploy using test-mode
secrets. On merge to main, run migrations via our migration tool, deploy to
production, and run smoke tests against /api/health and the login page.
Production secrets must come from the platform's secret store. Explain how to
roll back.
```

**4. Migration safety during deploy**

```text
Here is a planned schema change: [change]. Assume old and new code run at the
same time during deploy. Produce a step-by-step plan (separate migrations and
deploys) that never breaks the running version, using expand/contract. Flag
any step that could lock a large table or lose data.
```

**5. Cost and abuse review**

```text
Review this codebase for cost and abuse risks: endpoints that call paid APIs
without authentication or rate limits, unbounded retries or loops, missing
timeouts on outbound calls, recursive triggers, client-side polling or effect
loops, and debug-level logging in production. For each, propose a fix (auth,
per-user rate limit, bounded retries with backoff, timeouts, caching). Then
list the billing alerts and spending caps I should configure on each provider
we use.
```

**6. Incident playbook**

```text
Write a one-page incident playbook tailored to this app's stack ([hosting,
database, payments, LLM provider]). Include exact steps (with where to click
or which command) to: roll back a deploy, disable a feature flag, put the app
in maintenance mode, rotate each critical secret, revoke an API key, and
restore a database backup to a NEW database. Add a user-communication
template and a postmortem template.
```

---

## Exercise

**Part A: Environment map (15 minutes)**

Make a table with columns: Local, Preview, Staging, Production. Rows: database, payment keys, email provider, LLM key, file storage, error tracking project. Fill in what each environment actually uses. Any cell where production and non-production share something is a finding. Fix the scariest one this week.

**Part B: Agent lockdown (15 minutes)**

Check: can your AI agent read any file or environment variable containing production credentials? Can it run commands without approval? Lock it down using Prompt Template 2.

**Part C: Rollback drill (20 minutes)**

On staging (or production during a quiet time, if you don't have staging yet): deploy a harmless visible change, then roll it back using your platform's rollback feature. Time it. Write the steps in your playbook.

**Part D: Money guardrails (20 minutes)**

Log into every paid service you use (hosting, database, LLM APIs, email/SMS, cloud). For each: set a billing alert and, where available, a hard spending cap. Then find every endpoint that calls a paid API and confirm it requires authentication and has a rate limit.

**Part E: Write your playbook (20 minutes)**

Use Prompt Template 6, then edit it until you'd trust it at 3 a.m. Save it somewhere you can find while the site is down (not only on the site that's down).

---

## Key Takeaways

- Deploying is easy. Operating is the job. Production is where real users, data, money, attackers, and bills live.
- Separate environments completely: local, preview, staging, production. No shared databases or live keys.
- Configuration lives in environment variables, per environment, with secrets only in gitignored files and secret stores. Never in browser-exposed variables.
- Keep AI agents away from production, using access control rather than polite instructions.
- Production changes only through CI/CD: tests, preview, migrations, deploy, smoke tests.
- Know how to roll back before you need to, and remember a rollback can't undo database changes or external side effects.
- Make migrations backward-compatible; old and new code run side by side during deploys.
- Usage-based pricing scales your mistakes. Set billing alerts and caps, require auth on paid endpoints, and bound every retry.
- Rate-limit authentication and anything that costs money.
- When things break: stop the bleeding first (rollback, flag off, revoke), communicate, diagnose with evidence, fix through the pipeline, and write a blameless postmortem.

---

**Next up:** [Module 9: Tech Debt, Refactoring & Knowing When to Stop Vibing](09-tech-debt.md)
