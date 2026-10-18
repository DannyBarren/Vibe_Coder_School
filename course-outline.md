# Course Outline

## Software Engineering for Vibe Coders: What You Don't Know You Don't Know

*The intervention for people who ship AI-generated code and then act surprised when it catches fire.*

### The Pitch

AI tools can take you from idea to working app in an afternoon, and that's great. The trouble starts when "it works on the preview" becomes the only quality bar, and an app full of other people's data goes live without anyone having read the code, checked who can access what, or asked what happens when it breaks. This course teaches the engineering you skipped: the security fundamentals that keep you out of the headlines, and the reading, testing, architecture, observability, data, and deployment habits that turn a demo into something you can actually own. It's a jab at pure Accept-All vibe coding, but a helpful one. **You keep using AI. You just stop using it blindly.**

### Who It's For

- People building with Cursor, Claude, Lovable, Replit, Bolt, Base44, or whatever the current "just describe it" tool is.
- Founders and operators who can prompt their way to a working prototype and are about to let real users in.
- Developers who lean on AI heavily and have noticed that "it mostly works" is a fragile place to live.
- Anyone who has ever pasted a stack trace back into the chat and hoped for the best.
- Team leads who need a shared standard for how their people use AI coding tools.

No formal software engineering background is required. You need an AI coding tool, the ability to run code and open files, and a willingness to occasionally read what the AI wrote.

### How to Use This Course

1. **Start with Module 0**, then do **Modules 1 and 2** before anything else. They're the core of the course, and they have the most immediate payoff: they help you find holes that real attackers find in real apps.
2. **Work through Modules 3–9 in order** if you're building something long-lived, or jump to the module that matches your current pain. The [learning paths](#suggested-learning-paths) below suggest orders for common situations.
3. **Use the five-step loop on every exercise:** plan → generate → read → test → verify (introduced in Module 0). Module 0 also defines the **Vibe-Coder Maturity Ladder** (Levels 0–4) the rest of the course refers back to, so read that section even if you skim the rest.
4. **Do the exercises on your own project.** Each module ends with a hands-on task. Reading about security and actually curling your own database as a logged-out stranger are very different experiences.
5. **Steal the prompt templates.** Every module includes copy-paste prompts that make the AI do the careful version of the work, and checklists that tell you how to verify it did.
6. **Keep your notes.** Module 0's honesty check and Module 1's stranger audit come back later. Watch your "I don't know" answers turn into "yes, here's the evidence."

Every module includes **learning objectives**, a short **"Vibe Fail"** story, **core concepts**, **practical checklists and prompt templates**, an **exercise**, and **key takeaways**. The security modules cite their sources inline and in a Sources section.

---

## Modules

| # | Module | What you'll learn | Approx. time |
|---|---|---|---|
| 0 | [Welcome to the Intervention](modules/00-welcome.md) | What vibe coding actually is (Karpathy's term, and what he said it was for), the speed vs. ownership trade-off, a risk matrix for when pure vibe coding is fine and when it isn't, the course's plan → generate → read → test → verify workflow, a side-by-side of the same feature built Accept-All vs. with review, the **Vibe-Coder Maturity Ladder** (defined here and used by the whole course), and a glossary. Ends with a quick honesty check on your own app. | ~40 min (25 min read + exercise) |
| 1 | [Why Your Vibe-Coded App Is Probably Vulnerable](modules/01-security-reality.md) | **Security deep dive, part 1.** The research (Veracode found AI code introduced security flaws in 45% of tests; Stanford found AI users wrote less secure code and felt more confident), real incidents (Lovable, Moltbook, Tea, Base44, Replit, Google Antigravity) with two full incident timelines, the OWASP Top 10:2025 translated for vibe coders, seven structural reasons AI code is insecure, agent and MCP risks, and a plain-language STRIDE threat-model worksheet. Includes a hands-on “look at your app the way an attacker does” walkthrough and a sourced cost-of-a-breach section (IBM 2025, Tea lawsuits). | 1.5–2.5 hrs (~50 min read + the attacker walkthrough and stranger audit; longer if you’re fixing as you go) |
| 2 | [The Actual Vulnerabilities That Get You Hacked](modules/02-security-vulnerabilities.md) | **Security deep dive, part 2.** Broken authorization and RLS, authentication pitfalls, mass assignment, secrets, injection (including prompt injection), SSRF, file uploads, webhook signatures, security headers, insecure defaults, and hallucinated dependencies, each with bad vs. good code, checklists, and prompts. Each vulnerability class has a “try it yourself” mini-lab (vulnerable snippet → exploit request → fix → verification), with examples across Express, Next.js route handlers and server actions, FastAPI, and Supabase. Then a curl-based attacker test script, a severity triage guide, an AI security-review prompt pack, and the pre-launch checklist. | 4–7 hrs, best split over several sessions (~85 min read + extensive hands-on labs) |
| 3 | [Reading the Code You Accepted](modules/03-reading-code.md) | How to review AI diffs and AI summaries without becoming a full-time engineer: what to look at first, the red flags that matter, and how to make the AI explain itself honestly. | 1–1.5 hrs (~25 min read + exercise) |
| 4 | [Testing Beyond "Ask Claude for Tests"](modules/04-testing.md) | Why AI-generated tests are often theater (mocked everything, asserting nothing), and how to get tests that actually catch regressions, edge cases, and the unhappy paths attackers love. | 1.5–2 hrs (~25 min read + exercise) |
| 5 | [Architecture for Prompt Engineers](modules/05-architecture.md) | Separation of concerns, modularity, and the design decisions the AI won't make for you, so the next AI session improves your codebase instead of making it worse. | 1.5–2 hrs (~25 min read + exercise) |
| 6 | [Error Handling & Observability](modules/06-observability.md) | Logging that isn't `console.log`, errors that don't fail silently, and knowing your app is broken before your users tell you. | 1.5–2 hrs (~26 min read + exercise) |
| 7 | [Data Modeling & Persistence](modules/07-data-modeling.md) | Schemas, constraints, migrations, and backups, and why "just store it as JSON" has a shelf life. Your data outlives your code. | 1.5–2.5 hrs (~32 min read + exercise) |
| 8 | [Deployment, Environments & Production Reality](modules/08-deployment.md) | The gap between "runs on my machine" and "runs in production": environments, configuration, dev/prod separation, rollbacks, and basic operational hygiene. | 1.5–2 hrs (~29 min read + exercise) |
| 9 | [Tech Debt, Refactoring & Knowing When to Stop Vibing](modules/09-tech-debt.md) | How to clean up AI spaghetti safely, and how to tell when to stop prompting and start engineering, or call someone who does. | 1.5–2 hrs (~32 min read + exercise) |

*Times are estimates: reading at roughly 200 words per minute for dense technical text, plus doing that module's exercise on a real project. Your first pass through Modules 1–2 on an app with real issues will take longer, because you'll be fixing things, which is the point. Total: roughly 18–27 hours.*

> **Note: Modules 1 and 2 are the core security deep dive.** They're the heaviest part of the course on purpose. Every incident in them was preventable, and almost none needed a sophisticated attacker. If you only have time for two modules, make it these two, and run the pre-launch checklist at the end of Module 2 before your app touches real users.

---

## Suggested Learning Paths

Pick the path that matches your situation. Every path starts with the security deep dive, because that's where the most expensive mistakes live.

### 🚨 "My app is live and I'm worried" (one focused day)
1. **Module 0**, honesty check only (15 min).
2. **Module 1**, Part 2 (incidents), Part 5 (agents), and the 30-minute stranger audit.
3. **Module 2**, Part 5: run the attacker test script. Then Part 6: triage. Fix every P0 today, including rotating any leaked keys.
4. **Module 2**, Part 8: work through the pre-launch checklist, and schedule the P1s this week.
5. Then come back for the full course.

### 🚀 "I'm about to launch" (one to two weeks, a few hours a day)
1. Modules **0 → 1 → 2** in full, including every exercise.
2. **Module 4** (testing): turn your attacker tests into automated tests.
3. **Module 8** (deployment): separate dev and prod, set up backups and rollbacks.
4. **Module 6** (observability): know about failures before your users do.
5. Run Module 2's pre-launch checklist as your go/no-go.

### 🏗️ "I'm building something long-lived" (the full course, about a module a week)
Modules **0 through 9 in order.** Apply each module to your real project before moving on. Revisit Module 2's checklist before every major release.

### 👥 "I lead a team that uses AI tools"
Modules **0, 1, 2, 3, and 9** first. Then adopt Module 1's agent-safety checklist, Module 2's prompt pack and pre-launch checklist, and Module 3's review habits as **team policy**, not personal preference.

---

*See the [README](README.md) for the full philosophy behind the course, including the engineering first principles it's built on.*
