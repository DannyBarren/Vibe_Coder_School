# Module 3: Reading the Code You Accepted

> "I don't need to read it. It works."
> Famous last words, usually said about forty minutes before somebody has to read it at 2 a.m. while a customer waits.

You don't have to become a full-time software engineer to ship with AI. You do have to become someone who can **read**. Not write from scratch. Read. You don't need to be a mechanic to drive a car, but you should be able to tell when the dashboard is on fire.

This module teaches you how to review an AI diff in minutes, how to find your way around a codebase you didn't write (yes, that includes the one you "wrote" last Tuesday), how to follow one request from the button click to the database and back, and which red flags in AI-generated code should make you stop and look closer.

---

## Learning Objectives

By the end of this module you will be able to:

1. Review an AI-generated diff using a repeatable, ten-minute process instead of a vibe check.
2. Orient yourself in an unfamiliar codebase (entry points, routes, data layer, config) without reading every file.
3. Trace a single request end to end through UI, API, business logic, and database.
4. Spot the most common red flags in AI code: dead code, duplicated logic, swallowed errors, magic values, hallucinated APIs, and over-engineering.
5. Use the AI as a tutor to explain code, without letting it grade its own homework.
6. Use git habits (small commits, reviewing diffs before accepting, easy rollback) so a bad suggestion costs you minutes instead of days.

---

## Vibe Fail: The Discount That Applied Itself Twice

*(Illustrative story. The pattern is real; the company is made up.)*

Maya runs a small online store for hand-printed posters. She asks her AI assistant: "Add a 10% discount code, SAVE10, at checkout." Thirty seconds later there's a working coupon field. She types SAVE10, the total drops by 10%, she clicks Accept All, and she deploys.

Three weeks later her accountant asks why margins fell off a cliff. Here's what was in the diff she didn't read:

- The discount was calculated **in the browser** and the final price was sent to the server, which trusted it. (Anyone with dev tools could set their own price. See [Module 2](02-security-vulnerabilities.md) for why that's a security problem.)
- The AI had **duplicated** the price calculation in two places: the cart page and the checkout page. On checkout, the code applied the discount to a total that already had the discount. Ten percent became nineteen percent.
- A `try/catch` around the payment call caught every error and returned `{ success: true }`. Some orders "succeeded" without being charged at all.
- There was an unused `applyLoyaltyPoints()` function the AI had invented "for future use." It called an API method that doesn't exist in her payment library. It never ran, so nobody noticed. Yet.

None of this needed a senior engineer to catch. It needed someone to spend ten minutes reading the diff with a checklist. The AI's own summary even said "I've also added price calculation to the checkout page for consistency." The clue was right there in plain English.

---

## Core Concepts

### 1. Reading is a different skill from writing

The good news: reading code is much easier than writing it. You don't need to know how to build a regular expression to notice that one is being used to validate email addresses and could probably be replaced with a library. You don't need to know the payment API by heart to ask "does this method actually exist?"

When you review, you're answering four questions:

1. **What changed?** (Which files, which functions, how much.)
2. **Why did it change?** (Does it match what I asked for, and nothing more?)
3. **What could go wrong?** (Errors, edge cases, bad input, missing permissions.)
4. **How would I know if it broke?** (Tests, logs, something visible.)

If you can't answer all four, you're not done reviewing. You're done *hoping*.

### 2. How to read a diff

A diff is the before-and-after of a change. Lines starting with `-` were removed; lines with `+` were added. Every AI coding tool shows you one. Most vibe coders scroll past it like terms and conditions.

**The ten-minute diff review:**

1. **Read the AI's summary first.** It often tells you about side quests ("I also refactored...", "for consistency I added...", "I updated the config..."). Every side quest is something you didn't ask for. Be suspicious.
2. **Look at the file list before the code.** Did you ask for a button color change and get edits to `auth.ts`, `package.json`, and a database migration? Stop right there.
3. **Check the size.** A one-line request that produced 400 changed lines is a smell. Big diffs hide bugs because nobody reads them carefully, including you.
4. **Read deletions as carefully as additions.** AI agents love to "simplify" by removing a check that looked redundant. That check was often the authorization guard or the input validation.
5. **Follow new inputs.** Anywhere data comes in (form fields, query strings, request bodies, files, webhook payloads), ask: is it validated? On the server?
6. **Follow new outputs.** Anywhere data goes out (database writes, API calls, emails, logs), ask: is it the right data, going to the right place, for the right user?
7. **Look at error paths.** What happens when the network call fails, the record doesn't exist, or the user isn't logged in?
8. **Check new dependencies.** Any new package in `package.json`, `requirements.txt`, or equivalent: does it exist, is it the real one, and do you need it?
9. **Run it.** Not just the happy path. Try the empty input, the wrong user, the double click.
10. **Decide: accept, ask for changes, or reject.** Rejecting is allowed. It's often the fastest option.

#### Worked example: annotating a realistic AI diff

*(Illustrative. The diff is invented, but every problem in it is one you'll see in real AI output.)*

You asked: **"Let users update their display name on the profile page."** The AI replied: *"Done! I've added the update endpoint, simplified the auth handling, and added slug generation for nicer profile URLs."*

Two side quests in one sentence ("simplified the auth handling", "added slug generation"). Already suspicious. Here's the server part of the diff:

```diff
 // app/api/profile/route.ts
-import { requireUser } from "@/lib/auth";
-import { UpdateProfileSchema } from "@/lib/schemas";
+import { db } from "@/lib/db";
+import slugify from "slugify-pro";
 
 export async function PATCH(req: Request) {
-  const user = await requireUser(req);
-  const input = UpdateProfileSchema.parse(await req.json());
-  const updated = await profileService.updateDisplayName(user.id, input.displayName);
-  return Response.json(updated);
+  const body = await req.json();
+  const userId = body.userId;
+  const slug = slugify(body.displayName || "user-" + Math.random());
+
+  try {
+    await db.user.update({
+      where: { id: userId },
+      data: { ...body, slug },
+    });
+  } catch (e) {
+    console.log(e);
+  }
+
+  return Response.json({ success: true });
 }
```

Reading it line by line, with the questions from the checklist:

| Line | What it says | What you should think |
|---|---|---|
| `-import { requireUser }` | Auth import **deleted** | Deletions first. Who checks that the caller is logged in now? (Nobody.) |
| `-import { UpdateProfileSchema }` | Validation schema **deleted** | Input is no longer validated. |
| `+import slugify from "slugify-pro"` | New package | Did I ask for this? Does `slugify-pro` exist, and is it the real one? (The well-known package is `slugify`. A plausible-sounding variant is exactly what a hallucinated or typosquatted package looks like. Check npm before installing anything.) |
| `const userId = body.userId` | The user ID comes **from the request body** | Anyone can send someone else's ID and edit their profile. This is the classic broken-authorization bug from [Module 2](02-security-vulnerabilities.md). |
| `Math.random()` in the slug | Random fallback | Magic behavior. Two people with empty names get unpredictable slugs, and nothing guarantees they're unique. |
| `data: { ...body, slug }` | The whole request body is written to the user row | Mass assignment: a caller can include `role: "admin"`, `plan: "pro"`, or `email` and the database will happily save it. |
| `catch (e) { console.log(e) }` | Swallowed error | Failure is invisible, and... |
| `return { success: true }` | Unconditional success | ...the user is told it worked even when it didn't. |
| `-profileService.updateDisplayName(...)` | Existing service call removed | The project already had a function for this. The AI bypassed the service layer ([Module 5](05-architecture.md)) instead of reusing it. |

Nine findings in a 20-line diff, and the feature "works" in the preview. That's the whole problem with judging by the preview.

What you actually wanted is smaller than what the AI produced:

```ts
// app/api/profile/route.ts — the reviewed version
import { requireUser } from "@/lib/auth";
import { UpdateProfileSchema } from "@/lib/schemas"; // displayName: string, 1–50 chars, trimmed

export async function PATCH(req: Request) {
  const user = await requireUser(req);                         // identity comes from the session
  const input = UpdateProfileSchema.parse(await req.json());   // only displayName is accepted
  const updated = await profileService.updateDisplayName(user.id, input.displayName);
  return Response.json(updated);                               // errors propagate to the global handler
}
```

Slugs, if you want them, are a separate request, a separate diff, and a separate review.

**How to reply to the AI** when a diff looks like this. Don't say "fix the bugs." Be specific:

```text
Reject this change. Restore requireUser and UpdateProfileSchema. The user ID
must come from the session, never the request body. Only displayName may be
updated; do not spread the request body into the database. Do not add
slugify-pro or any new dependency. Reuse profileService.updateDisplayName.
Do not catch errors in the route; let the global handler deal with them.
Show me the new diff.
```

### 3. Orienting in an unfamiliar codebase

Whether you inherited a project or just let an agent run for three hours, you'll eventually face a repo you don't understand. Don't read it top to bottom. Read it like a map.

**The orientation tour (about 30 minutes):**

| Look at | What it tells you |
|---|---|
| `README`, `AGENTS.md`, `.cursorrules`, docs folder | What the project claims to be and how to run it |
| `package.json` / `pyproject.toml` / `requirements.txt` | Framework, libraries, scripts (`dev`, `test`, `build`, `migrate`) |
| Entry points (`main.py`, `app.ts`, `server.js`, `app/` or `pages/` folder) | Where execution starts |
| Routes / API folder | Every way the outside world can talk to the app |
| Database schema / migrations / models | What data exists and how it's related |
| Config and environment files (`.env.example`, `config/`) | What external services it depends on |
| Tests folder | What someone thought was important enough to check (or nothing) |
| `git log --oneline \| head -30` | Recent history, and whether commits are sane or "fix stuff" x40 |

Write down a one-paragraph summary when you're done: "This is a Next.js app with a Postgres database via Prisma. Users log in with Supabase auth. There are 14 API routes. Payments go through Stripe in `lib/payments.ts`. There are 3 tests, all for a utility function." That paragraph is worth more than any amount of scrolling.

### 4. Tracing a request end to end

The single most useful reading skill is following one action all the way through the system. Pick something concrete: "User clicks **Delete project**."

```text
Button click (components/ProjectCard.tsx)
  → calls deleteProject(id) (lib/api-client.ts)
    → HTTP DELETE /api/projects/:id (app/api/projects/[id]/route.ts)
      → checks session? (does it?)
      → checks this user owns the project? (does it?!)
      → calls projectService.delete(id) (services/projects.ts)
        → DB: DELETE FROM projects WHERE id = $1 (and the tasks? the files?)
      → returns 204
  → UI removes card, shows toast
```

At each hop, ask:

- **Who is allowed to do this, and where is that checked?** (On the server. Always the server.)
- **What data is passed, and is it trusted?** (The `id` came from the browser. Treat it as hostile.)
- **What happens if this step fails?** (Does the UI still show "Deleted!"?)
- **What else is affected?** (Child records, caches, files in storage, emails.)

You can do this with your editor's "Go to Definition" (F12 in VS Code and Cursor) and "Find All References." You can also ask the AI to do the trace for you, then verify each hop yourself. More on that below.

### 5. Red flags in AI-generated code

AI code has a recognizable set of bad habits. Once you know them, you'll see them everywhere.

#### Red flag 1: Dead code

Functions that are never called, imports that are never used, feature flags for features that don't exist, "helper" files from an approach the agent abandoned three prompts ago.

Why it matters: dead code isn't harmless. It confuses the next AI session (which will happily "fix" or reuse it), it hides real logic, and occasionally it wakes up. The most expensive example on record: in 2012, Knight Capital deployed new trading code to seven of eight servers. The eighth still had old, unused "Power Peg" code that a repurposed flag reactivated. In about 45 minutes it sent millions of orders and the firm lost more than $460 million ([SEC order](https://www.sec.gov/files/litigation/admin/2013/34-70694.pdf); [SEC press release](https://www.sec.gov/newsroom/press-releases/2013-222)). Your poster shop won't lose $460 million. But the principle scales down nicely: **code that isn't supposed to run should not exist.**

#### Red flag 2: Duplicated logic

The AI doesn't remember that you already have a `formatPrice()` function, so it writes another one. Then another. Each slightly different.

```ts
// cart.tsx
const total = items.reduce((s, i) => s + i.price * i.qty, 0) * 0.9;

// checkout.tsx  (written by a different prompt, a day later)
const subtotal = items.reduce((s, i) => s + i.price * i.qty, 0);
const total = applyDiscount(subtotal, code) * 0.9; // discount applied twice
```

This isn't just a vibe-coder problem; it shows up in the data. GitClear's analysis of 211 million changed lines found that 2024 was the first year copy/pasted lines outnumbered "moved" (refactored) lines, with an eightfold rise in duplicated blocks of five or more lines ([GitClear 2025 report](https://www.gitclear.com/ai_assistant_code_quality_2025_research/)). Duplication means every bug fix has to be made in N places, and you'll remember N-1 of them.

**What good looks like:** one function, one place, imported everywhere.

```ts
// lib/pricing.ts — the single source of truth for money math
export function calculateTotal(items: CartItem[], discount?: Discount): number {
  const subtotalCents = items.reduce((s, i) => s + i.priceCents * i.qty, 0);
  return discount ? applyDiscount(subtotalCents, discount) : subtotalCents;
}
```

#### Red flag 3: Swallowed errors

```js
// BAD: the error vanishes, the caller thinks everything worked
try {
  await chargeCard(order);
} catch (e) {
  console.log(e);
}
return { success: true };
```

```js
// GOOD: fail loudly, give the caller the truth, keep details for logs
try {
  await chargeCard(order);
} catch (err) {
  logger.error({ err, orderId: order.id }, "payment failed");
  throw new PaymentError("We couldn't process your payment. You have not been charged.");
}
```

An empty or log-only `catch` is the AI's favorite way to make an error "go away." It doesn't fix the problem; it just removes your ability to know about it. Module 6 goes deep on this.

#### Red flag 4: Magic values

```python
if user.plan == 3 and days > 14:
    time.sleep(2.5)
    send_email(user, template=7)
```

What is plan 3? Why 14 days? Why sleep for 2.5 seconds (spoiler: to hide a race condition)? What is template 7? Magic values are decisions with the explanation deleted.

```python
TRIAL_LENGTH_DAYS = 14

if user.plan == Plan.PRO_TRIAL and days_since_signup > TRIAL_LENGTH_DAYS:
    send_email(user, template=EmailTemplate.TRIAL_EXPIRED)
```

And if you see a `sleep()` "fixing" something, that's a bug wearing a disguise. Ask what it's waiting for.

#### Red flag 5: Hallucinated APIs and packages

The model confidently calls `stripe.charges.createAndCapture()` or `supabase.auth.getUserSafe()`. These sound plausible. They don't exist. Sometimes the code fails immediately (good, at least you notice). Sometimes it's in a branch that rarely runs (bad: it fails in production months later). Sometimes it's a whole **package** that doesn't exist, which attackers now register on purpose. A USENIX Security 2025 study of 576,000 generated code samples found that 19.7% of recommended packages were hallucinations ([Spracklen et al., USENIX Security 2025](https://www.usenix.org/conference/usenixsecurity25/presentation/spracklen)). The supply-chain side of this is covered in [Module 2](02-security-vulnerabilities.md).

**How to check:** "Go to Definition" on the method. If your editor can't find it in the library's type definitions, be very suspicious. Search the official docs, not the AI's memory. For packages, look them up on npm/PyPI: real download counts, a real repo, a real maintainer, a history longer than last week.

#### Red flag 6: Over-engineering

You asked for a contact form. You got an `AbstractFormHandlerFactory`, a dependency injection container, a plugin system, a custom event bus, and three interfaces with one implementation each.

AI over-engineers because its training data is full of enterprise patterns and because "more code" looks like "more effort." Every abstraction is something you must understand to change anything. The right question is: **"What is the simplest thing that would work, and what would make me need more?"** If the answer to the second part is "nothing we're planning," delete the scaffolding.

#### Bonus red flags (quick list)

- `// TODO: add auth` or `// TODO: validate input` in code that's being shipped
- `any` types everywhere in TypeScript, or `# type: ignore` sprinkled around
- Disabled lint rules (`eslint-disable`, `# noqa`) right above suspicious code
- Hard-coded URLs, IDs, emails, or keys
- Commented-out blocks of old code "just in case" (that's what git is for)
- Tests that were deleted or `.skip`ped in the same diff that changed the logic
- Comments that describe what the code *used* to do

### 6. Asking the AI to explain (without letting it grade itself)

The AI is a fantastic tutor for code reading. It's a terrible judge of its own work. Use it to *explain*, then verify.

Good uses:

- "Explain this file to me like I'm a smart person who doesn't know this framework."
- "Trace what happens when a user clicks Delete, listing every file and function with line references."
- "What assumptions does this function make about its inputs?"
- "List every place this function is called."

Then check the claims. If it says "the route checks that the user owns the project on line 42," go look at line 42. When the AI is wrong about its own code, it's usually wrong in exactly the way that matters.

Bad use: "Is this code secure?" → "Yes! This code follows best practices." That's not a review. That's a horoscope.

A useful trick: **ask a fresh session to review the code, with no context about who wrote it.** Models are less defensive about code they don't "remember" writing. Better still, ask it to find problems, not to confirm quality.

### 7. Git habits that make review possible

Git is your undo button, your audit log, and your review tool. Vibe coders often treat it as the place where code goes to be backed up once a week. That's like having seatbelts and keeping them in the trunk.

**Habits:**

1. **Commit before every AI session.** Clean working tree, then prompt. If the agent wrecks things, `git restore .` or `git reset --hard` (on uncommitted local work only) takes you back.
2. **One logical change per commit.** "Add discount code validation" is a commit. "Stuff" is not.
3. **Review the diff before you commit.** `git diff` (unstaged) and `git diff --staged` (staged). Your AI tool's diff view is fine too. The point is that *a human looks at it.*
4. **Stage selectively.** `git add -p` lets you accept changes hunk by hunk. It's the manual version of "Accept some."
5. **Write commit messages for future you.** Say *why*, not just *what*. Future you will be debugging at night and will appreciate it.
6. **Use branches for experiments.** `git switch -c try-new-checkout`. If it works, merge. If not, delete the branch and pretend it never happened.
7. **Never let an agent force-push, rewrite history, or commit secrets.** If it suggests `git push --force` on a shared branch, the answer is no.

```bash
git status                  # what's changed?
git diff                    # show me exactly what
git add -p                  # accept piece by piece
git commit -m "Validate discount codes server-side; remove duplicate price calc"
git log --oneline -10       # recent history
git restore path/to/file    # throw away changes to one file
```

#### Code archaeology: `git log`, `git blame`, and `git bisect` in plain language

Git doesn't just save your work. It's a searchable history of *who changed what, when, and why*. Three commands turn it into a detective kit.

**`git log`: "What happened here recently?"**

```bash
git log --oneline -20                 # last 20 commits, one line each
git log --oneline -- src/lib/pricing.ts   # only commits that touched this file
git log -p -- src/lib/pricing.ts      # same, with the actual changes
git log -S "applyDiscount" --oneline  # commits that added or removed this exact text
```

That last one (the "pickaxe") is gold: "When did `applyDiscount` start being called on the checkout page?" One command.

**`git blame`: "Who last touched this line, and in which commit?"**

```bash
git blame -L 40,60 src/app/checkout/page.tsx   # annotate lines 40-60
git show <commit-sha>                           # then read that whole commit
```

Despite the name, blame isn't for blaming. When the "who" is an AI session, it tells you *which prompt* introduced the line, and the commit message (if you wrote a good one) tells you *why*. Most editors show blame inline (in VS Code and Cursor, extensions like GitLens make it a hover).

**`git bisect`: "It worked last week. Which commit broke it?"**

Bisect does a binary search through your history. You tell it one commit that was good and one that's bad; it checks out the middle; you test and say "good" or "bad"; it halves the range again. Even across 100 commits, it finds the culprit in about 7 steps.

```bash
git bisect start
git bisect bad                 # the current version is broken
git bisect good a1b2c3d        # this older commit worked
# git checks out a commit in the middle. Test it (run the app or a test), then:
git bisect good                # or: git bisect bad
# ...repeat until git prints "<sha> is the first bad commit"
git bisect reset               # return to where you started
```

If you have a test (or a script) that fails when the bug is present, git can do the whole thing for you:

```bash
git bisect start HEAD a1b2c3d
git bisect run npm test -- src/lib/pricing.test.ts
git bisect reset
```

Bisect only works well if your commits are **small and each one runs**. That's one more reason for the habits above: a history of "wip", "stuff", and "fix" commits, half of which don't build, turns your detective kit into a pile of noise.

---

## What Good Looks Like

**A good review session for a single AI change:**

- You know what you asked for in one sentence.
- The diff touches only the files you'd expect.
- You've read the AI's summary and every "also" in it.
- New inputs are validated on the server.
- Errors are handled visibly, not swallowed.
- No new duplicate logic: if similar code exists, it's reused.
- No new packages you haven't verified.
- You ran the change, including at least one failure case.
- It's committed with a meaningful message, separate from unrelated changes.

**The Code Review Checklist (print it, tape it to your monitor):**

- [ ] Does the change match the request, and only the request?
- [ ] Is the diff small enough that I actually read all of it?
- [ ] Did any deletions remove checks, validation, or tests?
- [ ] Is user input validated on the server?
- [ ] Are permissions checked on the server for every new endpoint or query?
- [ ] Are errors handled, logged, and surfaced honestly?
- [ ] Any magic numbers or strings that need a name?
- [ ] Any duplicated logic that should call an existing function?
- [ ] Any dead code, commented-out code, or unused imports?
- [ ] Do all new methods and packages actually exist (checked in docs, not memory)?
- [ ] Is it simpler than I feared, or more complex than it needs to be?
- [ ] Did I run it, including an unhappy path?

**The "every diff" question card** (the short version of everything above; keep it next to your editor):

| # | Question | If the answer is bad... |
|---|---|---|
| 1 | What did I ask for, in one sentence? | You can't review what you can't state. Write it down first. |
| 2 | What did the AI *say* it did, and what are the "also"s? | Each "also" gets reviewed or reverted. |
| 3 | Which files changed, and should they have? | Unexpected files = stop and ask why. |
| 4 | What was **deleted**? | Deleted checks, validation, or tests = reject. |
| 5 | Where does new data come in, and is it validated on the server? | Add validation or reject. |
| 6 | Where does identity come from: session, or request? | Request-supplied user IDs = reject. |
| 7 | What happens when it fails? | Swallowed errors or fake success = fix. |
| 8 | Any new packages, APIs, or methods I haven't verified? | Verify in official docs and registries before accepting. |
| 9 | Did it duplicate something that already exists? | Point it at the existing function. |
| 10 | Did I run it, including one unhappy path? | Run it. |

---

## Prompt Templates

Copy, paste, adjust. These make the AI do the review-friendly version from the start.

**1. Small, reviewable changes**

```text
Make the smallest change that accomplishes the following: [task].
Constraints:
- Do not modify files outside: [list files/folders].
- Do not refactor, rename, or "clean up" unrelated code.
- Do not add new dependencies without asking me first.
- After the change, list every file you modified and why, and call out
  anything you deleted.
```

**2. Explain before changing**

```text
Before writing any code, explain how [feature] currently works: list the
files and functions involved in order, from UI to database, with line
references. Point out anything that looks duplicated, unused, or risky.
Don't change anything yet.
```

**3. Request trace**

```text
Trace what happens when a user [action]. For each step, give: file, function,
what data is passed, where authentication/authorization is checked (or say
explicitly that it is NOT checked), and what happens on failure.
```

**4. Red-flag review (use a fresh chat)**

```text
Review the following diff as a skeptical senior engineer. Do not praise it.
Find: dead code, duplicated logic, swallowed errors, magic values, calls to
methods or packages that may not exist, unnecessary abstractions, missing
input validation, and missing permission checks. For each finding, give the
line, why it matters, and the minimal fix. If you're unsure whether an API
exists, say so rather than guessing.

[paste diff]
```

**5. Verify an API**

```text
You used [library].[method]. Show me the exact signature from the official
documentation for version [x.y] and link to it. If you can't confirm it
exists, say so and rewrite the code using a method you can confirm.
```

**6. Kill the over-engineering**

```text
This implementation has [N] layers/classes/interfaces. Rewrite it as the
simplest version that meets these requirements: [requirements]. Remove any
abstraction that has only one implementation and no planned second one.
```

---

## Exercise

**Part A: The diff review (20 minutes)**

1. Open any project you've built with AI. Pick the last substantial AI change (use `git log` and `git show <commit>`, or your tool's history).
2. Run through the Code Review Checklist above. Write down every finding, even small ones.
3. Count how many red flags you find in each category: dead code, duplication, swallowed errors, magic values, hallucinated APIs, over-engineering.

**Part B: The trace (20 minutes)**

1. Pick one action that changes data (create, update, or delete something).
2. Trace it end to end by hand, using Go to Definition. Draw the chain like the example in Core Concepts.
3. Mark where authentication and authorization are checked. If you can't find where, that's a finding (and a trip to [Module 2](02-security-vulnerabilities.md)).
4. Now ask the AI to do the same trace with Prompt Template 3. Compare. Where did it disagree with you? Who was right?

**Part C: The git habit (ongoing)**

For the next week, commit before every AI session and review every diff before committing. At the end of the week, look at `git log`. Is it a story you could follow?

---

## Key Takeaways

- You don't need to write code from scratch to own it. You do need to **read** it. Reading is the minimum viable engineering skill.
- Read the AI's summary first. Every "I also..." is a side quest you didn't approve.
- Review the file list and the size of a diff before the content. Big, sprawling diffs hide bugs.
- Deletions are as dangerous as additions. AI loves removing the "redundant" check that was actually your security.
- Learn the red flags: dead code, duplicated logic, swallowed errors, magic values, hallucinated APIs, over-engineering.
- Trace requests end to end. If you can't say where permissions are checked, assume they aren't.
- Use the AI to explain and to hunt for problems, then verify its claims. Never accept "looks good to me" from the thing that wrote it.
- Small commits plus diff review turn a bad AI suggestion from a disaster into a `git restore`.

---

**Next up:** [Module 4: Testing Beyond "Ask Claude for Tests"](04-testing.md)
