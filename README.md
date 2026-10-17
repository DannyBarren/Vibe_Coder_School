# Software Engineering for Vibe Coders: What You Don’t Know You Don’t Know

### The intervention for people who ship AI-generated code and then act surprised when it catches fire.

> You prompted.  
> The AI wrote the code.  
> You hit Accept All.  
> It mostly works.  
>  
> Congratulations — you’ve discovered the fastest way to ship a security incident.

You got something working in an afternoon. The demo looks good. Real users (or worse, real data) are now involved.

This course is for that moment.

The wording is a jab. It is somewhat meant to be. I have seen too many vibe coders cause serious harm — major security incidents that were entirely preventable if someone had taken a few minutes to do the obvious fix instead of moving on to the next prompt. Just because you *can* finish an entire app in a day and juggle twenty projects at once does not mean you *should*. Your stakeholders, your users, and the software itself deserve more respectful due diligence and careful attention than that.

AI has hypnotized a lot of people — including some already accomplished professionals — into lazy prompting and careless shipping. This course is the antidote. It is meant to help you be a better professional and deliver better, more secure, and longer-lasting products. If you avoid the temptation to treat “it mostly works” as done and actually do careful work, you will surpass almost everyone else. Attention to detail and due diligence matter more now than they ever have.

### A Question for Vibe Coders

You are using AI to do the work. After it finishes, the AI usually writes a long explanation of what it just did. Do you read it?

Just reading what the AI wrote will often show you some of the holes. If you are not going to read your own code, the absolute minimum is to read what the AI says it did. We will cover how to do that properly in this course.

### A Hard Truth

Vibe coding is not engineering.

Engineering is not a degree. It is not a course. It is a specific way of approaching a problem and a task — a method of thinking that abides by a set of first principles and a structured model of cognition used to create a solution.

Those first principles include:

1. **Understand the problem and its constraints** before you attempt a solution.
2. **Explicitly consider what can go wrong** — failure modes, edge cases, and adversarial conditions.
3. **Make trade-offs conscious** rather than accidental.
4. **Design so the solution can be verified and observed.**
5. **Account for the full lifecycle** — not just “it works once,” but maintainability, security, and operational reality.
6. **Validate assumptions with evidence**, not with hope or a successful demo.
7. **Maintain ownership and accountability** for the outcome.

Sending a prompt into an agent with no due diligence or analysis, letting the agent run, and then not reading the output *or* the explanation the agent just wrote, and shipping that as a product (more accurately, that product is called a liability) does not adhere to any of those first principles.

Therefore, pure vibe coding is not engineering. There, I proved it.

To be an engineer, you have to abide by those principles. That requires careful attention to detail. This course exists to help you develop that habit while still using AI tools.

---

## Who This Is For

- People who build with Cursor, Claude, Lovable, Replit, or whatever the current “just describe it” tool is
- Founders and operators who can prompt their way to a working prototype but are starting to feel the walls closing in
- Developers who use AI heavily and have noticed that “it mostly works” is a fragile place to live
- Anyone who has ever pasted a stack trace back into the chat and hoped for the best
- Professionals who want to keep the speed of AI tools without the careless shipping that creates real harm

**You do not need to stop using AI.**  
You do need to stop treating “the preview looks right” as a quality bar, and stop skipping the obvious fixes because you’re stretched thin or moving too fast.

## Who This Is Not For

- People who want another course on how to write better prompts so the AI will just handle everything
- Pure traditionalists who think AI coding tools are a moral failure
- Anyone looking for a gentle, non-judgmental introduction to “the future of software”
- People who believe finishing an app in a day automatically means it is ready for other people’s data or money

---

## What You Will Learn

The things that separate a demo that impresses people from software that doesn’t embarrass you, leak data, fall over, or become unmaintainable in three weeks.

Specifically:

- Why pure vibe coding is structurally unsafe, and the real incidents that prove it
- How to review AI-generated code instead of just accepting it
- The security problems that show up constantly — and the obvious fixes that too many people skip
- Testing that catches more than the happy path
- Enough architecture and structure that the next AI session doesn’t make things worse
- How to get something from “it runs on my machine” to something you can actually operate
- When to keep vibing and when to stop and do careful engineering work

---

## Course Structure

**Module 0: Welcome to the Intervention**  
A short, honest look at what vibe coding is, why it’s so effective for prototypes, and exactly where it starts costing you (and your stakeholders).

**Modules 1–2: Security — Why This Gets You Owned**  
The heaviest section of the course, on purpose. Real numbers, real incidents, and the specific ways AI-generated code fails (broken authorization, missing row-level security, leaked secrets, injection, insecure defaults, hallucinated dependencies). Then the practical defenses and the obvious fixes that careful people apply before they ship.

**Module 3: Reading the Code You Accepted**  
How to look at AI diffs without becoming a traditional software engineer. What to watch for.

**Module 4: Testing Beyond “Ask Claude for Tests”**  
Why AI-generated tests are often theater, and how to get tests that actually protect you.

**Module 5: Architecture for Prompt Engineers**  
Separation of concerns, modularity, and design decisions the AI will not make for you.

**Module 6: Error Handling & Observability**  
Logging that isn’t `console.log`. What happens when it fails in production instead of in your terminal.

**Module 7: Data Modeling & Persistence**  
Schemas, migrations, and why “just use JSON” has a shelf life.

**Module 8: Deployment, Environments & Production Reality**  
The gap between local and live. Config, environments, and basic operational hygiene.

**Module 9: Tech Debt, Refactoring & Knowing When to Stop Vibing**  
How to clean up AI spaghetti, and how to tell when you should stop prompting and start engineering (or call someone who does).

---

## How This Works

Each module is short, practical, and opinionated. You’ll get:

- The specific failure mode, explained without the usual corporate hedging
- Real examples of what goes wrong when careful attention is skipped
- Concrete things you can do (checklists, review habits, prompt patterns, small exercises)
- The difference between “the AI made it work” and “you can own this responsibly”

You will still use AI tools heavily. The difference is that you’ll use them with the lights on, and you’ll give the work the due diligence it deserves.

---

## Prerequisites

- Comfort using an AI coding tool (Cursor, Claude, ChatGPT, Lovable, etc.)
- Basic ability to run code and look at files
- A willingness to occasionally read what the AI wrote
- A willingness to do the careful work instead of the fastest work

No prior formal software engineering background required. That’s the point.

---

## A Note on Tone

This course is a jab. It is also useful.

The jab is somewhat deserved. Too many people have shipped serious harm by treating software engineering as optional once the AI made something look finished. The goal is not to make you feel bad about using AI. The goal is to stop you from shipping things you don’t understand or haven’t properly checked — especially once other people’s data or money is involved.

Vibe coding is a legitimate way to explore and prototype. It is a bad default for production systems. If you do the careful, professional work instead of the careless version, you will build better, more secure, longer-lasting products — and you will surpass the people who are still just accepting everything and moving on.

If that distinction makes sense to you, you’re in the right place.

---

*Software Engineering for Vibe Coders: What You Don’t Know You Don’t Know*  
The course for when “it mostly works” stops being good enough.

**This course may save you a lawsuit, a career-ruining security incident, and a catastrophe that causes you and/or a stakeholder serious harm. Use it.**
