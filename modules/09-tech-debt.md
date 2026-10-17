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

*(Illustrative story. Ask anyone who has vibe coded past the prototype stage; they'll have their own version.)*

Alex's habit-tracking app had a bug: streaks reset at midnight UTC instead of the user's local midnight. Alex pasted the bug into the chat. The AI fixed it.

The fix changed a date helper used in four places. Now the weekly summary email counted the wrong days. Alex pasted that bug in. The AI fixed it by adding a *second* date helper just for emails. Now there were two helpers that disagreed. The calendar view used one, the streak logic used the other, and users saw a streak of 12 next to a calendar showing 11.

Alex pasted that in. The AI "unified" them, which reintroduced the original midnight bug, plus a new one in the export feature. Alex asked the AI to "fix all the date bugs." It rewrote 600 lines across 14 files. The app now crashed for users in time zones with half-hour offsets.

Six weeks, roughly forty prompts, and one very long chat history later, the date logic was worse than when Alex started. Every prompt took longer. The AI's context window couldn't hold the relevant code anymore, so each fix was made without seeing the whole picture. There were no tests, so nobody (human or AI) could tell which fixes had broken what.

Alex didn't need a better prompt. Alex needed to **stop prompting**, write ten tests that described correct date behavior, consolidate to one date module, and refactor in small, verified steps. That took two afternoons.

---

## Core Concepts

### 1. Technical debt: a loan with interest

Ward Cunningham introduced the debt metaphor in 1992: "Shipping first time code is like going into debt. A little debt speeds development so long as it is paid back promptly with a rewrite" ([Cunningham, OOPSLA '92](http://c2.com/doc/oopsla92.html)). He also warned that the danger comes when the debt is not repaid: every minute spent on not-quite-right code counts as interest.

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

Pure vibe coding generates accidental debt at industrial scale. GitClear's analysis of 211 million changed lines found that in 2024, copy/pasted code exceeded "moved" code (a signal of refactoring) for the first time in their data, while duplicated blocks rose roughly eightfold ([GitClear 2025](https://www.gitclear.com/ai_assistant_code_quality_2025_research/)). More code is being written; less is being consolidated. That's a lot of loans and not many repayments.

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

### 6. Graduation: from vibe coder to AI-assisted engineer

Remember the README's first principles? Understand the problem, consider what can go wrong, make trade-offs consciously, design for verification and observation, account for the full lifecycle, validate assumptions with evidence, and own the outcome. Every module in this course was one of those principles turned into habits.

Here's the ladder. Be honest about where you are. Everyone starts at the bottom.

**Level 0: Pure Vibe Coder**
Prompt → Accept All → ship. Doesn't read diffs or the AI's summary. No tests, no git habits, secrets wherever they landed. "It works on the preview." *(This is the liability the README warned about.)*

**Level 1: Vibe Coder With a Conscience**
Reads the AI's summary and skims diffs. Commits before AI sessions. Knows secrets don't go in the repo. Has separate local and production environments. Has heard of RLS and checked whether it's on.

**Level 2: AI-Assisted Builder**
Reviews every diff with a checklist ([Module 3](03-reading-code.md)). Writes tests from requirements, including auth tests ([Module 4](04-testing.md)). Has a rules file the AI follows ([Module 5](05-architecture.md)). Has error tracking and structured logs ([Module 6](06-observability.md)). Uses migrations, constraints, and has tested a restore ([Module 7](07-data-modeling.md)). Deploys through CI with rollbacks and billing alerts ([Module 8](08-deployment.md)).

**Level 3: AI-Assisted Engineer**
Starts with the problem and constraints, not the prompt. Threat-models features before building them ([Module 2](02-security-vulnerabilities.md)). Designs data and architecture deliberately, then uses AI to implement within those constraints. Tracks debt consciously and repays it. Debugs from evidence. Can explain every important part of the system without opening the chat history. Knows the limits of their knowledge and gets help at the right moments. Uses AI for speed, never as a substitute for judgment.

**Level 4: Engineer Who Happens to Use AI**
All of the above is automatic. Sets standards others follow. Reviews other people's AI-generated code. Mentors. Owns systems and outcomes, including the 3 a.m. pages.

You don't need to reach Level 4 to ship responsibly. **For anything that handles other people's data or money, Level 2 is the minimum, and Level 3 is the goal.**

### The Graduation Checklist

If you can check every box, you've graduated. Not from using AI, but from using it blindly.

**Security ([Modules 1-2](02-security-vulnerabilities.md))**
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

**Back to:** [Course Outline](../course-outline.md)
