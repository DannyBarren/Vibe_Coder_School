# Module 9: Tech Debt, Refactoring & Knowing When to Stop Vibing

> Every "just make it work" is a loan.
> The AI approves every application instantly, no credit check.
> The interest rate is variable, and it compounds.

You made it to the last module. If you've done the exercises, you're already not a pure vibe coder anymore. You read diffs. You test behavior. You know where your logic lives, what happens when it fails, how your data is shaped, and how to get it into production without lighting money on fire.

This module is about the long game. Every codebase accumulates mess. AI-assisted codebases accumulate it *faster*, because generating code is nearly free and understanding it is not. This module covers how to recognize technical debt, how to escape the spiral where every fix breaks two things, how to refactor safely, when to rewrite instead, when to bring in a real engineer, and finally, how to tell how far you've come.

---

## Learning Objectives

By the end of this module you will be able to:

1. Explain technical debt as a loan with interest, and tell deliberate debt from accidental debt.
2. Recognize the AI-generated spaghetti spiral and break out of it.
3. Refactor safely: tests first, small steps, behavior unchanged.
4. Decide between refactoring and rewriting using concrete criteria.
5. Recognize the situations where you should bring in an experienced engineer, and how to make that engagement worth it.
6. Place yourself on the maturity ladder from vibe coder to AI-assisted engineer, and know the next rung.

---

## Vibe Fail: The Fix That Broke Two Things (×40)
*Illustrative composite.*

Alex's habit-tracking app had a bug: streaks reset at midnight UTC instead of the user's local midnight. Alex pasted the bug into the chat. The AI fixed it.

The fix changed a date helper used in four places. Now the weekly summary email counted the wrong days. Alex pasted that bug in. The AI fixed it by adding a *second* date helper just for emails. Now there were two helpers that disagreed. The calendar view used one, the streak logic used the other, and users saw a streak of 12 next to a calendar showing 11.

Alex pasted that in. The AI "unified" them, which reintroduced the original midnight bug, plus a new one in the export feature. Alex asked the AI to "fix all the date bugs." It rewrote 600 lines across 14 files. The app now crashed for users in time zones with half-hour offsets.

Six weeks, roughly forty prompts, and one very long chat history later, the date logic was worse than when Alex started. Every prompt took longer. The AI's context window couldn't hold the relevant code anymore, so each fix was made without seeing the whole picture. There were no tests, so nobody (human or AI) could tell which fixes had broken what.

Alex didn't need a better prompt. Alex needed to **stop prompting**, write ten tests that described correct date behavior, consolidate to one date module, and refactor in small, verified steps. That took two afternoons.

---

## Core Concepts

### 1. Technical debt: a loan with interest

Ward Cunningham introduced the debt metaphor in 1992: "Shipping first time code is like going into debt. A little debt speeds development so long as it is paid back promptly with a rewrite" ([Cunningham, OOPSLA '92](https://c2.com/doc/oopsla92.html)). He also warned that the danger comes when the debt is not repaid: every minute spent on not-quite-right code counts as interest.

Translate that to your app:

- **Principal:** the shortcut you took. Hard-coded values, a copy-pasted function, no tests, everything in one component, a JSON blob instead of a table.
- **Interest:** the extra time and risk every future change costs because of that shortcut. Bugs that come back. Features that take three days instead of three hours. Fear of touching certain files.
- **Default:** the point where interest exceeds your capacity. Every change breaks something, nobody understands the system, and progress stops.

**Debt isn't automatically bad.** A prototype built to test whether anyone wants the product *should* take on debt. Speed matters more than structure when you might throw it away next week. The problem is:

| Deliberate debt | Accidental debt |
|---|---|
| "We're hard-coding this for the demo; ticket to fix before launch." | "Why are there three `formatDate` functions?" |
| Known, written down, has a repayment plan | Unknown until it bites |
| A conscious trade-off (README first principle #3) | A trade-off nobody made |

Pure vibe coding generates accidental debt at industrial scale. GitClear's analysis of 211 million changed lines found that in 2024, copy/pasted code exceeded "moved" code (a signal of refactoring) for the first time in their data, while the frequency of duplicated code blocks (five or more duplicated lines) rose eightfold during 2024 ([GitClear 2025](https://www.gitclear.com/ai_assistant_code_quality_2025_research/)). More code is being written; less is being consolidated. That's a lot of loans and not many repayments.

**Common forms of debt in AI-built apps** (you've met all of them in earlier modules):

- Duplicated logic and multiple sources of truth ([Module 3](03-reading-code.md), [Module 5](05-architecture.md))
- No tests, or theater tests ([Module 4](04-testing.md))
- Giant components and logic in the wrong layer ([Module 5](05-architecture.md))
- Swallowed errors and no observability ([Module 6](06-observability.md))
- A schema that grew by accident ([Module 7](07-data-modeling.md))
- Manual, unrepeatable deploys ([Module 8](08-deployment.md))
- Dead code, abandoned approaches, unused dependencies
- Security shortcuts ("we'll add auth later"). This is the most dangerous kind; see [Module 2](02-security-vulnerabilities.md).

### 2. The AI-generated spaghetti spiral

The spiral has a recognizable shape:

```text
bug appears
  → paste error into AI
    → AI patches symptom locally (it can't see the whole system)
      → patch breaks something elsewhere (no tests to catch it)
        → paste new error into AI
          → AI adds a workaround / duplicate / special case
            → complexity grows, context window overflows
              → AI sees even less of the system → worse patches
                → (repeat until despair)
```

**Signs you're in the spiral:**

- The same bug (or its cousin) has come back more than twice.
- Fixes keep getting bigger for problems that feel small.
- You've lost track of what the code is supposed to do in some area.
- The AI is contradicting its own earlier changes, or "fixing" things by reverting them.
- You're afraid to touch certain files.
- You have special cases for special cases (`if (user.id === "abc123") ...`).
- Your chat history for one feature is longer than the feature.

**How to break out:**

1. **Stop prompting for fixes.** Seriously. More patches feed the spiral.
2. **Commit or stash** where you are, so you can always get back.
3. **Write down what correct behavior is**, in plain language. Five to fifteen statements: "A streak counts consecutive days in the user's local time zone. Missing a day resets it to 0..."
4. **Turn those statements into tests** ([Module 4](04-testing.md)). Expected values from the requirements, not the code. Many will fail. Good. Now you have a map of what's broken.
5. **Find the single source of truth** (or create one). One date module, one pricing function, one permission check.
6. **Refactor in small steps** toward it, running tests after each (next section).
7. **Delete the dead ends**: duplicate helpers, workarounds, special cases.
8. **Start a fresh AI session** with a clean, focused context: the requirements, the tests, and the module. Long, polluted chat histories make AI output worse.

### 3. Refactoring safely

**Refactoring** means changing the structure of code **without changing its behavior.** Not adding features. Not fixing bugs (do those separately). Just making the code easier to understand and change.

The safety rules:

1. **Tests first.** If there are no tests covering the behavior, write characterization tests: tests that capture what the code does *today* (yes, this is the one time "assert whatever the code does" is useful, because you're trying to preserve behavior, bugs included, while you restructure). Fix bugs *after* the refactor, as separate commits.
2. **One small step at a time.** Extract one function. Rename one thing. Move one file. Run tests. Commit.
3. **Never mix refactoring with behavior changes** in the same commit. If tests fail after a "pure refactor," you know the refactor broke something.
4. **Keep the app working at every step.** If you can't deploy between steps, the steps are too big.
5. **Review every diff** ([Module 3](03-reading-code.md)). AI refactors love to sneak in "improvements."

**Common, safe refactoring moves:**

| Move | Example |
|---|---|
| Extract function | Pull pricing math out of a component into `lib/pricing.ts` |
| Consolidate duplicates | Replace three `formatDate` functions with one, update callers one at a time |
| Rename for clarity | `data2` → `activeSubscriptions`; `plan === 3` → `Plan.PRO` |
| Move to the right layer | DB query from component → repo function; rule from client → server |
| Replace magic values | `14` → `TRIAL_LENGTH_DAYS` |
| Delete dead code | Remove unused functions, flags, files (git remembers them) |
| Split giant files | `Dashboard.tsx` → `ProjectList`, `Filters`, `BillingBanner` |

**Bad vs. good refactoring prompts:**

```text
BAD:  "Refactor the whole app to be clean and follow best practices."
      (Result: 4,000-line diff, new architecture, subtle behavior changes, nobody can review it.)

GOOD: "Extract the streak calculation from components/Streak.tsx into a pure
       function in lib/streaks.ts. Do not change behavior. Update the one
       caller. Do not touch any other file. Run the tests and show me the result."
```

**The strangler pattern** for bigger changes: build the new version *alongside* the old one, route a small piece of functionality to the new version, verify, move the next piece, and remove the old code once nothing uses it. It's expand/contract ([Module 7](07-data-modeling.md)) applied to code.

### 4. Rewrite vs. refactor

Every messy codebase eventually triggers the thought: "It'd be faster to just start over." With AI, starting over feels especially cheap. Sometimes it is the right call. Usually it isn't.

Why rewrites are riskier than they look:

- The old code, ugly as it is, contains **hundreds of decisions and bug fixes** you've forgotten about: edge cases, weird customer requirements, workarounds for third-party quirks. A rewrite loses them and rediscovers them one bug report at a time.
- While you rewrite, the old system still needs maintenance. Now you have two systems.
- If you rewrite with the same habits that created the mess (pure vibing, no tests, no structure), you'll get the same mess, just newer.
- **Your data doesn't get rewritten.** Whatever you build must work with the existing database and its history ([Module 7](07-data-modeling.md)).

**Lean toward refactoring when:**

- The app works for users and the problems are mostly internal (hard to change, slow to develop).
- The data model is roughly right.
- The problems are concentrated in some areas, not everywhere.
- You can write tests around the current behavior.

**A rewrite may be justified when:**

- It's still a prototype with few or no real users, and you've learned what the product actually is.
- The foundation is wrong in a way refactoring can't fix: wrong data model at the core, wrong platform for your needs, a security architecture that can't be patched (e.g., all authorization done in the client against a database exposed to the internet).
- Nobody can understand the code, no tests are possible, and each change carries unacceptable risk.
- The cost of continuing (interest) clearly exceeds the cost of rebuilding plus migration, and you've actually estimated both.

**If you rewrite, rewrite like an engineer:** write down requirements first (mine the old code and bug history for them), write tests from those requirements, design the data model and migration path from the old data, set up the architecture rules file ([Module 5](05-architecture.md)) before the first prompt, and migrate gradually (strangler pattern) rather than with a big-bang switchover.

#### A rewrite-vs-refactor decision table

When the "just start over" urge hits, score your situation honestly. Each row leans one way:

| Question | Leans **refactor** | Leans **rewrite** |
|---|---|---|
| Do real users depend on it today? | Yes, many, or paying | Few or none (still a prototype) |
| Is the data model roughly right? | Yes, mostly | No: the core entities or relationships are wrong |
| Where are the problems? | Concentrated in a few areas | Everywhere, including the foundations |
| Can you write tests around current behavior? | Yes | No: behavior is unknowable or random |
| Is the security architecture fixable in place? | Yes (missing checks can be added) | No (e.g., all authorization in the client against an exposed database) |
| Do you understand what it's supposed to do? | Yes, and it's documented or testable | No, and the old code is the only spec, but it's wrong |
| Is the platform/stack fundamentally unsuited? | No, just messy | Yes: you've hit hard limits of the platform |
| Can you migrate gradually (strangler)? | n/a | Yes, which makes a rewrite much safer |
| Can you afford months of maintaining two systems? | n/a | Only rewrite if yes |

**How to read it:** if most answers land in the refactor column, refactor, even if it feels slower. If most land in the rewrite column, a rewrite may be justified, but write down the requirements and the data migration plan first, and estimate both paths. If it's mixed, refactor the worst area first and re-score in a month; you'll often find the "rewrite" urge was really a few terrible files.

One honest test: *"If I rewrote this with AI next week, using the same habits, would the result be better?"* If the answer is "only if I change how I work," then change how you work first. That fixes the existing code too.

### 5. When to bring in a real engineer

Using AI doesn't mean you have to do everything alone. Knowing when to get help is a professional skill, not a failure. Here are the moments when an experienced engineer is worth the money:

**Bring someone in before you:**

- Handle **payments** beyond a hosted checkout, or store any financial data.
- Store **health, children's, financial, or government-ID data**, or anything regulated (HIPAA, PCI DSS, GDPR obligations, SOC 2 requests from customers).
- Onboard **business customers** who will ask security questionnaires.
- Scale past the point where downtime or data loss would seriously hurt people (or your revenue).
- Do a **major data migration** or schema overhaul on production data.
- **Rewrite** the core of a live product.

**Bring someone in when you notice:**

- You've been in the spaghetti spiral for more than a week.
- You can't explain how authentication and authorization work in your own app.
- You suspect a security issue and don't know how to confirm or fix it.
- A security researcher, customer, or platform has reported a vulnerability.
- Your cloud bill is growing faster than your users and you can't explain why.
- You're afraid to deploy.

**How to make that engagement worth it** (and cheaper):

- Do the work from this course first: a README, an architecture doc, tests, a `.env.example`, a list of known issues. An engineer who can understand your system in two hours instead of two days costs a lot less.
- Ask for a **review and a plan**, not just "fix it." A one- or two-day code and security review with a prioritized list of findings is often the best money you'll spend.
- Pay for a **security review or penetration test** before launch if you handle sensitive data ([Module 2](02-security-vulnerabilities.md)).
- **Learn while they work.** Ask them to explain decisions. Pair on the fixes. Your goal is to need them less next time.

#### What to bring a hired engineer: the handoff doc

Engineers bill by the hour, and the first hours on an unfamiliar AI-built codebase are mostly archaeology. A two-page handoff doc can save a day or more of that. Put it in `docs/HANDOFF.md`:

```markdown
# Handoff: [App name]

## What it is (3 sentences)
What the app does, who uses it, and what matters most (e.g., "Small businesses
send invoices and get paid by card. ~300 active accounts, ~$8k/month processed.").

## Why I'm bringing you in
The specific problem or goal: "security review before our first enterprise
customer", "checkout keeps breaking", "plan a migration off X".
What success looks like at the end of the engagement.

## Stack and hosting
Framework, language, database, auth provider, payments, AI APIs, email, hosting.
Links to dashboards (access granted separately, least privilege).

## How to run it
Clone → install → copy .env.example → seed → run → test. Exact commands.
Known setup gotchas.

## Architecture
Layers and folders (or link to AGENTS.md / docs/decisions/).
Diagram or list of the main request flows (login, core action, payments, webhooks).

## Data
Main tables and relationships (or link to schema). Multi-tenancy approach.
Where personal / sensitive data lives. Backup setup and last tested restore date.

## Auth and permissions
How login works. Where authorization is enforced (server? RLS?).
What I'm unsure about.

## Environments and deploys
Which environments exist, how deploys happen (CI/CD link), how to roll back.
Who has production access.

## Known issues and debt
Link to DEBT.md. Top 5 problems in my own words. Recent incidents.

## Things I'm least confident about
The honest list. This is where your money is best spent.

## Constraints
Budget, deadlines, things that must not change (e.g., URLs, a customer's integration).
```

Also give them: a repo invite with the right permissions, read-only or staging access before production access, test accounts, and 30 minutes on a call where you click through the app. **Never** send production secrets over chat or email; use your platform's team access or a password manager's sharing feature.

### 6. The debt register: making debt visible

Debt you can't see, you can't prioritize. A **debt register** is a simple file (`DEBT.md`) or an issue label where every known shortcut gets written down. It turns "the codebase feels bad" into a list you can actually work through.

```markdown
# Debt register

Last reviewed: 2026-10-17

| ID | What | Where | Interest (why it hurts) | Risk | Effort | Fix idea | Status |
|----|------|-------|-------------------------|------|--------|----------|--------|
| D1 | Plan check done in browser | components/Export.tsx | Free users can export | **Security / High** | S | Move hasProFeatures() server-side, add auth test | Open |
| D2 | 3 date helpers disagree | lib/, emails/, components/ | Streak bugs keep coming back | High | M | Consolidate into lib/dates.ts (see section 8) | In progress |
| D3 | No tests on pricing | lib/pricing.ts | Every change is scary | High | M | Characterization tests, then requirement tests | Open |
| D4 | Dashboard.tsx is 2,900 lines | components/ | AI edits break unrelated features | Medium | L | Extract per Module 5 plan, one step per commit | Open |
| D5 | Hard-coded staff discount email | services/orders.ts | Surprise $0 orders | Medium | S | Move to a role/flag in the DB | Open |
| D6 | Unused Stripe v1 webhook route | app/api/stripe-old/ | Confuses AI sessions, attack surface | Low | S | Delete | Open |

## Done
- 2026-10-10 — D0: swallowed errors in signup (fixed, regression test added)
```

How to keep it useful:

- **Add an entry the moment you take a shortcut**, deliberately or not. "I'll remember" is how accidental debt is born.
- **Security items go to the top and get fixed first**, always.
- **Rank by interest, not by ugliness.** Ugly code nobody touches costs nothing. Mediocre code you change every week costs a lot. Use the "most changed files" command from the triage checklist.
- **Link entries to evidence:** incidents, recurring bugs, slow features.
- **Review it monthly**, and celebrate the "Done" section growing. It's the most honest progress report your project has.
- **Tell the AI about it.** "Read DEBT.md before starting; don't make D2 worse" keeps new sessions from adding a fourth date helper.

### 7. The boy scout routine: a little cleanup every week

The old rule: *leave the campground cleaner than you found it.* Applied to code, it means every time you touch an area, you leave it slightly better. Plus a small, scheduled cleanup so debt repayment isn't something you only do when things are on fire.

**Every change (2-10 minutes):**

- Delete dead code and unused imports in the files you touched.
- Rename one confusing variable or magic number you had to puzzle over.
- Add one missing test for the behavior you just changed.
- If you added a shortcut, add it to `DEBT.md`.

**Weekly (30-60 minutes, same time every week):**

1. Check the error tracker: any recurring issue that points at a structural problem? Add it to the register.
2. Pick **one** item from `DEBT.md` (highest interest you can finish in the time box).
3. Fix it in small commits, tests passing at each step.
4. Run a quick dependency check (`npm outdated` / `npm audit`, `pip list --outdated` / `pip-audit`) and note anything risky (security updates first; see [Module 2](02-security-vulnerabilities.md)).
5. Delete one thing: a stale feature flag, an unused env var, an abandoned branch, a dead file.
6. Update `DEBT.md` and the rules file if you learned something.

**Monthly (1-2 hours):** run the debt triage checklist, review alerts for noise ([Module 6](06-observability.md)), and do a quick restore or rollback drill on rotation ([Module 7](07-data-modeling.md), [Module 8](08-deployment.md)).

Small, regular payments beat heroic refactoring weekends. The weekend never comes, and when it does, you're too far behind.

### 8. Worked example: refactoring the date mess, with tests

Let's fix Alex's streak bug from the Vibe Fail properly. The original code:

```ts
// BEFORE — lib/streaks.ts (dates handled in UTC by accident)
export function dayKey(d: Date): string {
  return d.toISOString().slice(0, 10); // UTC calendar day, not the user's!
}

export function currentStreak(checkIns: Date[], now: Date): number {
  const days = new Set(checkIns.map(dayKey));
  let streak = 0;
  let cursor = new Date(now);
  while (days.has(dayKey(cursor))) {
    streak++;
    cursor = new Date(cursor.getTime() - 24 * 60 * 60 * 1000);
  }
  return streak;
}
```

Two problems hiding in eight lines: `toISOString()` gives the **UTC** day, not the user's day; and stepping back exactly 24 hours breaks on days that are 23 or 25 hours long (daylight saving changes; see [Module 7](07-data-modeling.md)).

**Step 1: characterization tests** (commit: "Pin current streak behavior"). Capture what the code does today, *including* the bug, clearly labeled:

```ts
// Step 1 — characterization tests: pin down what the code does TODAY (bug included)
import { describe, expect, it } from "vitest";
import { currentStreak } from "./before";

describe("currentStreak (current behavior)", () => {
  it("counts consecutive UTC days", () => {
    const checkIns = [new Date("2026-03-01T12:00:00Z"), new Date("2026-03-02T12:00:00Z")];
    expect(currentStreak(checkIns, new Date("2026-03-02T18:00:00Z"))).toBe(2);
  });

  it("KNOWN BUG: a New York evening check-in lands on the next UTC day", () => {
    // 6pm Mar 1 and 8pm Mar 2 in New York = 23:00Z Mar 1 and 01:00Z Mar 3
    const checkIns = [new Date("2026-03-01T23:00:00Z"), new Date("2026-03-03T01:00:00Z")];
    const now = new Date("2026-03-03T02:00:00Z"); // 9pm Mar 2 in New York
    expect(currentStreak(checkIns, now)).toBe(1); // the user expects 2
  });
});
```

Now you have a safety net and a precise, reproducible statement of the bug.

**Step 2: introduce the single source of truth** (commit: "Add lib/dates.ts with localDay and previousDay"). New code, new tests, nothing uses it yet:

```ts
// AFTER — lib/dates.ts: the ONE place that turns instants into calendar days
const formatters = new Map<string, Intl.DateTimeFormat>();

/** The user's local calendar day for an instant, as "YYYY-MM-DD". */
export function localDay(date: Date, timeZone: string): string {
  let fmt = formatters.get(timeZone);
  if (!fmt) {
    // en-CA formats dates as YYYY-MM-DD
    fmt = new Intl.DateTimeFormat("en-CA", { timeZone, year: "numeric", month: "2-digit", day: "2-digit" });
    formatters.set(timeZone, fmt);
  }
  return fmt.format(date);
}

/** The calendar day before "YYYY-MM-DD". Pure date math: no time zones, no DST surprises. */
export function previousDay(day: string): string {
  const [y, m, d] = day.split("-").map(Number);
  return new Date(Date.UTC(y, m - 1, d - 1)).toISOString().slice(0, 10);
}
```

**Step 3: switch callers one at a time** (one commit each): the email summary, the calendar view, then the export. Run tests after each. Delete each old helper once nothing references it (your editor's "Find All References" or `rg "emailDay"` tells you when).

**Step 4: fix the bug, as its own commit** (commit: "Fix streaks to use the user's local day"). This is a deliberate behavior change, so the test that pinned the bug is replaced by one that states the requirement:

```ts
// AFTER — lib/streaks.ts
import { localDay, previousDay } from "./dates";

export function currentStreak(checkIns: Date[], now: Date, timeZone: string): number {
  const days = new Set(checkIns.map((d) => localDay(d, timeZone)));
  let streak = 0;
  for (let day = localDay(now, timeZone); days.has(day); day = previousDay(day)) {
    streak++;
  }
  return streak;
}
```

```ts
import { describe, expect, it } from "vitest";
import { localDay, previousDay } from "./dates";
import { currentStreak } from "./streaks";

describe("localDay", () => {
  it("uses the user's time zone, not UTC", () => {
    expect(localDay(new Date("2026-03-01T03:30:00Z"), "America/New_York")).toBe("2026-02-28");
  });
  it("handles half-hour offsets", () => {
    expect(localDay(new Date("2026-03-01T20:00:00Z"), "Asia/Kolkata")).toBe("2026-03-02");
  });
});

describe("previousDay", () => {
  it("crosses month, leap-day, and year boundaries", () => {
    expect(previousDay("2026-03-01")).toBe("2026-02-28");
    expect(previousDay("2028-03-01")).toBe("2028-02-29");
    expect(previousDay("2027-01-01")).toBe("2026-12-31");
  });
});

describe("currentStreak", () => {
  const NY = "America/New_York";

  it("FIXED: New York evening check-ins count on the local day", () => {
    const checkIns = [new Date("2026-03-01T23:00:00Z"), new Date("2026-03-03T01:00:00Z")];
    expect(currentStreak(checkIns, new Date("2026-03-03T02:00:00Z"), NY)).toBe(2);
  });

  it("survives the DST change (Mar 8, 2026 has 23 hours in New York)", () => {
    const checkIns = [
      new Date("2026-03-08T01:30:00Z"), // 8:30pm Mar 7 (EST)
      new Date("2026-03-09T00:30:00Z"), // 8:30pm Mar 8 (EDT)
      new Date("2026-03-10T00:30:00Z"), // 8:30pm Mar 9 (EDT)
    ];
    expect(currentStreak(checkIns, new Date("2026-03-10T01:00:00Z"), NY)).toBe(3);
  });

  it("is 0 when there is no check-in today", () => {
    expect(currentStreak([new Date("2026-03-01T12:00:00Z")], new Date("2026-03-03T12:00:00Z"), NY)).toBe(0);
  });
});
```

Look at what made this safe:

- Behavior was pinned **before** anything changed.
- Structure changes (steps 2-3) and behavior changes (step 4) were in **separate commits**. If something breaks, `git bisect` ([Module 3](03-reading-code.md)) points straight at the culprit.
- The tests include the edge cases that caused the spiral in the first place: evening check-ins, half-hour time zones, leap days, and DST.
- Each step was small enough to review in a minute.

The AI can do every one of these steps for you, *one at a time*, using Prompt Templates 3 and 4. What it can't do is decide the steps, the order, and when each one is verified. That part is engineering.

### 9. Measuring debt: churn × complexity

"This codebase has a lot of debt" is a feeling. "These three files account for most of our pain" is a plan. The most useful cheap measurement comes from Adam Tornhill's work on behavioral code analysis (his book *Your Code as a Crime Scene* and the open-source [code-maat](https://github.com/adamtornhill/code-maat) tool): **hotspots**, the files that are both *complicated* and *changed often*.

The logic is simple once you say it out loud:

- A messy file nobody touches costs you almost nothing. Leave it.
- A clean file that changes daily is fine. That's healthy work.
- A **big, messy file that changes every week** is where your time, bugs, and AI confusion pile up. That's where repayment pays off.

You can approximate it with git and a line count (lines of code is a crude but surprisingly useful stand-in for complexity):

```bash
#!/usr/bin/env bash
# hotspots.sh: files that change often AND are big. Run from the repo root.
# usage: ./hotspots.sh "6 months ago"
since="${1:-6 months ago}"

git log --since="$since" --format= --name-only \
  | grep -E '\.(ts|tsx|js|jsx|py)$' \
  | sort | uniq -c \
  | while read -r changes file; do
      [ -f "$file" ] || continue                 # skip files that were deleted or renamed
      lines=$(wc -l < "$file")
      echo "$((changes * lines)) $changes $lines $file"
    done \
  | sort -rn | head -15 \
  | awk 'BEGIN { printf "%8s %8s %6s  %s\n", "score", "changes", "lines", "file" }
               { printf "%8s %8s %6s  %s\n", $1, $2, $3, $4 }'
```

Here's typical output for a vibe-coded app *(illustrative)*:

```text
   score  changes  lines  file
   28830       31    930  src/app/dashboard/page.tsx
    3410       11    310  src/lib/billing.ts
     880       16     55  src/lib/format.ts
```

Read it like this: `format.ts` changed often, but it's small, so it's probably fine. `billing.ts` is medium on both, so it's worth a look (it's billing, after all). `dashboard/page.tsx` is the hotspot: 930 lines, changed 31 times in the period, and nearly ten times the score of anything else. If you only refactor one thing this quarter, it's that file. It's also almost certainly where your AI sessions go off the rails, because it barely fits in the context window.

Tips that make the numbers honest:

- **Use a time window** (the last 3-6 months). Ancient history doesn't predict where you'll be working next.
- **Pair it with your debt register** (section 6) and your bug history. If the hotspot is also where the last five bugs came from, the case makes itself.
- **Re-run it after refactoring.** A hotspot that drops down the list is the most satisfying chart in software.
- If you want better complexity numbers than line counts, linters can report cyclomatic complexity (for example ESLint's `complexity` rule, or `radon` for Python). Start with line counts; they're usually enough to find the worst file.

### 10. Worked example: untangling an AI spaghetti function

Here's a different flavor of mess from section 8's date bug: a function that grew by accretion, one prompt at a time ("add refunds to the stats," "add a tier," "also show the last 30 days"). *(Illustrative, but you've probably seen its cousin.)*

```js
// lib/stats.js (BEFORE): what the AI produced after a dozen "add X to the stats" prompts
export async function getStats(db, userId) {
  try {
    const orders = await db.getOrders(userId);
    let total = 0, count = 0, refunded = 0;
    for (let i = 0; i < orders.length; i++) {
      if (orders[i].status != "cancelled") {
        if (orders[i].status == "refunded") {
          refunded = refunded + orders[i].amount;
        } else {
          if (orders[i].amount > 0) {
            total = total + orders[i].amount;
            count++;
          }
        }
      }
    }
    let avg = count > 0 ? total / count : 0;
    let tier = "bronze";
    if (total > 50000) tier = "silver";
    if (total > 200000) tier = "gold";
    if (total > 1000000) tier = "platinum";
    // last 30 days
    const recent = orders.filter((o) => new Date(o.createdAt) > new Date(Date.now() - 2592000000));
    let recentTotal = 0;
    for (let i = 0; i < recent.length; i++) {
      if (recent[i].status != "cancelled") {
        if (recent[i].status == "refunded") {
        } else {
          if (recent[i].amount > 0) recentTotal = recentTotal + recent[i].amount;
        }
      }
    }
    return { total, count, avg: Math.round(avg), refunded, tier, recentTotal };
  } catch (e) {
    console.log(e);
    return { total: 0, count: 0, avg: 0, refunded: 0, tier: "bronze", recentTotal: 0 };
  }
}
```

Before touching it, list what's wrong, and classify each item as **structure** (safe to change without changing behavior) or **behavior** (changes what users see, needs a decision):

- *Structure:* the completed-order logic is written twice (once for totals, once for "recent"), with nested `if`s and an empty branch. Magic numbers: `2592000000` (30 days in milliseconds), `50000`, `200000`, `1000000` (tier thresholds in cents). `Date.now()` is buried inside, so tests can't control time. Loose `!=`.
- *Behavior:* the `catch` swallows a database failure and returns **zeros and "bronze."** A platinum customer who opens their dashboard during a database blip is told they've spent $0 and lost their status. If anything downstream (perks, discounts, support priority) reads `tier`, the blip just downgraded them for real.

**Step 1: pin the current behavior with characterization tests.** These describe what the code does *today*, right or wrong, so you'll know if a refactor changes anything by accident. Freeze the clock so "last 30 days" is testable:

```js
// stats.test.js: characterization tests. They pin what the code does TODAY.
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { getStats } from "./lib/stats.js";

const order = (amount, status, createdAt) => ({ amount, status, createdAt });
const dbWith = (orders) => ({ getOrders: async () => orders });

beforeEach(() => {
  vi.useFakeTimers();
  vi.setSystemTime(new Date("2026-10-17T12:00:00Z"));
});
afterEach(() => vi.useRealTimers());

describe("getStats (characterization)", () => {
  it("sums completed orders and ignores cancelled ones", async () => {
    const stats = await getStats(dbWith([
      order(10_000, "paid", "2026-10-01T00:00:00Z"),
      order(5_000, "paid", "2026-01-01T00:00:00Z"),
      order(9_999, "cancelled", "2026-10-02T00:00:00Z"),
      order(2_500, "refunded", "2026-10-03T00:00:00Z"),
      order(0, "paid", "2026-10-04T00:00:00Z"),
    ]), "u_1");
    expect(stats).toEqual({
      total: 15_000, count: 2, avg: 7_500, refunded: 2_500, tier: "bronze", recentTotal: 10_000,
    });
  });

  it.each([
    [50_000, "bronze"], [50_001, "silver"], [200_001, "gold"], [1_000_001, "platinum"],
  ])("a lifetime total of %i cents is tier %s", async (amount, tier) => {
    const stats = await getStats(dbWith([order(amount, "paid", "2026-10-01T00:00:00Z")]), "u_1");
    expect(stats.tier).toBe(tier);
  });

  it("counts an order from exactly 30 days ago as not recent", async () => {
    const stats = await getStats(dbWith([order(1_000, "paid", "2026-09-17T12:00:00Z")]), "u_1");
    expect(stats.recentTotal).toBe(0);
  });
});
```

Notice the boundary tests: a total of exactly 50,000 cents is still bronze (the code uses `>`), and an order from exactly 30 days ago isn't "recent." You might not have chosen those rules, but they're the rules your users have today. Characterization tests make them visible; changing them is a separate, deliberate decision.

Also pin the swallowed error, explicitly labeled as current behavior:

```js
import { expect, it, vi } from "vitest";
import { getStats } from "./lib/stats.js";
it("CURRENT behavior: a database failure looks like a brand-new customer", async () => {
  vi.spyOn(console, "log").mockImplementation(() => {});
  const stats = await getStats({ getOrders: async () => { throw new Error("db down"); } }, "u_1");
  expect(stats).toMatchObject({ total: 0, tier: "bronze" });
});
```

Run the tests: all green against the old code. Commit.

**Step 2: extract and name things (structure only).** Pull the duplicated logic into one predicate, name the magic numbers, replace the nested `if`s with small pure functions, and pass `now` in instead of reaching for the clock. Run the characterization tests after each small move. Commit when green.

**Step 3: change the behavior on purpose, in its own commit.** Stop pretending a database failure is a brand-new customer: let the error propagate so the API returns a proper 500 ([Module 6](06-observability.md)) and the dashboard shows "couldn't load your stats" instead of a lie. Replace the "current behavior" test with the new rule:

```js
import { expect, it } from "vitest";
import { getStats } from "./lib/stats.js";
it("a database failure is reported as an error, not as $0 and bronze", async () => {
  const db = { getOrders: async () => { throw new Error("db down"); } };
  await expect(getStats(db, "u_1")).rejects.toThrow("db down");
});
```

The result:

```js
// lib/stats.js (AFTER)
const DAY_MS = 24 * 60 * 60 * 1000;
const RECENT_DAYS = 30;

// Lifetime spend thresholds in cents, highest first. "above" means strictly greater than.
const TIERS = [
  { name: "platinum", above: 1_000_000 }, // $10,000
  { name: "gold", above: 200_000 },       // $2,000
  { name: "silver", above: 50_000 },      // $500
];

const isCompleted = (order) =>
  order.status !== "cancelled" && order.status !== "refunded" && order.amount > 0;

const sumAmounts = (orders) => orders.reduce((sum, order) => sum + order.amount, 0);

export function tierFor(totalCents) {
  return TIERS.find((tier) => totalCents > tier.above)?.name ?? "bronze";
}

export function summarizeOrders(orders, now) {
  const completed = orders.filter(isCompleted);
  const total = sumAmounts(completed);
  const recentSince = now.getTime() - RECENT_DAYS * DAY_MS;

  return {
    total,
    count: completed.length,
    avg: completed.length > 0 ? Math.round(total / completed.length) : 0,
    refunded: sumAmounts(orders.filter((order) => order.status === "refunded")),
    tier: tierFor(total),
    recentTotal: sumAmounts(completed.filter((order) => new Date(order.createdAt).getTime() > recentSince)),
  };
}

export async function getStats(db, userId, now = new Date()) {
  const orders = await db.getOrders(userId); // a database failure is an error, not "you have $0"
  return summarizeOrders(orders, now);
}
```

Same six characterization tests, still passing, plus the new error test. What changed, in practical terms: the "what counts as a completed order" rule exists once, so the next prompt ("exclude test orders from stats") is a one-line change instead of two edits the AI will get half right. The tier table reads like a business rule a non-engineer could check against the pricing page. And the next time the database hiccups, nobody gets downgraded.

The order was the point: **pin, restructure, then change behavior**, each in its own commit. Asking the AI to "clean up this function and fix the error handling" in one go mixes all three, and if a number changes, you won't know whether it was the cleanup or the fix.

### 11. When to stop vibing: real-life signals

Section 5 covered *when to bring in an engineer*. This is the smaller, more frequent decision: **when to stop prompting and switch modes** (slow down, read, test, design) for a particular piece of work. *(Each scenario is an illustrative composite.)*

**"The third fix for the same bug."** You've asked the AI to fix the checkout total three times; each fix breaks a different case. *Signal:* nobody, including the AI, has a clear model of the rules. *Switch to:* write the rules down as a table of examples (inputs → expected total), turn them into tests, and then let the AI make the tests pass. Section 8's date story is this exact pattern.

**"It touches money, auth, or other people's data."** You're about to add team billing, or let users share documents. *Signal:* mistakes here are expensive, quiet, and sometimes irreversible. *Switch to:* threat-model it first ([Module 1, Part 6](01-security-reality.md#part-6-threat-modeling-lite-stride-in-plain-language)), design the schema and access rules yourself ([Module 7](07-data-modeling.md)), and review every line of the diff rather than skimming the summary.

**"The AI keeps forgetting how the app works."** It reinvents a helper that already exists, uses the wrong table, imports a library you removed. *Signal:* the code has outgrown what fits in a session's context. *Switch to:* run the hotspot script (section 9), split the biggest file, and write or update your rules file ([Module 5](05-architecture.md)).

**"You're afraid to deploy."** Every deploy breaks something, so you batch changes and deploy weekly, which makes each deploy bigger and scarier. *Signal:* no safety net. *Switch to:* stop adding features for a few days and build the net: CI with tests on the critical paths ([Module 4](04-testing.md)), preview deploys and rollback ([Module 8](08-deployment.md)).

**"A real customer depends on it now."** The first company pays you annually, or the first user puts their business's data in your app. *Signal:* the cost of a bad day just went from "embarrassing" to "contract-breaking." *Switch to:* check your level against the target for this kind of project (section 12), and close the gap before the next big feature.

**"It's a weekend experiment."** None of the above applies, nobody else's data is involved, and you might throw it away on Monday. *Signal:* this is exactly where vibing shines. *Keep vibing,* and enjoy it. The skill isn't abstaining from vibe coding; it's knowing which mode the work in front of you deserves.

### 12. Graduation: from vibe coder to AI-assisted engineer

Remember the README's first principles? Understand the problem, consider what can go wrong, make trade-offs consciously, design for verification and observation, account for the full lifecycle, validate assumptions with evidence, and own the outcome. Every module in this course was one of those principles turned into habits.

Here's the maturity ladder from [Module 0](../modules/00-welcome.md), which is its canonical home. Be honest about where you are. Everyone starts at the bottom.

**Level 0: Pure vibes.** Accept All, ship, pray. Doesn't read diffs or the AI's summary. No tests, no git habits, secrets wherever they landed.

**Level 1: Reader.** Reads the AI's summary and skims the diff. Commits before AI sessions. Knows secrets don't go in the repo.

**Level 2: Questioner.** Asks the AI to explain the risky parts and pushes back when the answer doesn't hold up. Reviews diffs with a checklist ([Module 3](03-reading-code.md)), including the deletions.

**Level 3: Verifier.** Doesn't take "it works" on faith: tests as a logged-out user and as a different user, runs scanners, checks access rules. Writes tests from requirements ([Module 4](04-testing.md)), has error tracking and alerts ([Module 6](06-observability.md)), uses migrations and has tested a restore ([Module 7](07-data-modeling.md)), deploys through CI with rollbacks and billing alerts ([Module 8](08-deployment.md)).

**Level 4: Owner.** Can explain the architecture, the threat model ([Module 1, Part 6](01-security-reality.md#part-6-threat-modeling-lite-stride-in-plain-language)), and the failure modes without opening the chat history. Reviews every change. Tracks debt and repays it, and knows when to bring in help.

**Which level you need depends on what you're building** (from Module 0):

| What you're building | Target level |
|---|---|
| Throwaway prototype | Level 0 |
| Personal tool | Level 1 |
| Internal tool | Level 2 (Level 3 if it can write to production data or move money) |
| Public app with real users | Level 3 |
| Money, health, identity, or kids' data | Level 4 |

**The course goal: Level 3 for anything with real users, Level 4 for anything that matters.**

### The Graduation Checklist

If you can check every box, you've graduated. Not from using AI, but from using it blindly.

**Security ([Module 1](01-security-reality.md) and [Module 2](02-security-vulnerabilities.md))**
- [ ] Authorization is enforced on the server (and in the database where applicable) for every piece of user data, and I have tests proving users can't access each other's data.
- [ ] No secrets in the repo, the client bundle, or logs. I know how to rotate each one.
- [ ] I verify every new dependency exists and is legitimate before installing it.

**Reading and reviewing ([Module 3](03-reading-code.md))**
- [ ] I review every AI diff before accepting, including the summary and deletions.
- [ ] I can trace any key request end to end.

**Testing ([Module 4](04-testing.md))**
- [ ] Critical logic has behavior tests with expected values from requirements.
- [ ] Tests run in CI on every push; breaking a critical line makes a test fail.

**Architecture ([Module 5](05-architecture.md))**
- [ ] My app has clear layers; rules that matter run on the server.
- [ ] I have a rules file the AI follows, and single sources of truth for key logic.

**Observability ([Module 6](06-observability.md))**
- [ ] No swallowed errors; users see friendly messages and I see details.
- [ ] Error tracking, structured logs (without secrets), health check, and alerts are live.

**Data ([Module 7](07-data-modeling.md))**
- [ ] Schema uses keys and constraints; changes go through migrations.
- [ ] I've restored a backup successfully in the last quarter.

**Deployment ([Module 8](08-deployment.md))**
- [ ] Environments are separated; agents can't touch production.
- [ ] Deploys go through CI/CD; I can roll back; billing alerts and rate limits exist.

**Debt (this module)**
- [ ] I keep a list of known debt with rough priorities, and I repay some of it every week or two.
- [ ] I know the signs of the spiral and what to do instead of prompting harder.
- [ ] I know when I need to bring in help, and who I'd call.

---

## What Good Looks Like

- Debt is visible: a `DEBT.md` or issue label listing known shortcuts, their impact, and a rough fix.
- Some capacity in every cycle (say, 10-20%) goes to repaying debt, prioritized by interest: the code you touch most and the risks that would hurt most.
- Refactors are small, tested, behavior-preserving, and reviewed; feature work and refactors are separate commits.
- When the same area breaks repeatedly, you stop patching and fix the structure.
- Rewrite decisions are made with written criteria and a migration plan, not frustration.
- You get expert help *before* high-stakes moments, not after an incident.

**Debt triage checklist (run monthly):**

- [ ] Which files changed most often in the last month? (`git log --format= --name-only --since="1 month ago" | sort | uniq -c | sort -rn | head`) Those are where debt interest is highest.
- [ ] Which bugs came back more than once? What structure caused them?
- [ ] Any security shortcuts still open? (Fix these first, always.)
- [ ] Any duplicated logic, dead code, or giant files added recently?
- [ ] Any area with no tests that changes frequently?
- [ ] Pick the top 1-3 items. Schedule them. Actually do them.

---

## Prompt Templates

**1. Debt inventory**

```text
Analyze this codebase and produce a technical debt inventory as a table:
item, location (files), category (duplication, dead code, missing tests,
wrong layer, giant file, swallowed errors, schema, security shortcut, deploy),
impact if left alone, estimated effort (S/M/L), and recommended order.
Security shortcuts first. Be specific; no generic advice. Don't change any code.
```

**2. Break the spiral**

```text
Stop. Don't write a fix yet. We've had repeated bugs in [area]. First, read
the code for [area] and list: every function involved, every duplicate or
conflicting implementation, and every special case. Then write out the
correct behavior as 10-15 plain-language rules, and ask me to confirm them
before writing tests.
```

**3. Characterization tests before refactoring**

```text
Before we refactor [module], write characterization tests that capture its
CURRENT behavior for typical inputs and edge cases, so we can detect any
behavior change during the refactor. Mark any behavior that looks like a bug
with a TODO comment, but assert the current behavior for now. Don't modify
the module.
```

**4. One safe refactoring step**

```text
Refactoring step: [one specific move, e.g., "consolidate formatDate helpers
into lib/dates.ts"]. Rules: do not change behavior; touch only the files
needed; keep the app working; run the tests and show the output; list every
file changed. If any test fails, stop and report instead of changing tests.
```

**5. Rewrite vs. refactor assessment**

```text
Assess whether [area/app] should be refactored or rewritten. Consider: whether
the data model is sound, how concentrated the problems are, test coverage,
security architecture, how many users and how much data depend on it, and
the effort for each path including data migration. Give a recommendation
with reasoning and a step-by-step plan for the recommended path, using the
strangler pattern if rewriting.
```

**6. Prepare for an expert review**

```text
Help me prepare this project for a review by an experienced engineer.
Produce: a one-page system overview (stack, architecture, data model, auth
approach, deploy process), a list of known issues and debt, the areas I'm
least confident about, and specific questions I should ask them. Base it
only on what's actually in the code; flag anything you're unsure about.
```

---

## Exercise

**Part A: Find your interest payments (20 minutes)**

1. Run the git command from the debt triage checklist to find your most frequently changed files.
2. For the top three, ask: why does this file change so often? Is it because the product evolves there (fine), or because it keeps breaking (debt)?
3. Run Prompt Template 1 and compare its findings with yours.

**Part B: Start a debt register (15 minutes)**

Create `DEBT.md` in your repo. List at least five items, each with: what it is, where, why it matters, and a rough fix. Put security items at the top. Commit it.

**Part C: One real refactor (45-60 minutes)**

Pick one item from your debt register that's in frequently changed code.

1. Write characterization tests (Prompt Template 3) if tests don't exist.
2. Do the refactor in at least three small steps (Prompt Template 4), committing after each with tests passing.
3. Review each diff yourself.
4. Write a short note in `DEBT.md` marking it done and what you learned.

**Part D: Find your rung (15 minutes)**

Go through the Graduation Checklist honestly. Count your checkmarks. Identify your current level on the ladder. Pick the **three unchecked boxes** that would reduce the most risk for your users, and schedule them for the next two weeks.

**Part E: Know who to call (10 minutes)**

Write down: under what circumstances you'd bring in an experienced engineer for your project, and one or two people or firms you'd contact. Decide now, while nothing is on fire.

---

## Key Takeaways

- Technical debt is a loan. Deliberate, documented debt can be smart; accidental debt compounds silently until you default.
- AI makes code nearly free to generate and no cheaper to understand. That's why AI-built apps accumulate debt so fast.
- Recognize the spiral: recurring bugs, growing fixes, contradictory changes, fear of certain files. When you see it, stop prompting.
- Break out by writing down correct behavior, turning it into tests, consolidating to a single source of truth, and refactoring in small steps with a fresh AI context.
- Refactoring means changing structure without changing behavior: tests first, one step at a time, never mixed with feature work.
- Prefer refactoring a working system. Rewrite only with clear criteria, real requirements, tests, and a data migration plan.
- Bring in an experienced engineer before high-stakes moments (payments, sensitive data, major migrations, rewrites) and whenever you're stuck or scared to deploy. Prepare well and learn from them.
- The goal of this course was never to make you stop using AI. It was to make you the person who's responsible for what the AI produces.

---

## Course Wrap-Up

You started this course as someone who could make an AI produce working software. You're finishing it as someone who can *own* software: read it, test it, structure it, observe it, protect its data, operate it in production, and keep it healthy over time.

That's the whole difference between vibe coding and engineering. Not a degree. Not avoiding AI. **Attention, verification, and ownership.**

The tools will keep getting faster. The people who pair that speed with careful judgment will build the things that last, and they'll be trusted with the things that matter. Be one of them.

Now go read that diff.

---

**Finished!** [Back to course outline](../course-outline.md)
