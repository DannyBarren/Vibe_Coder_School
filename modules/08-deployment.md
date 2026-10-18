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
*Illustrative composite.*

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

Why it matters: in July 2025, a Replit AI agent deleted a user's production database during a code freeze. One contributing factor was that development and production shared a database; Replit has since made them separate by default ([full timeline in Module 1](01-security-reality.md#29-anatomy-of-an-incident-2-replit-and-the-saastr-database)).

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

Module 1 covers agent guardrails in depth ([Part 5: your AI agent is part of the attack surface](01-security-reality.md#part-5-your-ai-agent-is-part-of-the-attack-surface)). The deployment-specific version fits in three rules:

- **Production credentials never live where the agent can read them.** They belong in your hosting platform's secret store and CI secrets, not in `.env` files on your laptop.
- **Agents work against local or dev databases only.** Anything that changes production goes through CI/CD and reviewed migrations (sections 4 and 7), never an agent's terminal.
- **Make it technically impossible, not politely discouraged.** A rule in `AGENTS.md` is a good reminder, but agents have ignored explicit instructions. Least-privilege credentials and human approval for destructive commands are what actually hold.

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

### 11. A sample pipeline: lint, test, build, preview, migrate, promote

Here's the CI/CD flow from section 4 as a real GitHub Actions workflow for a Next.js + Prisma app deployed to Vercel. It uses Vercel's documented CLI pattern (`vercel pull` → `vercel build` → `vercel deploy --prebuilt`; see [Vercel's guide](https://vercel.com/kb/guide/how-can-i-use-github-actions-with-vercel)). Swap the deploy steps for your platform's CLI if you're elsewhere; the shape stays the same.

```yaml
# .github/workflows/ci-cd.yml
name: ci-cd

on:
  pull_request:
  push:
    branches: [main]

# One run per branch at a time. A new push to a PR cancels the stale run;
# on main, runs queue instead, so a production deploy or migration is never cut off halfway.
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

env:
  VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
  VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready --health-interval 5s --health-timeout 5s --health-retries 10
    env:
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/postgres   # throwaway CI database
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npx prisma migrate deploy   # proves migrations apply cleanly to an empty DB
      - run: npm test
      - run: npm run build

  preview:
    if: github.event_name == 'pull_request'
    needs: test
    runs-on: ubuntu-latest
    environment: preview
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm install --global vercel@latest
      - run: vercel pull --yes --environment=preview --token=${{ secrets.VERCEL_TOKEN }}
      - run: vercel build --token=${{ secrets.VERCEL_TOKEN }}
      - name: Deploy preview
        id: deploy
        run: echo "url=$(vercel deploy --prebuilt --token=${{ secrets.VERCEL_TOKEN }})" >> "$GITHUB_OUTPUT"
      - run: echo "Preview ready at ${{ steps.deploy.outputs.url }}"

  production:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    needs: test
    runs-on: ubuntu-latest
    environment: production   # add required reviewers in GitHub settings = a manual "promote" button
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - name: Migrate production database (backward-compatible migrations only)
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: ${{ secrets.PROD_MIGRATION_DATABASE_URL }}
      - run: npm install --global vercel@latest
      - run: vercel pull --yes --environment=production --token=${{ secrets.VERCEL_TOKEN }}
      - run: vercel build --prod --token=${{ secrets.VERCEL_TOKEN }}
      - run: vercel deploy --prebuilt --prod --token=${{ secrets.VERCEL_TOKEN }}
      - name: Smoke test
        run: |
          curl --fail --silent --show-error --retry 5 --retry-delay 5 --retry-all-errors https://yourapp.example/api/health
          curl --fail --silent --show-error --output /dev/null https://yourapp.example/login
```

What each part buys you:

- **`test` runs on every PR and every push to main.** Lint, typecheck, migrations against a throwaway database, tests, build. If any step fails, nothing deploys.
- **`preview` deploys every PR** to its own URL with preview-environment variables (test-mode keys, non-production database). You review the running app *and* the diff before merging.
- **`production` only runs on main, after tests pass**, and uses a GitHub *environment* called `production`. In the repo settings you can require a reviewer for that environment, which turns the job into a "promote to production" button: the pipeline waits until you approve. Environment-scoped secrets (`PROD_MIGRATION_DATABASE_URL`) are only available to that job.
- **Migrations run before the new code goes live**, which is why they must be backward-compatible (section 7).
- **Smoke tests** confirm the health endpoint and login page respond after deploy. If they fail, you roll back (section 14).
- **`concurrency`** stops two runs of the same branch from racing each other. On pull requests, a newer push cancels the outdated run (saves CI minutes). On `main`, it **never cancels**: killing a job halfway through `prisma migrate deploy` or a production deploy can leave the database migrated but the old code live, which is exactly the half-finished state you're trying to avoid. Instead, the newer run waits its turn.

Setup notes:

- Add `VERCEL_TOKEN`, `VERCEL_ORG_ID`, `VERCEL_PROJECT_ID`, and `PROD_MIGRATION_DATABASE_URL` as GitHub secrets (the last one as an *environment* secret on `production`). Never paste them into the YAML.
- If Vercel's built-in Git integration is also connected, every push deploys twice. Pick one: either disable Git-triggered deploys or drop the deploy jobs here.
- Pull requests from forks don't get your secrets (by design), so preview deploys only work for branches in your own repo.
- If preview URLs are protected (they should be; section 5), smoke-testing them needs your platform's protection-bypass mechanism. Otherwise just click through the preview yourself.
- Use a dedicated database user for migrations with only the permissions migrations need, separate from the app's runtime user.

### 12. Environment variable matrix (template)

Keep this table in your repo (`docs/environments.md`) and update it whenever you add a variable. It answers "what does each environment use?" at a glance, and it's how you catch "wait, preview is using the live Stripe key."

| Variable | Secret? | Exposed to browser? | Local | Preview | Staging | Production | Owner / rotation |
|---|---|---|---|---|---|---|---|
| `DATABASE_URL` | Yes | No | local Docker DB | preview/branch DB | staging DB | prod DB (app user) | you, on staff change |
| `PROD_MIGRATION_DATABASE_URL` | Yes | No | n/a | n/a | n/a | prod DB (migration user, CI only) | you |
| `STRIPE_SECRET_KEY` | Yes | No | `sk_test_…` | `sk_test_…` | `sk_test_…` | `sk_live_…` (prod only) | yearly or on leak |
| `STRIPE_WEBHOOK_SECRET` | Yes | No | CLI-generated | test endpoint | test endpoint | live endpoint | on endpoint change |
| `OPENAI_API_KEY` | Yes | No | dev key, low cap | dev key, low cap | staging key | prod key, own cap | on leak |
| `SENTRY_DSN` | No* | Yes (frontend) | dev project or empty | staging project | staging project | prod project | n/a |
| `NEXT_PUBLIC_APP_URL` | No | Yes | `http://localhost:3000` | preview URL | staging URL | `https://yourapp.example` | n/a |
| `LOG_LEVEL` | No | No | `debug` | `info` | `info` | `info` | n/a |

\* Frontend error-tracker keys are designed to be public; scope them to their project and enable allowed-origin filtering where offered.

How to use it:

- Any row where Production shares a value with another column is a finding.
- Anything marked "Exposed to browser" must not be a secret. Check the framework prefix (`NEXT_PUBLIC_`, `VITE_`, `PUBLIC_`).
- The "Owner / rotation" column is what you'll want during an incident ([Module 2](02-security-vulnerabilities.md) covers rotation).

### 13. The cost-guardrail checklist

Run this before launch, and again whenever you add a paid service:

**Every provider (hosting, database, LLM, email/SMS, storage, maps, cloud):**

- [ ] Billing alert at roughly 50%, 80%, and 100% of expected monthly spend, going to a channel you read.
- [ ] Hard spending cap or usage limit set, where the provider offers one. Know what happens when it's hit (service stops vs. overage charges).
- [ ] Payment method has a sensible limit, or a virtual card with a cap, for side projects.
- [ ] You know where the usage dashboard is and have looked at it once since launch.

**API keys:**

- [ ] Separate keys per environment and per app. Never one key for everything.
- [ ] Per-key limits or budgets where the provider supports them.
- [ ] Keys scoped to the minimum permissions needed.
- [ ] No paid-API key in browser code. Calls go through your server.

**Your code:**

- [ ] Every endpoint that triggers a paid call requires authentication.
- [ ] Per-user rate limits and daily quotas on paid features, tied to their plan.
- [ ] Retries are bounded with backoff; outbound calls have timeouts.
- [ ] No recursive triggers (function → storage → same function).
- [ ] LLM calls set `max_tokens` (or equivalent) and cap input size.
- [ ] Caching for repeated, identical expensive calls where it's safe.
- [ ] Usage per user is logged, so you can spot one account burning 90% of the budget.

**Platform:**

- [ ] Serverless concurrency or instance limits set where supported.
- [ ] Bot protection or firewall rules on public endpoints that are being abused.
- [ ] Large files served via CDN/object storage, not through your functions.

### 14. The rollback drill

Rolling back under pressure, for the first time, while customers are emailing you, is a bad way to learn your platform's UI. Practice once a quarter. It takes 20 minutes.

1. **Pick a quiet time** and tell anyone who needs to know.
2. **Note the current production version** (deployment ID or git SHA). Most platforms show it; your `/api/health` should too ([Module 6](06-observability.md)).
3. **Ship a harmless, visible change** through the normal pipeline, e.g. a footer text "Drill 2026-10-17."
4. **Start a timer, then roll back** using your platform's rollback feature (on Vercel, "Instant Rollback" in the dashboard or `vercel rollback`; on other platforms, "redeploy previous release" or equivalent).
5. **Verify**: the footer text is gone, the health endpoint reports the old SHA, the error tracker shows no new issues. Stop the timer.
6. **Roll forward** (redeploy the latest version), or revert the drill commit and let the pipeline deploy.
7. **Write it down**: the exact clicks or commands, who has permission, how long it took, and anything that surprised you. Put it in the incident playbook (section 10).

Bonus round: do the drill with a **migration** involved, on staging. Deploy a change with an additive migration, roll back the code, and confirm the old code still works with the new schema. If it doesn't, your migrations aren't backward-compatible yet ([Module 7](07-data-modeling.md)).

Remember: some platforms stop auto-promoting new deployments after a rollback. Vercel, for example, turns off automatic assignment of production domains after an Instant Rollback, so new pushes won't go live until you promote a deployment again (`vercel promote`; see [Vercel's rollback docs](https://vercel.com/docs/instant-rollback)). Know how yours behaves, or you'll spend the next morning wondering why your fix isn't live.

### 15. Launch-day runbook

Launching (on Product Hunt, Hacker News, to a mailing list, or to a paying client) is a planned traffic spike with an audience watching. Treat it like one.

**One week before**

- [ ] Pre-launch production checklist (below) fully checked.
- [ ] Load-relevant limits reviewed: database connection limits and plan, serverless concurrency, email sending limits, LLM rate limits and tier.
- [ ] Rate limits on signup, login, and paid features tested.
- [ ] Billing alerts and caps set (section 13).
- [ ] Backup restore tested ([Module 7](07-data-modeling.md)); rollback drill done (section 14).
- [ ] Error tracker, uptime monitor, and alerts verified by triggering each one ([Module 6](06-observability.md)).
- [ ] A status page or a pinned post where you'll communicate if something breaks.

**The day before**

- [ ] **Code freeze.** No new features. Only fixes, through the pipeline. (Enforced by you, not by asking an agent nicely.)
- [ ] Full run-through on production: sign up as a new user, do the core action, pay with a real card and refund it, receive every email.
- [ ] Incident playbook open in a tab; secrets-rotation steps and rollback steps at hand.
- [ ] Decide what you'll turn off if load is too high (feature flags for expensive features, signups waitlist).

**Launch hour**

- [ ] Watch, don't build: error tracker, logs, platform metrics, database connections, billing/usage dashboards.
- [ ] Do the full user journey yourself once traffic arrives.
- [ ] Any new error type gets triaged immediately: ignore, fix-forward through the pipeline, flag off, or roll back.

**First 24-48 hours**

- [ ] Check spend against expectations every few hours.
- [ ] Reply to user-reported bugs quickly, even if just "we're on it."
- [ ] Keep a list of everything that went wrong or almost did. Turn it into tests, alerts, and runbook updates next week.
- [ ] Only then lift the code freeze.

### 16. Walkthrough: Vercel + Supabase, from laptop to production

Here's the whole path for one of the most common vibe-coded stacks, step by step: a Next.js app on Vercel with Supabase for the database and auth. (Neon, Railway, Render, and others follow the same shape; notes at the end.) Commands follow the [Supabase CLI docs](https://supabase.com/docs/guides/local-development/cli-workflows); check them for current flags.

**Step 1: three separate databases, not one.** Local (on your machine), a staging Supabase project, and a production Supabase project. Separate projects mean separate data, separate keys, and separate auth users. A bug or an agent in development *physically cannot* reach production data, because it doesn't have the address.

**Step 2: develop against a local database.**

```bash
supabase init                              # creates supabase/ with config and a migrations folder
supabase start                             # runs Postgres, Auth, and Studio locally (needs Docker)
supabase migration new create_projects     # creates supabase/migrations/<timestamp>_create_projects.sql
# ...write the SQL (tables, constraints, RLS policies) in that file...
supabase db reset                          # rebuilds the LOCAL database from all migrations + seed.sql
```

`supabase db reset` is your best friend locally, because it proves your migrations build the whole schema from nothing. That's what production will rely on. Commit the migration files with the code that uses them.

**Step 3: push schema changes to staging, then production, through migrations.**

```bash
supabase login
supabase link --project-ref <staging-project-ref>
supabase db push --dry-run                 # shows which migrations WOULD run. Read it.
supabase db push                           # applies pending migrations to the linked project
```

Then the same for production, ideally from CI rather than your laptop (the pipeline in section 11 runs migrations in a gated `production` job; swap `prisma migrate deploy` for `supabase db push` if Supabase owns your migrations). Two commands that should never touch production: `supabase db reset --linked` (wipes the linked remote database) and `supabase db push --include-seed` (inserts your fake seed data into real tables).

**Step 4: environment variables per Vercel environment.** In the Vercel project settings, each variable is assigned to Production, Preview, and/or Development:

| Variable | Production | Preview | Development |
|---|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | prod project URL | staging project URL | local URL (`http://127.0.0.1:54321`) |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | prod anon key | staging anon key | local anon key |
| `SUPABASE_SERVICE_ROLE_KEY` | prod service key | staging service key | local service key |
| `STRIPE_SECRET_KEY` | `sk_live_...` | `sk_test_...` | `sk_test_...` |
| `APP_ENV` | `production` | `preview` | `development` |

The service-role key bypasses Row Level Security: it must **never** get a `NEXT_PUBLIC_` prefix and must only be used in server code ([Module 2](02-security-vulnerabilities.md)). You can add values from the CLI too (`vercel env add STRIPE_SECRET_KEY production`), and `vercel env pull` writes your Development values into `.env.local`.

**Step 5: fix the auth URLs before anyone tries to log in.** The classic first-deploy bug: sign-in works locally, but in production the magic link or OAuth login redirects to `http://localhost:3000`. In the Supabase dashboard's Auth URL configuration, set the **Site URL** to your production domain and add your preview URLs to the allowed **Redirect URLs** (Supabase supports wildcard patterns for Vercel preview domains; see its redirect-URL docs). Do this per project: production's list should not include localhost.

**Step 6: deploy and smoke test like a stranger.** Merge to main, let the pipeline deploy, then, in a private browser window: load the home page, sign up as a brand-new user, do the core action, and check the health endpoint. Then sign up as a *second* user and try to open the first user's data by ID. If you can, stop and fix RLS before telling anyone the app exists.

**Step 7: turn on the safety nets** from earlier sections: error tracking with `environment` set ([Module 6](06-observability.md)), billing alerts (section 8), backups with a tested restore ([Module 7](07-data-modeling.md)).

**If you use Neon instead:** Neon's branching gives you a cheap copy of the database per branch, and its Vercel integration can create a database branch for each preview deployment, so every PR gets its own data to break. The production branch stays the production branch, with its own credentials only Production can see.

### 17. Environment mix-ups: the bugs where the code was fine

Some of the most expensive incidents involve no bug in the code at all, just the right code pointed at the wrong place. *(Each story is an illustrative composite of common patterns.)*

**The seed script that ran against production.** A developer copies the production `DATABASE_URL` into `.env.local` "for five minutes" to debug a customer's issue. Later that day, an agent is asked to "reset the dev data so tests are clean." It runs `npm run db:reset`, which drops every table and loads 20 fake users. Into production. Restoring from the nightly backup loses the day's 300 new signups and every order placed since midnight. Then come the support emails.

**The staging app that emailed real customers.** Someone imports a production database dump into staging to reproduce a bug, and staging still has the real email provider key. The nightly "your trial is ending" job runs in staging, and 4,000 real customers get a second, wrong email with a link to a staging URL. Some click it and enter their passwords into a site that isn't the real one.

**The test key in production.** A deploy goes out with `sk_test_...` in Production. Checkout "works": customers see a success page, test-mode charges are created, and no money moves. It's discovered three days later when the founder asks why revenue is zero. Every one of those customers now needs to be contacted and charged properly, and some won't come back.

The fixes are **guards that make the wrong combination impossible**, not reminders to be careful.

**Guard 1: destructive scripts check where they're pointed** (an allowlist, so an unknown host fails safe):

```js
// scripts/lib/dbGuard.js — destructive scripts call this first
const SAFE_HOSTS = ["localhost", "127.0.0.1", "db"]; // local machine and docker-compose
const extra = (process.env.DEV_DB_HOSTS ?? "").split(",").map((h) => h.trim()).filter(Boolean);

export function assertSafeDatabase(action) {
  const host = new URL(process.env.DATABASE_URL ?? "").hostname;
  if (process.env.APP_ENV === "production" || ![...SAFE_HOSTS, ...extra].includes(host)) {
    console.error(`Refusing to ${action}: APP_ENV=${process.env.APP_ENV ?? "unset"}, database host "${host}". Only known dev databases are allowed.`);
    process.exit(1);
  }
}
```

```js
// scripts/reset-db.js
import { assertSafeDatabase } from "./lib/dbGuard.js";

assertSafeDatabase("reset and reseed the database");
// ...only now drop and reseed...
```

```text
$ DATABASE_URL=postgres://...@db.abcdefgh.supabase.co:5432/postgres node scripts/reset-db.js
Refusing to reset and reseed the database: APP_ENV=unset, database host "db.abcdefgh.supabase.co". Only known dev databases are allowed.
```

**Guard 2: the app refuses to boot with the wrong keys for its environment.** Extend your config validation ([Module 5](05-architecture.md)'s single config module):

```ts
// src/lib/config.ts — validate env at startup, including "is this the right environment?"
import { z } from "zod";

const Env = z
  .object({
    APP_ENV: z.enum(["development", "preview", "production"]),
    DATABASE_URL: z.string().url(),
    STRIPE_SECRET_KEY: z.string().startsWith("sk_"),
  })
  .superRefine((env, ctx) => {
    const live = env.STRIPE_SECRET_KEY.startsWith("sk_live_");
    if (env.APP_ENV === "production" && !live) {
      ctx.addIssue({ code: z.ZodIssueCode.custom, path: ["STRIPE_SECRET_KEY"],
        message: "Production must use a live Stripe key (sk_live_...)." });
    }
    if (env.APP_ENV !== "production" && live) {
      ctx.addIssue({ code: z.ZodIssueCode.custom, path: ["STRIPE_SECRET_KEY"],
        message: "Live Stripe key outside production. Use a test key (sk_test_...)." });
    }
  });

export const config = Env.parse(process.env); // throws at boot with a clear message
```

A deploy with a test key in production now fails at startup with a clear message, instead of silently taking fake payments for three days.

**Guard 3: make non-production environments look different.** A colored banner ("STAGING: test data, test payments") on every page outside production costs five lines and prevents a surprising number of "wait, which tab was I in?" mistakes, for you and for anyone you hand a staging link.

**Guard 4: non-production can't reach real people.** Use your email provider's sandbox or test mode (or an allowlist of your own team's addresses) in staging, and test-mode keys for payments and SMS. If staging ever needs production-like data, use a scrubbed copy with fake emails and phone numbers, never a raw dump.

### 18. A zero-downtime migration, worked through

Section 7 gave the rules. Here they are applied to a real change: **adding `org_id` to an existing `tasks` table that already has 200,000 rows**, as part of moving to the multi-tenant model from [Module 7](07-data-modeling.md). The naive migration is one line:

```sql
-- DON'T: on a big, busy table this is how you get an outage
ALTER TABLE tasks ADD COLUMN org_id uuid NOT NULL REFERENCES orgs(id);
```

It fails outright on a table with existing rows (they'd all be NULL), and the AI's usual "fix" (add a made-up default, or backfill and then add `NOT NULL` in the same migration) holds strong locks while scanning the whole table. While it runs, every request that touches `tasks` waits. On a large table that can be minutes of "the app is down" at peak hour, and if the deploy times out halfway, you get to find out what state you're in.

The safe version takes three migrations and two deploys, and the app stays up throughout:

**Migration A (expand): add a nullable column.** Adding a nullable column without a default doesn't rewrite the table, so it's instant.

```sql
-- Migration A (expand): add the column as nullable. Instant: no table rewrite.
SET lock_timeout = '5s';   -- give up instead of queueing every other query behind us
ALTER TABLE tasks ADD COLUMN org_id uuid;
```

`lock_timeout` matters more than it looks. `ALTER TABLE` needs a brief exclusive lock; if a long-running query is holding the table, the `ALTER` waits, and every new query lines up behind *it*. With a timeout, the migration fails fast and you retry, instead of freezing the app.

**Deploy 1: new code writes `org_id`** on every new task (copied from the task's project). Old rows still have NULL; the code tolerates that for now.

**Backfill in batches** (a script or a scheduled job, not one giant transaction):

```sql
-- Backfill in batches; run repeatedly until it reports UPDATE 0
UPDATE tasks t
SET org_id = p.org_id
FROM projects p
WHERE p.id = t.project_id
  AND t.id IN (SELECT id FROM tasks WHERE org_id IS NULL LIMIT 5000);
```

Each batch locks only 5,000 rows briefly. For our 200,000 rows that's 40 batches, then one that reports `UPDATE 0`.

**Migration B (enforce): make it required without a long lock.**

```sql
-- Migration B (enforce), after the backfill reports UPDATE 0:
SET lock_timeout = '5s';
ALTER TABLE tasks ADD CONSTRAINT tasks_org_id_fkey
  FOREIGN KEY (org_id) REFERENCES orgs(id) NOT VALID;          -- instant; checks new writes only
ALTER TABLE tasks VALIDATE CONSTRAINT tasks_org_id_fkey;       -- checks old rows without blocking writes
ALTER TABLE tasks ADD CONSTRAINT tasks_org_id_not_null
  CHECK (org_id IS NOT NULL) NOT VALID;                        -- instant; new writes must have org_id
ALTER TABLE tasks VALIDATE CONSTRAINT tasks_org_id_not_null;   -- scans old rows without blocking writes
ALTER TABLE tasks ALTER COLUMN org_id SET NOT NULL;            -- uses the validated CHECK: no second scan
ALTER TABLE tasks DROP CONSTRAINT tasks_org_id_not_null;       -- redundant now
```

The trick (documented in [Postgres's `ALTER TABLE` reference](https://www.postgresql.org/docs/current/sql-altertable.html)): a `NOT VALID` constraint is added instantly and applies to new writes; `VALIDATE` checks existing rows with a lighter lock that doesn't block normal reads and writes; and when `SET NOT NULL` finds a validated `CHECK (org_id IS NOT NULL)`, Postgres skips its own full-table scan. With debug logging on, Postgres tells you so: *"existing constraints on column "tasks.org_id" are sufficient to prove that it does not contain nulls."*

And if the backfill missed rows, `VALIDATE` fails with `check constraint "tasks_org_id_not_null" ... is violated by some row` and changes nothing. That's the safety net working: run the backfill again until it reports `UPDATE 0`, then retry.

**Migration C: index it without blocking writes.**

```sql
-- Migration C: index without blocking writes (cannot run inside a transaction block)
CREATE INDEX CONCURRENTLY idx_tasks_org ON tasks (org_id);
```

`CONCURRENTLY` builds the index while the table stays writable. It can't run inside a transaction, and many migration tools wrap each migration in one, so check your tool's docs for how to mark a migration as non-transactional.

**Deploy 2: new code relies on `org_id`** (filters by it, requires it).

The timeline, so you can see that every moment is safe:

| Moment | Database | Running code | Safe? |
|---|---|---|---|
| Before | no `org_id` | v1 (ignores it) | yes |
| After migration A | nullable `org_id` | v1 | yes: v1 doesn't know the column exists |
| During deploy 1 | nullable `org_id` | v1 and v2 side by side | yes: v1 writes NULL, v2 writes values, both allowed |
| During backfill | filling in | v2 | yes: small batches, short locks |
| After migration B | `org_id NOT NULL` | v2 (always writes it) | yes: v1 is fully gone, so nobody writes NULL |
| After deploy 2 | indexed, required | v3 (uses it) | yes |

The one thing this plan needs is patience: migration B must wait until no v1 instance is still running. That's why it's a *separate* migration, not the end of migration A.

### 19. Runaway costs, with the math

The Vibe Fail at the top of this module is a $4,000 weekend. Here's how fast that number arrives, so the guardrails stop feeling optional. *(Illustrative: the prices are hypothetical but in the range of popular hosted models; plug in your provider's current pricing.)*

Assume a model priced at **$2.50 per million input tokens** and **$10 per million output tokens**, and a "summarize this PDF" endpoint that sends about 8,000 tokens in and gets 800 back:

| | Calculation | Cost |
|---|---|---|
| One request | 8,000 × $2.50/M + 800 × $10/M = $0.020 + $0.008 | **$0.028** |
| An abuser's script: 1,000 requests/hour | 1,000 × $0.028 | $28/hour |
| Plus a retry loop: each timeout retried, ~3 calls per request | 3,000 × $0.028 | **$84/hour** |
| Friday 6 p.m. to Sunday 6 p.m. (48 hours) | $84 × 48 | **$4,032** |

Nobody did anything clever. Three cents a call is invisible in testing, and $84 an hour is invisible if nobody is watching. Now the same weekend with the guardrails from this module:

| Guardrail | Effect on that weekend |
|---|---|
| Authentication on the endpoint (section 8) | The anonymous script gets `401`s. Cost: ~$0. |
| Per-user rate limit, e.g. 20 summaries/hour (section 9) | Even a real account that abuses it is capped at ~$0.56/hour. |
| Bounded retries (max 3 attempts, only for retryable errors, with backoff) | Removes the 3× multiplier during provider trouble. |
| Input cap, e.g. 4,000 tokens per request | Halves the input cost of every call. |
| Hard monthly spending cap of $50 on the production key | **The absolute worst case is $50**, whatever else goes wrong. |
| Billing alert at $10 | You hear about it Friday night, not Monday morning. |

The spending cap is the one that turns "unbounded" into "bounded." Every other guardrail reduces the odds; the cap limits the damage when the odds lose. Spend 15 minutes setting the cap and the alerts on every usage-billed service today (section 13 has the full checklist).

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

**Next up:** [Module 9: Tech Debt, Refactoring & Knowing When to Stop Vibing](09-tech-debt.md) · [Back to course outline](../course-outline.md)
