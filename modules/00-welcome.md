# Module 0: Welcome to the Intervention

> *"I 'Accept All' always, I don't read the diffs anymore."*
> — Andrej Karpathy, in the [February 2025 post](https://x.com/karpathy/status/1886192184808149383) that named vibe coding. He also said it was "not too bad for throwaway weekend projects." Plenty of people skipped that part.

Hi. You're here because you shipped something, or you're about to, and some part of you suspects that "it works on the preview" isn't the same as "it works."

That part of you is right. This module is short. It explains what vibe coding is, what it's actually good at, the trade-off you make each time you hit Accept All, and how the rest of the course is laid out. There's no code yet. Think of it as the conversation before the intervention: everyone sits down, nobody's in trouble yet, and we'd like to talk about your choices.

---

## Learning Objectives

By the end of this module you will be able to:

1. Define vibe coding accurately, including what Karpathy said it was *for*.
2. Explain the **speed vs. ownership** trade-off and spot which side of it a given project sits on.
3. Say concretely why "pure Accept All" is dangerous once real users, real data, or real money are involved.
4. Describe the mindset this course asks for: keep the AI, add the engineering.
5. Do a two-minute honesty check on a project you've already shipped.
6. Place any project on a simple risk matrix and decide how much engineering it needs.
7. Use the course's five-step AI workflow: **plan → generate → read → test → verify**.

---

## Vibe Fail: The Weekend Launch *(illustrative composite, not a real event)*

Priya has a great idea for a booking app for dog groomers. On Saturday morning she opens an AI app builder and types: *"Build me a booking app for dog groomers with logins, a calendar, and Stripe payments."*

By Saturday night the app exists. It has a landing page with a cartoon corgi, a sign-up flow, a calendar that snaps nicely into place, and a checkout page. Each error she pastes back into the chat disappears. She never opens a code file. Why would she? It works.

On Sunday she posts it on social media. Forty groomers sign up and two hundred dog owners book appointments. Everyone loves the corgi.

On Wednesday a stranger emails her. He's polite, which is the scary kind. He opened his browser's developer tools, copied the database URL and public key out of the JavaScript, and ran a single query that returned every booking: names, phone numbers, home addresses, and the times each customer would be away from home at the groomer's. He also points out that he could have rewritten the prices.

Priya didn't do anything unusual. She did the default thing: she described what she wanted, accepted what came back, and shipped. The app did exactly what she asked for. She just never asked for "and only let people see their own bookings," and nobody (human or model) asked her who was allowed to see what.

The fix took about twenty minutes once someone explained it. Finding out that she needed it took a stranger and a lucky email.

This exact failure (a public key plus a database with no row-level access rules) has happened at scale to real apps. [Module 1](01-security-reality.md) has the receipts.

---

## Core Concepts

### 1. What vibe coding actually is

Andrej Karpathy (OpenAI co-founder, former head of AI at Tesla) coined the term on **February 2, 2025**: "a new kind of coding … where you fully give in to the vibes, embrace exponentials, and forget that the code even exists." In the same post he described accepting every change without reading the diff, pasting error messages back with no comment, and sometimes just asking for "random changes" until a bug went away. He finished with: "it's not really coding — I just see stuff, say stuff, run stuff, and copy paste stuff, and it mostly works." ([source](https://x.com/karpathy/status/1886192184808149383))

Note the important words: **"throwaway weekend projects"** and **"mostly works."** Karpathy was describing a fun, honest way to build disposable things. The internet heard "this is how software gets made now."

For this course, **pure vibe coding** means:

- You describe the outcome in natural language.
- The AI writes all of the code.
- You judge success by whether the screen looks right.
- You don't read, test, or understand the code before shipping it.

The first three are fine. The fourth is the problem.

### 2. Vibe coding is genuinely great at some things

Let's be fair before we get mean. Vibe coding is excellent for:

- **Prototypes and demos.** Ask "Is this idea any good?" and get an answer by Friday instead of next quarter.
- **Throwaway tools.** One-off scripts, internal toys, personal dashboards nobody else touches.
- **Learning by doing.** Seeing working code for a problem you care about is a decent way to start understanding it.
- **Exploring options.** Three versions of a UI in an hour, then you pick one.
- **Getting unstuck.** Boilerplate, config files, "how does this library want to be called."

If your project is one of those and nobody's data is at risk, vibe away. This course isn't against AI. The instructor uses it daily. It's against **shipping things nobody understands to people who trusted you.**

### 3. The speed vs. ownership trade-off

Every time you accept code you don't understand, you trade **ownership** for **speed**.

- **Speed** is how fast you get to "it runs."
- **Ownership** is your ability to explain the code, change it safely, fix it at 2 a.m., and answer the question "is our users' data safe?" with evidence instead of vibes.

On a prototype that trade is a bargain: you gain a lot of speed and lose ownership of something you'll throw away. In production it's a loan at a terrible interest rate. The bill shows up later as:

- A security incident you can't diagnose, because you don't know what the code does.
- A bug the AI "fixes" by creating two new ones, because neither of you understands the system.
- A feature that should take an hour but takes a week, because every change breaks something.
- A conversation with a customer, a regulator, or a lawyer in which "the AI wrote it" isn't a defense.

Here's the hard truth: **you can delegate the typing, but you can't delegate the responsibility.** Your name is on the app, and your users trusted *you*, not the model.

### 4. Why pure Accept All is dangerous (the short version)

Modules 1 and 2 cover this in depth. Here's the preview:

1. **AI code fails security checks a lot.** In Veracode's 2025 test of more than 100 models, the generated code introduced a known security flaw in **45%** of tasks, and newer, larger models weren't reliably better at security ([Veracode, 2025](https://www.veracode.com/blog/genai-code-security-report/)).
2. **People who use AI assistants feel more confident and write less secure code.** A Stanford user study found exactly that combination ([Perry et al., CCS 2023](https://arxiv.org/abs/2211.03622)).
3. **The model optimizes for "works," not "safe."** A demo that lets everyone read everything *works*. That's the problem.
4. **The model doesn't know your threat model.** It doesn't know your data is medical records, or that your users are teenagers, unless you tell it and then check.
5. **Agents can take real, destructive actions.** In July 2025 an AI agent deleted a founder's production database during an explicit code freeze ([The Register](https://www.theregister.com/software/2025/07/21/vibe-coding-service-replit-deleted-production-database/719783)).
6. **"It works" hides the failures you can't see.** Missing access rules, leaked keys, and unchecked inputs don't change what the preview looks like. They show up when someone who isn't you starts poking.

### 5. The mindset this course asks for

The [README](../README.md) lays out engineering's first principles. Here's the one-line version for vibe coders:

> **Use the AI for speed. Use your brain for judgment. Never ship what you haven't checked.**

In practice that means:

- **Read what the AI tells you it did.** At minimum, read its summary. It often admits the shortcuts it took ("for simplicity, I've disabled…").
- **Ask "what could go wrong?" before you ask "does it work?"**
- **Treat the AI as a fast, confident junior developer** who has never met your users, never been paged at 3 a.m., and never been sued.
- **Verify with evidence**, such as a test, a query run as a logged-out user, or a scan, not with a feeling.

### 6. When is vibe coding fine? A risk matrix

"Should I vibe-code this?" doesn't have one answer. It depends on two things: **who gets hurt if it breaks or leaks**, and **how hard it is to undo**. Find your project in this table:

| Project type | Examples | Who gets hurt if it fails? | Pure vibe coding? | Minimum bar before shipping |
|---|---|---|---|---|
| **Throwaway prototype** | A weekend demo, a pitch mock-up, a UI experiment | Nobody (maybe your ego) | ✅ Go for it | Don't put real data in it. Don't deploy it with real keys. |
| **Personal tool** | A script that renames your files, a habit tracker only you use | You | ✅ Mostly fine | Back up anything it touches. Keep keys out of public repos. |
| **Internal tool** | A dashboard for your team, an ops script, an admin helper | Your colleagues, your company's data | ⚠️ Prototype yes, ship no | Auth on everything, no production write access without review, someone else reads the code. |
| **Public app with real users** | Anything strangers sign up for | Users who trusted you with their accounts and data | ❌ Not without review | Modules 1–2 pre-launch checklist, tested access rules, secret scanning, backups. |
| **Money, health, identity, or kids' data** | Payments, medical notes, ID verification, anything used by minors | People who may face fraud, stalking, discrimination, or legal harm | ❌ Absolutely not | All of the above, *plus* an experienced human reviewer, logging and alerting, an incident plan, and legal and compliance awareness. |

**Quick self-assessment.** Answer these about your current project. Each "yes" moves you one row down the table:

1. Will anyone other than me use it?
2. Will it store anything about a person (email, name, location, photos, messages)?
3. Can it spend money, move money, or send messages on someone's behalf?
4. Would a leak be embarrassing, harmful, or reportable for someone other than me?
5. Would I be in trouble (with a customer, a boss, or a regulator) if it went down for a day or lost its data?

**0 yes:** vibe freely. **1–2:** vibe the prototype, then apply Modules 1–4 before real users arrive. **3+:** treat this as an engineering project that happens to use AI, not an AI project that happens to touch people.

The point isn't that vibe coding is bad. The point is that **the bar rises with the stakes**, and most of the incidents in Module 1 happened because someone shipped a top-row process into a bottom-row situation.

### 7. How to use AI in this course: plan → generate → read → test → verify

You'll use AI on every exercise in this course. You'll just use it with a loop instead of a slot machine. Here's the five-step workflow every later module builds on:

| Step | You do | The AI does | Output |
|---|---|---|---|
| **1. Plan** | State the goal, users, sensitive data, and constraints. Decide what "done" means. | Asks clarifying questions, proposes an approach, lists risks. | A short written plan you agree with *before* code exists. |
| **2. Generate** | Ask for one small, well-scoped change at a time. | Writes the code. | A diff you can actually read (dozens of lines, not thousands). |
| **3. Read** | Read the AI's summary and skim the diff for red flags (Module 3). | Explains anything you ask about, including its shortcuts. | You can explain what changed in two sentences. |
| **4. Test** | Run it, including the unhappy paths: wrong user, bad input, logged out. | Writes tests you've checked are real (Module 4). | Tests that would fail if the feature broke. |
| **5. Verify** | Check with evidence outside the AI: curl as a stranger, scanners, logs. | Suggests what to verify and how. | Proof, not vibes. |

Then commit, and repeat. Small loops beat giant prompts: when something goes wrong in a 40-line change you can find it, and in a 4,000-line change you can't.

**A planning prompt to start any feature:**

```text
I want to build: [feature].
Users: [who]. Sensitive data involved: [what, or "none"].
Before writing any code:
1. Ask me up to 5 clarifying questions.
2. Propose the simplest design that works, in plain English.
3. List what could go wrong: security, data loss, cost, and edge cases.
4. Break the work into small steps I can review one at a time.
Do not write code until I approve the plan.
```

Notice what that prompt does: it gets the AI to do step 1 *with* you instead of skipping straight to step 2. The Stanford researchers found that people who gave the AI clearer specifications, and who edited and questioned its output, ended up with more secure code ([Perry et al.](https://arxiv.org/abs/2211.03622)). Planning is how you give it those specifications.

---

## What Good Looks Like: The Vibe-Coder Maturity Ladder

| Level | You… | Fine for… |
|---|---|---|
| 0. Pure vibes | Accept All, ship, pray | Throwaway toys with no users or data |
| 1. Reader | Read the AI's summary and skim the diff | Personal tools |
| 2. Questioner | Ask the AI to explain risky parts, and push back | Internal tools with low-risk data |
| 3. Verifier | Test as a logged-out or other user, run scanners, check access rules | Small public apps |
| 4. Owner | Can explain the architecture, threat model, and failure modes; reviews every change | Anything handling other people's data or money |

The goal of this course is to get you to Level 3 quickly and to Level 4 for anything that matters. You'll still prompt all day. You'll just stop doing it with your eyes closed.

### A prompt habit to start today

Paste this after any significant AI change:

```text
Before I accept this change, give me:
1. A plain-English summary of what you changed and why.
2. Every shortcut, assumption, or "for simplicity" decision you made.
3. Anything you disabled, mocked, hard-coded, or left as a TODO.
4. Who can access any data this code touches (logged out? any user? only the owner?).
5. What would break, or leak, if a malicious user called this directly.
Be blunt. Do not reassure me.
```

It won't catch everything, because the model can be confidently wrong about its own code. It will catch a surprising amount, and it trains *you* to ask the right questions.

---

## Glossary: Words This Course Uses

You don't need to memorize these. Come back when a module uses one and it doesn't click.

| Term | Plain-English meaning |
|---|---|
| **Vibe coding** | Building software by describing it to an AI and accepting the output without reading or understanding it (Karpathy, Feb 2025). |
| **Accept All** | Applying every change the AI proposes without reviewing the diff. |
| **Diff** | The before-and-after view of what a change adds and removes in your code. |
| **Frontend / client** | The code that runs on the user's device (browser or phone). Users can read and modify it. |
| **Backend / server** | The code that runs on machines you control. This is where real rules must be enforced. |
| **API** | The set of URLs your frontend (or anyone with `curl`) calls to read and change data. |
| **Authentication (authn)** | Proving *who* someone is, such as logging in. |
| **Authorization (authz)** | Deciding *what* that person is allowed to do, such as "only see your own invoices." |
| **RLS (Row Level Security)** | Database rules (Postgres and Supabase) that decide which rows each user can read or change. |
| **Security rules** | Firebase's equivalent of RLS for Firestore and Cloud Storage. |
| **Secret / API key** | A credential that grants access to a service. Some keys are public by design; secret keys must never reach the browser. |
| **Environment variable** | A setting (often a secret) provided to your app at runtime instead of hard-coded in the source. |
| **Injection** | When user input gets treated as code: SQL, shell commands, HTML, or LLM instructions. |
| **XSS** | Cross-site scripting: an attacker's JavaScript running in another user's browser. |
| **Dependency / package** | Someone else's code your app installs (from npm, PyPI, etc.). |
| **Hallucination** | The AI confidently stating something false, such as a package that doesn't exist or code that "is secure." |
| **Agent** | An AI that takes actions (edits files, runs commands, calls tools) rather than just suggesting text. |
| **MCP** | Model Context Protocol: a standard way to connect AI agents to tools and data sources (databases, GitHub, etc.). |
| **Production (prod)** | The live system real users depend on. Its opposite is development (dev) or staging. |
| **Threat model** | A written answer to "what are we protecting, from whom, and what could go wrong?" |
| **Least privilege** | Giving every person, key, and agent only the minimum access it needs. |
| **Tech debt** | Shortcuts that make today faster and every future change slower. |

---

## How the Course Is Organized

- **Module 0 (you are here):** the honest intro.
- **Modules 1–2: the security deep dive.** This is the heaviest part of the course on purpose. Module 1 covers why vibe-coded apps are so often vulnerable, with real numbers and real incidents. Module 2 covers the specific vulnerabilities that get you hacked and the defenses and prompts that prevent them.
- **Modules 3–9: the rest of engineering.** Reading code you accepted, testing, architecture, observability, data modeling, deployment, and knowing when to stop vibing.

### Course roadmap

Think of the course as three stages, each building on the last:

| Stage | Modules | The question it answers | You'll leave with |
|---|---|---|---|
| **1. Stop the bleeding** | 0, 1, 2 | "Is my app leaking data or about to?" | A stranger audit, fixed access rules, a secrets scan, and a pre-launch checklist with evidence |
| **2. Understand what you have** | 3, 4, 5 | "Do I actually know what this code does, and will I notice when it breaks?" | A diff-reading habit, real tests, and a structure the AI can't easily wreck |
| **3. Run it like you mean it** | 6, 7, 8, 9 | "Can I operate, change, and grow this without fear?" | Logging and alerts, sane data and migrations, dev/prod separation, and a plan for tech debt |

If your app already has real users, do Stage 1 *this week*, before anything else.

See the full [course outline](../course-outline.md).

Each module has learning objectives, a "Vibe Fail" story, core concepts, checklists and prompt templates, an exercise, and key takeaways. Do the exercises. Reading about security and actually checking your app are very different experiences, as you're about to find out.

---

## Exercise: The Two-Minute Honesty Check

Pick one app you've built mostly with AI. If you haven't shipped one yet, use the one you're building. Answer each question **yes**, **no**, or **I don't know**. Write the answers down.

1. Can I explain, in two sentences, where this app stores its data?
2. If I log out and call the app's API or database directly, can I read anyone's data?
3. Are there any API keys, passwords, or tokens in the code, the repo history, or the browser bundle?
4. If the AI agent ran a destructive command right now, could it touch production data?
5. If this broke at 2 a.m., would I know before a user told me?
6. Could I add a feature next month without being afraid of breaking three others?

**Scoring:** every "I don't know" counts as a "no." That's not cynicism. An attacker doesn't care whether you knew.

**Part 2 (3 minutes).** Place the same app on the risk matrix from Core Concept 6. Write down its row, and the row it'll be in six months from now if it succeeds. Most people discover their "internal tool" is one sign-up page away from being a public app.

**Part 3 (5 minutes).** Pick the next feature you plan to build and run the planning prompt from Core Concept 7 on it *before* generating any code. Compare the risks the AI lists with what you'd have thought of on your own.

If you have more than one "no," you're the target audience, and Modules 1 and 2 are about to be very uncomfortable and very useful. Keep the list. You'll revisit it at the end of Module 2, and again at the end of the course.

---

## Key Takeaways

- **Vibe coding** (Karpathy, Feb 2025) means giving in to the vibes and forgetting the code exists. Its inventor pitched it for throwaway weekend projects.
- It's **excellent for prototypes** and **a bad default for production**, because it gives up ownership in exchange for speed.
- **You can delegate the typing, not the responsibility.** "The AI wrote it" isn't a defense.
- Pure Accept All is dangerous because AI code fails security tests often, users overtrust it, the model optimizes for "works," it doesn't know your threat model, and agents can take destructive actions.
- The fix isn't to quit AI. It's **AI for speed, humans for judgment, evidence before shipping.**
- **The bar rises with the stakes.** Prototypes can be pure vibes. Real users, money, health, identity, or kids' data cannot.
- Use the loop on every change: **plan → generate → read → test → verify**, in small steps.
- Start today: **read what the AI says it did**, and ask it to list its shortcuts.

Next: [Module 1: Why Your Vibe-Coded App Is Probably Vulnerable →](01-security-reality.md)
