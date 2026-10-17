# Module 5: Architecture for Prompt Engineers

> The AI will happily build you a house. It just might put the bathroom in the kitchen,
> run the plumbing through the bedroom, and add a second front door every time you ask for a window.

Architecture sounds like something for people with whiteboards and job titles that include "Principal." It isn't. Architecture is just **deciding where things go and how they're allowed to talk to each other**, and then sticking to it.

The AI won't make those decisions for you. Or rather, it will make them, differently, every single session. Monday's prompt puts the database query in the React component. Tuesday's prompt puts it in an API route. Wednesday's creates a `services/` folder. Thursday's creates `lib/services/` *and* `utils/data/`. By Friday you have four ways of doing everything and a 3,000-line component nobody (including the AI) can safely change.

This module gives you just enough architecture to keep that from happening, and shows you how to write it down so the AI follows it too.

---

## Learning Objectives

By the end of this module you will be able to:

1. Explain separation of concerns and why mixing everything together makes every change dangerous.
2. Describe the four basic layers of a typical app (UI, API, business logic, data) and what belongs in each.
3. Decide where logic belongs: server vs. client, and why "the client is a suggestion" is a law of nature.
4. Recognize tight coupling and the "3,000-line component," and know how to break it up.
5. Keep a single source of truth for data, types, and business rules.
6. Set up sane file and folder conventions.
7. Write an architecture doc or rules file (`AGENTS.md`, `.cursor/rules`, `.cursorrules`, `CLAUDE.md`) so every AI session builds things the same way.

---

## Vibe Fail: Dashboard.tsx, Line 2,847

*(Illustrative story. If it sounds familiar, that's because it happens constantly.)*

Priya built a project-management app over a few weekends. The AI was incredible: every feature request turned into working code in seconds. All of it went into `Dashboard.tsx`, because that's where the dashboard was.

By month two, `Dashboard.tsx` was 2,900 lines. It contained:

- 31 `useState` hooks.
- Direct database calls through the Supabase client, with the filtering logic in the browser.
- Three different functions for formatting dates, one of which was off by a day in some time zones.
- The billing logic that decided whether a user was on a paid plan, computed in the browser from a field the user could edit.
- An inline email template.
- A `useEffect` that refetched everything whenever anything changed, which sometimes looped forever.

Then a customer asked for a tiny change: show overdue tasks in red. The AI made the change. It also broke the billing check (they shared a variable), which unlocked paid features for every free user, and the infinite loop came back, which burned through the database's free tier in an afternoon.

Every new prompt now took longer, broke more things, and the AI's context window couldn't even hold the whole file anymore. Priya didn't have a code problem. She had an **architecture** problem: there was no structure, so every change touched everything.

---

## Core Concepts

### 1. Separation of concerns

**Separation of concerns** means each piece of code has one job, and it doesn't do other pieces' jobs.

A restaurant works because the waiter doesn't cook, the cook doesn't handle the cash register, and the accountant doesn't seat guests. Each can be replaced, trained, or improved without retraining everyone else. If one person does everything, it works great for a food truck and falls apart at 200 covers a night.

In code, "concerns" are things like:

- Showing things on screen (UI)
- Receiving and validating requests (API)
- Deciding what should happen (business rules)
- Storing and retrieving data (persistence)
- Talking to outside services (email, payments, AI APIs)

When these are mixed, a change to one breaks another. When they're separated, you can change the button color without touching billing, and you can test billing without rendering a button.

### 2. The four layers

Most web apps, regardless of framework, can be thought of in four layers:

```text
┌─────────────────────────────────────┐
│  UI (client)                        │  What the user sees and clicks.
│  components, pages, forms           │  Displays data. Collects input. No secrets. No rules that matter.
├─────────────────────────────────────┤
│  API (server boundary)              │  The front door. Authenticates, validates input,
│  routes, handlers, server actions   │  calls business logic, returns responses.
├─────────────────────────────────────┤
│  Business logic (server)            │  The rules of YOUR business. Pricing, permissions,
│  services, domain functions         │  workflows. Plain functions, easy to test.
├─────────────────────────────────────┤
│  Data access (server)               │  Talks to the database and external services.
│  repositories, db client, adapters  │  The only layer that knows SQL or the ORM.
└─────────────────────────────────────┘
```

**The rules:**

- Each layer only talks to the layer directly below it. The UI doesn't run SQL. The data layer doesn't know about HTTP.
- The API layer is thin: authenticate, validate, call a service, return. If your route handler is 300 lines, business logic is leaking into it.
- Business logic is plain functions that take data and return data. No `req`/`res`, no React, no database client imported directly if you can help it. That makes it trivially testable (see [Module 4](04-testing.md)).

**Bad: everything in the route handler**

```ts
// app/api/orders/route.ts
export async function POST(req: Request) {
  const body = await req.json();
  const { data: user } = await supabase.auth.getUser();
  const items = await db.query("SELECT * FROM products WHERE id = ANY($1)", [body.ids]);
  let total = 0;
  for (const i of items.rows) total += i.price;
  if (body.coupon === "SAVE10") total = total * 0.9;
  if (user.email.endsWith("@mycompany.com")) total = 0; // staff discount lol
  await db.query("INSERT INTO orders (user_id, total) VALUES ($1, $2)", [user.id, total]);
  await fetch("https://api.sendgrid.com/v3/mail/send", { /* ...inline email... */ });
  return Response.json({ ok: true, total });
}
```

**Good: thin route, real layers**

```ts
// app/api/orders/route.ts  (API layer)
export async function POST(req: Request) {
  const user = await requireUser(req);                 // auth
  const input = CreateOrderSchema.parse(await req.json()); // validation
  const order = await orderService.createOrder(user, input); // business logic
  return Response.json(order, { status: 201 });
}

// services/orderService.ts  (business logic)
export async function createOrder(user: User, input: CreateOrderInput) {
  const products = await productRepo.findByIds(input.productIds);
  const totalCents = pricing.calculateTotal(products, input.couponCode, user);
  const order = await orderRepo.insert({ userId: user.id, totalCents });
  await notifications.orderConfirmed(user, order);
  return order;
}

// repos/orderRepo.ts  (data access)
export function insert(order: NewOrder) {
  return db.order.create({ data: order });
}
```

More files, yes. But now pricing is testable on its own, the staff discount is a visible business rule instead of a joke comment, and swapping SendGrid for another provider touches one file.

### 3. Where logic belongs: server vs. client

This one is not a style preference. It's physics.

**Anything running in the browser is under the user's control.** They can read it, modify it, skip it, or replay requests without it. The browser is a *suggestion box*, not an enforcement mechanism.

| Belongs on the client | Belongs on the server |
|---|---|
| Layout, styling, animations | Authentication and authorization |
| Form validation **for UX** (instant feedback) | Form validation **for real** (the one that counts) |
| Optimistic UI updates | Prices, totals, discounts, credits |
| Sorting/filtering data the user is already allowed to see | Deciding what data the user is allowed to see |
| Calling your own API | Calling third-party APIs that need secret keys |
| Feature flags for display | Plan limits, quotas, paywalls |

If a rule matters (money, access, limits), it lives on the server. Validate on the client for convenience, then validate again on the server because the client lies. The security consequences of getting this wrong are in [Module 2](02-security-vulnerabilities.md); the architectural point is simply that **the server is the source of truth for every decision that matters.**

Frameworks blur this line (Next.js server components, server actions, Supabase clients that run in the browser). Make sure you know, for each file, **where it runs**. If you're not sure, ask the AI, and then verify by checking for `"use client"`, `"use server"`, or which bundle it ends up in.

### 4. Coupling: when touching one thing breaks another

**Coupling** is how much one piece of code depends on the internals of another. Tight coupling means you can't change A without changing B, C, and D.

Signs of tight coupling:

- Changing a database column name requires edits in 25 files, including React components.
- Two features share a variable, and changing one breaks the other (Priya's billing bug).
- Components reach into each other's state.
- Everything imports from everything (circular imports, or a `utils.ts` that every file depends on).
- You can't test a function without spinning up the whole app.

How to loosen it:

- **Talk through interfaces, not internals.** The UI calls `getProjects()`; it doesn't know whether that's SQL, an API, or a cache.
- **Pass data in, get data out.** Functions that take arguments and return values are less coupled than functions that read and write global state.
- **Map database rows to app types at the data layer**, so a column rename stays contained.
- **Keep features in their own folders**, so "billing" and "tasks" don't share internals.

### 5. Single source of truth

Every important fact in your app should be defined in exactly **one** place.

- **Data:** one table holds the user's plan. Not a column in `users`, a field in `profiles`, *and* a value cached in local storage that disagree with each other.
- **Types:** one definition of `Order`, ideally generated from or validated against your schema (e.g., Zod schemas, Prisma/Drizzle types, Pydantic models), and imported everywhere.
- **Business rules:** one `calculateTotal()`. One `canEditProject(user, project)`. Imported, never re-implemented.
- **Config:** one place that reads environment variables and exports typed config.

```ts
// BAD: three places decide what "pro" means
if (user.plan === "pro") ...                     // Header.tsx
if (profile.tier >= 2) ...                       // Settings.tsx
if (localStorage.getItem("isPro") === "true") ...// Export.tsx

// GOOD: one function, server-side data, imported everywhere
// lib/permissions.ts
export function hasProFeatures(account: Account): boolean {
  return account.plan === Plan.PRO && account.subscriptionStatus === "active";
}
```

AI tools are the number one source of duplicate truth in modern codebases, because each session reinvents instead of reusing. The fix is to tell the AI where the truth lives (see the rules file section).

### 6. The 3,000-line component

Every vibe-coded app eventually grows one. Here's how to recognize and defuse it.

**Symptoms:** a single file over ~300-500 lines; more than ~8 pieces of state; data fetching, business rules, and markup all together; the AI starts "losing" parts of it or breaking unrelated features when editing it.

**How to break it up (safely):**

1. **Write tests first** for the behavior you care about (Module 4). Even a few E2E tests are a safety net.
2. **Extract pure logic** (calculations, formatting, permission checks) into plain functions in `lib/` or `services/`. Easiest, safest win.
3. **Move data fetching** out of the component into the API/data layer or a dedicated hook (`useProjects()`).
4. **Split the UI** into smaller components by visual section (`ProjectList`, `ProjectFilters`, `BillingBanner`).
5. **Do one extraction per commit**, run the app and tests between each. Don't ask the AI to "refactor the whole file" in one go; that's how you get a 3,000-line diff nobody can review (Module 3).

Module 9 goes deeper on refactoring without breaking things.

### 7. File and folder conventions

The specific convention matters less than **having one and following it**. Two common shapes:

**Layer-based** (fine for small apps):

```text
src/
  components/     # UI pieces
  app/ or pages/  # routes/pages (framework-defined)
  api/            # route handlers (if not inside app/)
  services/       # business logic
  repos/ or db/   # data access
  lib/            # shared utilities, config, types
tests/
```

**Feature-based** (scales better as the app grows):

```text
src/
  features/
    billing/
      components/  service.ts  repo.ts  billing.test.ts
    projects/
      components/  service.ts  repo.ts  projects.test.ts
  shared/
    db.ts  config.ts  auth.ts  ui/
```

Conventions worth writing down:

- File naming (`kebab-case.ts` vs `PascalCase.tsx` for components).
- Where tests live (next to the code, or in `tests/`).
- One component per file.
- No new top-level folders without a reason.
- Max file size before you split (a soft limit like 300 lines is a good prompt for the AI).

### 8. Give the AI an architecture doc (rules files)

Here's the vibe-coder superpower most people skip: **you can write your architecture down once, and the AI will read it every session.**

Most AI coding tools support a project-level instruction file:

- `AGENTS.md`: an open, tool-agnostic convention read by many coding agents ([agents.md](https://agents.md/)).
- Cursor: project rules in `.cursor/rules/` (older projects use a single `.cursorrules` file).
- Claude Code: `CLAUDE.md`.
- Others: GitHub Copilot uses `.github/copilot-instructions.md`; many tools also read a `README` or `CONTRIBUTING.md`.

Check your tool's current docs for the exact file name; the idea is the same everywhere. A good rules file is **short, specific, and enforceable**. Not "write clean code," but "all database access goes through `src/repos/`."

**Example `AGENTS.md`:**

```markdown
# Project rules for AI agents

## Stack
- Next.js (App Router), TypeScript strict, Postgres via Drizzle, Zod for validation.
- Auth: Supabase Auth. Payments: Stripe.

## Architecture
- Layers: UI (src/app, src/components) → API (src/app/api) → services (src/services) → repos (src/repos).
- UI never imports from src/repos or the db client directly.
- Route handlers are thin: requireUser() → Zod parse → call one service function → return.
- All business rules (pricing, permissions, plan limits) live in src/services and run on the server only.
- Money is stored and computed in integer cents. Never floats.

## Single sources of truth
- Types: src/lib/types.ts and Zod schemas in src/lib/schemas.ts. Reuse; don't redefine.
- Permissions: src/services/permissions.ts. Never re-implement access checks elsewhere.
- Config/env: src/lib/config.ts is the only file that reads process.env.

## Conventions
- Components: PascalCase.tsx, one per file, < 300 lines. Split if larger.
- Tests next to code: *.test.ts. Every service function gets unit tests.
- No new dependencies without asking. No new top-level folders.

## Never
- Never put secrets in client code or commit .env files.
- Never disable lint rules or TypeScript checks to make errors go away.
- Never modify migrations that have already been applied; create a new one.
- Never delete or weaken tests to make them pass.
```

Keep it updated. When the AI does something you don't want twice, that's a new rule. When you make an architecture decision, write a one-line note (some teams keep a `docs/decisions/` folder of short "Architecture Decision Records" explaining what was decided and why).

---

## What Good Looks Like

- You can draw your app's layers on a napkin and point to the folder for each.
- Route handlers are short. Business logic lives in plain, tested functions.
- Every rule that matters (money, access, limits) runs on the server.
- Each important fact (plan, price, permission, type) is defined once.
- No file is so big that the AI can't hold it in context or you can't skim it.
- New features follow the same pattern as old ones, because the rules file told the AI how.
- Changing a database column touches the data layer and maybe one type, not 25 components.

**Architecture checklist for any new feature:**

- [ ] Which layer(s) does this touch, and does each piece live in the right one?
- [ ] Are rules and permissions enforced on the server?
- [ ] Does it reuse existing types, services, and helpers instead of creating new ones?
- [ ] Is the route handler thin?
- [ ] Is any file now over the size limit?
- [ ] Did it add a new pattern, folder, or library? If so, was that a conscious decision?
- [ ] Does the rules file need a new line because of this?

---

## Prompt Templates

**1. Plan before code**

```text
Before writing code, propose where this feature should live: [feature].
Follow the architecture in AGENTS.md. For each new or changed file, give the
path, its layer (UI/API/service/repo), and its responsibility. Identify
existing functions, types, and components you will reuse. Wait for my
approval before implementing.
```

**2. Generate a rules file from the existing code**

```text
Analyze this codebase and draft an AGENTS.md describing: the stack, the
layers and which folders belong to each, where business logic, data access,
validation, and config live, naming conventions, and any inconsistencies you
found (multiple patterns for the same thing). Recommend ONE pattern for each
inconsistency. Keep it under 80 lines.
```

**3. Architecture review of a diff**

```text
Review this diff against our architecture rules (AGENTS.md). Flag: business
logic in UI or route handlers, database access outside the repo layer,
client-side enforcement of anything security- or money-related, duplicated
types or helpers that already exist, and files over 300 lines. Suggest the
minimal moves to fix each.
```

**4. Defuse the giant component (one step at a time)**

```text
[File] is too large. Propose a step-by-step extraction plan, smallest and
safest steps first (pure functions, then data fetching, then UI splits).
Each step should be a separate, independently testable commit that doesn't
change behavior. Do ONLY step 1 now.
```

**5. Find duplicate truth**

```text
Search the codebase for duplicated definitions of: types/interfaces for
[entity], formatting helpers (dates, money), permission checks, and
environment variable reads. List each duplicate with file paths and propose
which one should be the single source of truth.
```

**6. Where does this run?**

```text
For each file in [folder], tell me whether it runs on the server, the client,
or both, and how you know (directives, imports, framework rules). Flag any
client-side file that references secrets, does permission checks, or
computes prices.
```

---

## Exercise

**Part A: Draw your architecture (15 minutes)**

On paper, draw the four layers. For your current project, write which folders and files belong to each. Mark anything that doesn't clearly belong anywhere, or belongs to several layers at once. Those are your hotspots.

**Part B: Find your biggest file (10 minutes)**

Run one of these from your project root:

```bash
# macOS/Linux: largest source files by line count
find src -name "*.ts*" -o -name "*.py" | xargs wc -l | sort -n | tail -10
```

For the largest file, list the separate concerns it contains (UI, fetching, rules, formatting, etc.). Use Prompt Template 4 to get an extraction plan. Do step 1 only, verify the app still works, and commit.

**Part C: Write your rules file (30 minutes)**

Use Prompt Template 2 to draft an `AGENTS.md` (or your tool's equivalent). Edit it by hand: delete anything vague, add any rule you wish the AI had followed last week. Then start a fresh AI session and ask for a small feature. Did it follow the rules? If not, make the rule more specific.

**Part D: Server vs. client audit (15 minutes)**

List every rule in your app involving money, access, or limits. For each, find where it's enforced. If any are enforced only in the browser, write them down and fix them (with [Module 2](02-security-vulnerabilities.md) open in another tab).

---

## Key Takeaways

- Architecture is deciding where things go and how they talk, then sticking to it. If you don't decide, the AI decides differently every session.
- Separate concerns into layers: UI → API → business logic → data. Each layer talks only to the one below.
- Keep route handlers thin. Put business rules in plain, testable functions.
- The client is a suggestion box. Anything that matters (money, access, limits) is enforced on the server.
- Loose coupling means changing one thing doesn't break five others. Pass data in, get data out, talk through interfaces.
- Every important fact has one source of truth. AI sessions love to create a second and third one; don't let them.
- Break up giant components gradually: tests first, extract pure logic, then data fetching, then UI, one commit at a time.
- Write your architecture down in a rules file (`AGENTS.md`, `.cursor/rules`, `CLAUDE.md`). It's the cheapest way to make every future AI session consistent.

---

**Next up:** [Module 6: Error Handling & Observability](06-observability.md)
