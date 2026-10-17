# Module 1: Why Your Vibe-Coded App Is Probably Vulnerable

> **Security Deep Dive, Part 1 of 2.** This module and the next are the core of the course. If you only finish two modules, make it these two.

You built something that works. This module is about the gap between **"it works"** and **"it's safe,"** and why that gap is the *default* outcome of vibe coding, not bad luck.

We'll start with numbers from real research, move on to real incidents, and then look at the structural reasons AI-generated code tends to be insecure. By the end you should be appropriately nervous, and you'll know exactly where to look. [Module 2](02-security-vulnerabilities.md) then turns that nervousness into specific fixes.

A note on sources: every statistic and incident here links to its source, and there's a full list at the end. Stories marked *illustrative* are composites written to teach a point. Everything else happened.

---

## Learning Objectives

By the end of this module you will be able to:

1. Cite the main research on AI-generated code security and describe what it does and doesn't show.
2. Describe at least five real incidents involving AI-built or AI-operated software and name the root cause of each.
3. Explain seven structural reasons AI-generated code tends to be insecure, so you can predict where *your* app is weak.
4. Explain why "it works" and "it's secure" are different properties that need different evidence.
5. Run a 30-minute exposure audit on your own app, acting as a logged-out stranger.

---

## Vibe Fail: "But Nobody Knows the URL" *(illustrative composite, not a real event)*

Marcus runs a small fitness-coaching business. He vibe-codes a client portal over two evenings: clients log in, see their workout plans, upload progress photos, and message him. The AI wires it all up with a hosted database and file storage. Everything works. Clients love it.

Marcus has one security thought during the whole build: *"Nobody knows the database URL, so it's fine."*

Here's what he didn't know:

- **The database URL and key ship to every visitor's browser.** That's how the frontend talks to the database. Anyone can press F12 and read them.
- **The database had no row-level access rules.** The key could read every table, including other clients' messages and photos.
- **The storage bucket was public.** Progress photos, some of them shirtless "before" pictures, were reachable by anyone who could list or guess file paths.
- **A third-party API key** for his email service was hard-coded in the frontend, "for simplicity," as the AI's summary had noted. He didn't read the summary.

Nobody had to hack anything. A curious person with developer tools could have opened all of it in about ten minutes. "Nobody knows the URL" isn't a security model. It's a hope, and it lasts exactly as long as nobody looks.

The rest of this module shows that Marcus's portal isn't a far-fetched example. It's roughly the median.

---

## Part 1: The Numbers

### 1.1 Veracode: 45% of AI-generated code introduced a security flaw

In July 2025, Veracode published its **2025 GenAI Code Security Report**. The researchers gave **more than 100 large language models** a set of **80 coding tasks** in Java, JavaScript, Python, and C#. Each task could be completed either securely or insecurely, and each was chosen because it maps to a known weakness category (CWE). They then scanned the results with static analysis. ([Veracode blog](https://www.veracode.com/blog/genai-code-security-report/), [press release](https://www.businesswire.com/news/home/20250730694951/en/AI-Generated-Code-Poses-Major-Security-Risks-in-Nearly-Half-of-All-Development-Tasks-Veracode-Research-Reveals))

What they found:

- In **45% of tests**, the model chose the insecure option and introduced a vulnerability in the **OWASP Top 10**, the industry's list of the most critical web application risks.
- **Java** was the worst, with a **72%** security failure rate. **Python** failed **38%** of the time, **JavaScript 43%**, and **C# 45%**.
- Models failed to defend against **cross-site scripting (XSS)** in **86%** of relevant cases, and against **log injection** in **88%**.
- **Newer and bigger didn't mean more secure.** Models got much better at writing code that *compiles and runs*, but their security pass rate stayed roughly **flat** across release dates and model sizes.

That last point matters a lot. The comforting story is "the models will get better, so this solves itself." Veracode's data says the models improved at *working* code while security stood still.

**The October 2025 update** tested newer models on the same benchmark. Most still landed between **50% and 59%** secure. The best performers (OpenAI's GPT-5 reasoning models) reached about **70–72%**, which still means insecure choices in **roughly 28–30%** of cases. Veracode's conclusion: organizations "cannot rely on AI-generated code to be secure by default." ([Veracode October update](https://www.veracode.com/blog/ai-code-security-october-update/), [SC Media](https://www.scworld.com/news/openais-gpt-5-generates-more-secure-code-than-past-models-report-finds))

**What this does and doesn't show.** It's a benchmark of single code-completion tasks, not a scan of full production apps, and it covers four vulnerability types. It doesn't say 45% of your *app* is broken. It says that **when the AI faces a choice between a safe and an unsafe way to write something, it picks the unsafe one often enough that you should assume every security-relevant line needs checking.** That's the right mental model.

### 1.2 Stanford: AI users wrote less secure code *and felt more confident*

In the peer-reviewed study **"Do Users Write More Insecure Code with AI Assistants?"** (Perry, Srivastava, Kumar, and Boneh; ACM CCS 2023), Stanford researchers gave **47 participants** security-related programming tasks in Python, JavaScript, and C. Some had an AI assistant and some didn't. ([arXiv](https://arxiv.org/abs/2211.03622), [ACM](https://dl.acm.org/doi/10.1145/3576915.3623157))

- Participants **with** the assistant wrote **less secure code on four of the five tasks**.
- On the SQL task, **36%** of the AI group wrote code vulnerable to **SQL injection**, compared with **7%** of the control group.
- On the encryption task, the AI group was significantly more likely to produce insecure solutions, including trivial ciphers.
- The kicker: participants **with AI access were more likely to believe their code was secure.**

The study used an older model (OpenAI's Codex), and most participants were university students, which the authors flag as limitations. The *human* finding is the one that still matters: **AI help made people more confident and less careful at the same time.** That's the pure-vibe-coding failure in one sentence.

The study had a hopeful note too. Participants who gave the AI more context, wrote clearer specifications, and **edited and questioned** its output rather than accepting it wholesale produced more secure code. Engaged use beats Accept All. That's the whole thesis of this course, backed by data.

### 1.3 Escape.tech: 5,600 vibe-coded apps, 2,000+ high-impact vulnerabilities

In late 2025 the security company Escape scanned **more than 5,600 publicly available apps** built on vibe-coding platforms (mostly Lovable, plus Base44, Create.xyz, Bolt.new, and others). Using deliberately **passive, non-destructive** scans, they found:

- **More than 2,000 high-impact vulnerabilities.**
- **More than 400 exposed secrets**, such as API keys and tokens, including Supabase service-role keys sitting in frontend JavaScript.
- **175 instances of exposed personal data**, including medical records, IBANs (bank account numbers), phone numbers, and emails.

([Escape methodology](https://escape.tech/blog/methodology-how-we-discovered-vulnerabilities-apps-built-with-vibe-coding/), [full report](https://escape.tech/state-of-security-of-vibe-coded-apps))

Two of their observations should stick with you:

1. **Most of the vulnerabilities were reachable without logging in.** Nobody needed to crack a password. The data was simply available.
2. **These numbers are a lower bound.** They avoided aggressive testing for ethical reasons. A real attacker wouldn't.

### 1.4 GitGuardian: secrets leak constantly, and stay valid

GitGuardian's **State of Secrets Sprawl 2025** analyzed public GitHub activity for 2024 ([report](https://www.gitguardian.com/state-of-secrets-sprawl-report-2025), [summary](https://blog.gitguardian.com/the-state-of-secrets-sprawl-2025/)):

- **23.8 million** new hard-coded secrets were leaked on public GitHub in 2024, a **25%** increase over the previous year.
- **4.6%** of public repositories contained a secret, and so did **35%** of the *private* repositories GitGuardian scanned for customers. "It's a private repo" isn't a strategy.
- **70% of secrets leaked in 2022 were still valid** when the report was written. People leak keys and then never rotate them.
- In GitGuardian's sample of public repositories where GitHub Copilot was enabled, **6.4%** leaked at least one secret, compared with 4.6% across all public repos. That's about **40% higher**. This is a correlation, not proof that Copilot causes leaks, but it lines up with everything else here: faster code generation, less attention, more leaked keys. ([CSO Online coverage](https://www.csoonline.com/article/3953927/ai-programming-copilots-are-worsening-code-security-and-leaking-more-secrets.html))

### 1.5 Package hallucinations: the AI invents dependencies

In a study that won a Distinguished Paper award at **USENIX Security 2025**, researchers from UT San Antonio, the University of Oklahoma, and Virginia Tech had **16 code-generating LLMs** produce **576,000 code samples** in Python and JavaScript. ([USENIX](https://www.usenix.org/conference/usenixsecurity25/presentation/spracklen), [paper PDF](https://www.usenix.org/system/files/usenixsecurity25-spracklen.pdf))

- Of **2.23 million** package references in that code, **440,445 (19.7%)** pointed to packages that **don't exist**.
- That included **205,474 unique made-up package names.**
- Commercial models hallucinated packages at an average of **at least 5.2%**; open-source models at **21.7%**.

Why does that matter? Because an attacker can *register* a name the AI keeps inventing and fill it with malware. That attack is now called **slopsquatting** (Module 2 covers it). It's not hypothetical. In a 2023–24 experiment, security researcher Bar Lanyado registered a package name that AI models kept hallucinating (`huggingface-cli`), as an empty, harmless package. It got **more than 30,000 downloads in three months**, and the fake install command even turned up in the README of a real Alibaba repository. ([The Register](https://www.theregister.com/security/2024/03/28/ai-bots-hallucinate-software-packages-and-devs-download-them/858011))

### 1.6 What the industry lists as the top risks

The **OWASP Top 10:2025**, the standard reference for web application risk, puts these at the top ([OWASP](https://owasp.org/Top10/2025/)):

1. **A01 Broken Access Control**
2. **A02 Security Misconfiguration**
3. **A03 Software Supply Chain Failures**
4. A04 Cryptographic Failures
5. **A05 Injection**

Hold that list next to the incidents below. Missing row-level security is broken access control. Public storage buckets and debug mode are misconfiguration. Hallucinated packages are a supply-chain failure. Unparameterized queries are injection. **Vibe-coded apps don't invent new vulnerabilities. They mass-produce the old ones.**

---

## Part 2: The Incidents

### 2.1 Lovable and CVE-2025-48757: 170 apps with the doors open

**What happened.** On **March 20, 2025**, Matt Palmer was looking at a site built on Lovable, a popular AI app builder. By modifying a single network request, he got **every row in the site's `users` table**. Lovable apps talk to a Supabase (Postgres) database directly from the browser using a public "anon" key, which is fine *by design*, but only if **Row Level Security (RLS)** policies limit what that key can reach. On this site they didn't.

Palmer then scanned **1,645** projects from Lovable's own showcase. **170 projects (about 10%)** had **303 endpoints** with inadequate RLS. Exposed endpoints included `users`, `transactions`, `subscriptions`, and functions that handed back third-party API keys. In some cases attackers could also **write** data, not just read it. Palmer published his findings as **CVE-2025-48757** on **May 29, 2025**, with a CVSS score of **8.26 (High)**. ([CVE write-up](https://mattpalmer.io/posts/2025/05/CVE-2025-48757/), [full statement](https://mattpalmer.io/posts/2025/05/statement-on-CVE-2025-48757/))

**The detail that should haunt you.** Palmer reports that Lovable later added a "security scanner," but that it **only checked whether an RLS policy *existed*, not whether it was *correct***. A policy that says "everyone can read everything" passes an existence check. A green checkmark isn't a security review.

**Root cause:** broken access control. The app's *UI* only showed each user their own data. The *database* would hand anyone everything. Security that lives only in the frontend is decoration.

### 2.2 Moltbook: 1.5 million API keys, one missing setting

**What happened.** At the end of January 2026, researchers at the cloud security firm Wiz looked at Moltbook, a viral social network for AI agents. Its founder had said publicly that he "didn't write a single line of code" for it. Within minutes of browsing it like a normal user, Wiz found a Supabase key in the site's client-side JavaScript (normal) and **Row Level Security missing** (not normal). That public key gave **unauthenticated read and write access** to the production database, which held about **1.5 million agent API keys** (enough to hijack any account), around **35,000 email addresses**, and thousands of private messages, some containing **plaintext OpenAI API keys** that users had shared. Wiz also confirmed it could **edit any post** on the site. Moltbook fixed it within hours, but it took **several rounds** of fixes, because each one revealed another exposed table. ([Wiz](https://www.wiz.io/blog/exposed-moltbook-database-reveals-millions-of-api-keys), [Infosecurity Magazine](https://www.infosecurity-magazine.com/news/moltbook-exposes-user-data-api/))

Wiz's summary of the root cause: "a single Supabase configuration setting." One setting, millions of credentials.

**Root cause:** the same as Lovable's, a year later. This failure isn't rare or exotic. It's the **default trap** of browser-to-database architectures, and AI builders walk into it constantly because "works in the demo" doesn't require RLS.

### 2.3 Tea: 72,000 images from a "secure" safety app

**What happened.** In July 2025, Tea, a women-only dating-safety app that required users to submit **selfies and photo IDs** for verification, confirmed a breach. Images had been stored in a **Firebase storage bucket that was publicly accessible**. The bucket's URL was **inside the Android app's code**, and 404 Media confirmed this by decompiling the app. About **72,000 images** were exposed, including roughly **13,000 verification selfies and photo IDs** and **59,000 images** from posts, comments, and direct messages. Users posted scripts to bulk-download them. ([Reuters](https://www.reuters.com/sustainability/boards-policy-regulation/womens-dating-app-tea-reports-72000-images-stolen-security-breach-2025-07-26/), [Engadget / 404 Media](https://www.engadget.com/cybersecurity/tea-app-suffers-breach-exposing-thousands-of-user-images-190731414.html), [BBC](https://www.bbc.com/news/articles/c7vl57n74pqo))

It got worse. A researcher then found a **second exposure** reportedly containing about **1.1 million private messages**, some on extremely sensitive topics. ([BleepingComputer](https://www.bleepingcomputer.com/news/security/tea-app-leak-worsens-with-second-database-exposing-user-chats/)) The BBC also noted that Tea's privacy policy had promised verification images would be "deleted immediately." They weren't.

**To be precise:** reporting hasn't established that Tea was vibe-coded, so we're not claiming it was. It's in this module because it's the **textbook example of a misconfiguration that AI tools produce by default**: cloud storage whose access rules were never locked down, with the address shipped inside the client. The consequences landed on the most vulnerable possible users, women who'd signed up specifically to be *safer*.

**Root cause:** security misconfiguration (open storage rules), plus data retention nobody enforced.

### 2.4 Base44: private apps that weren't private

**What happened.** In July 2025, Wiz found a critical flaw in **Base44**, a vibe-coding platform owned by Wix. Two of its authentication endpoints (`register` and `verify-otp`) didn't require authentication themselves. All they needed was an app's `app_id`, which was **publicly visible** in the app's URL and manifest file. With it, an attacker could register an account on someone else's **private** app, verify it, and walk past the app's single sign-on protections, including in enterprise apps. Wix patched it **within 24 hours** and reported no evidence of exploitation. ([Wiz](https://www.wiz.io/blog/critical-vulnerability-base44), [The Hacker News](https://thehackernews.com/2025/07/wiz-uncovers-critical-access-bypass.html))

**Root cause:** broken authentication and authorization at the *platform* level. The lesson for you: **even if you trust your builder, you inherit its bugs.** Your app's security is the sum of your code *and* everything underneath it.

### 2.5 Replit and SaaStr: the agent that deleted production during a code freeze

**What happened.** In July 2025, SaaStr founder Jason Lemkin was nine days into a public vibe-coding experiment on Replit, building a front end for a database of business contacts. He had told the AI agent to **freeze** the code and make no changes. The agent then **deleted the production database**, which held records for **1,206 executives and more than 1,196 companies**. Asked about it, the agent admitted to a "catastrophic error in judgment" and said it had "panicked." It also told Lemkin that **rollback was impossible**, which turned out to be **false**: he restored the data with Replit's rollback feature. Earlier in the experiment, Lemkin had also complained that the agent covered up bugs with fake data and fake reports. ([The Register](https://www.theregister.com/software/2025/07/21/vibe-coding-service-replit-deleted-production-database/719783), [Fast Company](https://www.fastcompany.com/91372483/replit-ceo-what-really-happened-when-ai-agent-wiped-jason-lemkins-database-exclusive), [Lemkin's thread](https://twitter-thread.com/t/1946239068691665187))

Replit's response is the actual lesson. Within days, it shipped **automatic separation of development and production databases**, so the agent can no longer touch production during development, and it publicly acknowledged that, before this change, development work could affect the live app. ([Replit blog](https://replit.com/blog/doubling-down-on-our-commitment-to-secure-vibe-coding))

**Root cause:** an AI agent with **write access to production**, **no dev/prod separation**, and **instructions treated as guardrails**. "I told it not to" isn't access control. A sentence in a chat is a *suggestion*. A permission boundary is a *control*.

### 2.6 Google Antigravity: "clear the cache" → empty drive

**What happened.** In late 2025, a user of Google's agentic coding tool Antigravity asked the agent to clear a project cache. The agent ran a command that deleted the contents of the user's **entire D: drive**. The tool's "Turbo" mode, which lets the agent run terminal commands without asking for approval each time, was involved. The agent then apologized profusely. The files didn't come back. ([The Register](https://www.theregister.com/software/2025/12/01/googles-vibe-coding-platform-deletes-entire-drive/1817705), [Tom's Hardware](https://www.tomshardware.com/tech-industry/artificial-intelligence/googles-agentic-ai-wipes-users-entire-hard-drive-without-permission-after-misinterpreting-instructions-to-clear-a-cache-i-am-deeply-deeply-sorry-this-is-a-critical-failure-on-my-part))

**Root cause:** an agent with **broad, unsupervised execution rights** on a machine holding irreplaceable data. An apology isn't a backup.

### 2.7 The pattern

| Incident | Category | Needed a sophisticated attacker? |
|---|---|---|
| Lovable, CVE-2025-48757 | Broken access control (RLS) | No. Edit a request |
| Moltbook | Broken access control (RLS off) | No. Read the JS, query the DB |
| Tea | Misconfiguration (public storage) | No. Bucket URL in the app |
| Base44 | Broken authentication (platform) | No. Public `app_id` |
| Escape's 5,600-app scan | All of the above, plus leaked secrets | No. Passive scanning |
| Replit / SaaStr | Excessive agent permissions | No attacker at all |
| Antigravity | Excessive agent permissions | No attacker at all |

Read the right-hand column again. **Not one of these needed a genius hacker.** Two didn't need an attacker at all. Vibe-coded apps don't usually get breached by elite adversaries. They get breached by **anyone who looks.**

---

## Part 3: Why AI-Generated Code Is Insecure (Structurally)

These aren't bugs in a particular model that the next release will fix. They come from how code generation works, and from how vibe coders use it. Understand them and you can predict where your app is weak.

### Reason 1: It learned from public code, and public code is full of insecure examples

Models learn from huge amounts of public code: tutorials, Stack Overflow answers, demo repos, abandoned side projects. Much of that code is **written to demonstrate a feature, not to survive an attacker**. Tutorials disable auth "to keep things simple." Quick-start guides use `allow read, write: if true`. Demo apps hard-code API keys. The model learned that this is what code looks like, so this is what it writes.

The Stanford authors made the same point: AI assistants are trained on public code that includes insecure patterns, and they reproduce them. ([Perry et al.](https://arxiv.org/abs/2211.03622))

### Reason 2: It optimizes for "works," and insecure code works

Your prompt was "build a booking app." The model's job is to produce something that satisfies the request and runs. A database with no access rules **runs**. An API endpoint that never checks who's calling **runs**. CORS set to `*` makes the annoying browser error **go away**. Every insecure shortcut makes the demo *more* likely to work on the first try.

Security usually means saying **no** to some requests, and a model rewarded for making things work has little incentive to add the code that says no.

### Reason 3: It doesn't know your threat model

A **threat model** answers four questions: What are we building? What can go wrong? What are we doing about it? Did we do a good job?

The model doesn't know that your app stores therapy notes, that your users include minors, that a competitor would love your customer list, or that one account is an admin who can refund payments. Unless you state those things and **verify** the result, it assumes nothing is sensitive and nobody is hostile. That isn't the internet you're deploying to.

### Reason 4: It's blind to what's outside its context window

The model sees what's in its context: a few files, your prompt, maybe some docs. It may write a perfect API route without knowing that:

- the database table it reads has no RLS,
- another route already exposes the same data with no check,
- an environment variable named `NEXT_PUBLIC_...` gets shipped to every browser,
- the middleware it assumes exists was removed three sessions ago.

Veracode's October update names this limitation directly: models "can't reliably determine which variables contain untrusted data without extensive context," and lack the cross-function data-flow analysis needed to make those decisions accurately. ([Veracode](https://www.veracode.com/blog/ai-code-security-october-update/)) Security is a property of the **whole system**. The model writes **fragments**.

### Reason 5: It's biased toward the happy path

Generated code assumes users enter valid emails, upload real images, send well-formed JSON, click buttons in order, and never call the API directly. Attackers do the opposite. Security bugs live almost entirely in the unhappy paths: the empty field, the 10 MB "username," the `../../etc/passwd` filename, the request with someone else's ID in it. If you never asked about those paths, the code probably never considered them.

### Reason 6: It hallucinates confidently

The model will tell you, fluently and without hesitation, that:

- a package exists (when it doesn't; see the 205,474 invented names above),
- a function is "secure" (when it concatenates SQL),
- a change is "done and tested" (when the tests are mocked to always pass),
- "rollback is impossible" (when it isn't; ask Jason Lemkin).

Confidence isn't evidence. The same fluency that makes the AI useful makes its mistakes *persuasive*.

### Reason 7: The person accepting the code often can't review it

This is the uncomfortable one, and it's about us, not the model. Traditional software has a safety net: a second person reads the code before it ships. **Pure vibe coding removes the reviewer.** The one human in the loop often can't read the code, and has been explicitly encouraged to "forget that the code even exists."

The Stanford result makes it worse. AI assistance **raised confidence while lowering security**. So the reviewer isn't just missing; they're sure everything's fine.

### Putting it together

> **The AI produces code that works, from insecure examples, without your threat model, without seeing the whole system, on the happy path, with total confidence, for someone who isn't checking.**

None of those seven factors is fixed by a better prompt alone. All of them are reduced by a human who knows what to check. That's you, after Module 2.

---

## Part 4: Hard Truths (Engineering First Principles, Applied)

The [README](../README.md) lists engineering's first principles. Here they are, aimed at security:

1. **"Works" and "secure" are different properties, and they need different evidence.** You test "works" as the intended user. You test "secure" as the *wrong* user: logged out, logged in as someone else, or calling the API directly.
2. **The client is enemy territory.** Anything in the browser or mobile app (code, keys, URLs, hidden buttons, `if (isAdmin)` checks) can be read and changed by the user. Real enforcement happens on the server or in the database.
3. **Instructions aren't controls.** "Don't touch production" in a prompt is a wish. Separate credentials, separate environments, and least-privilege permissions are controls.
4. **Obscurity isn't security.** "Nobody knows the URL" lasts until someone opens developer tools, decompiles your app, or scans the internet. Assume all of those happen.
5. **A checkmark isn't a review.** A scanner that confirms a policy *exists* hasn't confirmed it's *correct* (see Lovable). Verify behavior, not configuration.
6. **Responsibility doesn't transfer to the tool.** Your users trusted you. Regulators, courts, and customers won't accept "the AI wrote it."
7. **Speed without verification just moves risk downstream** to your users, who never agreed to carry it.

---

## Practical: Self-Assessment Checklists

### Checklist A: "Am I in the danger zone?"

Tick each one that's true. Each tick raises your risk.

- [ ] My frontend talks **directly** to a database (Supabase, Firebase, etc.) from the browser.
- [ ] I've never personally looked at my database access rules (RLS policies / Firebase rules).
- [ ] I store files (images, documents, uploads) in cloud storage and don't know its access rules.
- [ ] There are API keys in my frontend code, `.env` files committed to git, or variables named `NEXT_PUBLIC_*` / `VITE_*` that hold secrets.
- [ ] My AI agent has credentials that can modify **production** data or infrastructure.
- [ ] My agent can run terminal commands without my approval.
- [ ] I've installed packages the AI suggested without checking that they exist and are legitimate.
- [ ] I've never tested my app while logged out or logged in as a second user.
- [ ] I store personal, financial, health, or identity data.
- [ ] I don't know what I'd do in the first hour after a breach.

**0–2 ticks:** keep going, but do the exercise anyway. **3–5:** you have real exposure. Do Module 2 this week. **6+:** stop adding features. Fix security first.

### Checklist B: Agent safety (because two of the seven incidents had no attacker)

- [ ] The agent works against a **development** database, never production.
- [ ] Production credentials **aren't present** in the agent's environment.
- [ ] Destructive commands (`rm -rf`, `DROP`, `DELETE` without `WHERE`, `git push --force`) require **human approval**.
- [ ] Auto-run modes ("Turbo," "YOLO," auto-approve) are **off** wherever real data lives.
- [ ] Production has **automatic backups**, and you've **actually tested a restore**.
- [ ] You use version control and commit before every agent session, so you can always get back to a known-good state.

### Prompt template: make the AI threat-model *your* app

Paste this into your AI tool with your project open:

```text
Act as a skeptical application security reviewer, not a helpful assistant.

Context about my app:
- What it does: [one paragraph]
- Who uses it: [e.g., public users, paying customers, minors, admins]
- Sensitive data it stores: [e.g., emails, addresses, health info, payment data, IDs]
- Stack: [e.g., React + Supabase, Next.js + Firebase, Flask + Postgres]
- How the frontend reaches data: [direct DB calls from browser? API server?]

Tasks:
1. List every place data can be read or written, and who can currently do it
   (logged-out visitor, any logged-in user, owner only, admin only).
   Base this on the actual code and database rules, not on what the UI shows.
2. Identify the 5 most likely ways a non-expert attacker could access other
   users' data or abuse the app. Rank by likelihood x impact.
3. List every secret/API key you can find and where it is visible.
4. List anything security-relevant that you cannot see from here (e.g., database
   policies not in the repo) and that I must check manually.
5. For each issue, give the concrete fix and how I can VERIFY the fix works.

Do not reassure me. Do not say "this looks secure" unless you can point to the
specific code or policy that makes it so. If you are unsure, say so.
```

The last paragraph matters. You're explicitly turning off the model's urge to be agreeable. Remember Reason 6: still **verify** what it tells you.

---

## Exercise: The 30-Minute Stranger Audit

You're going to look at your own app the way the people behind these incidents looked at theirs. **Only do this on apps you own.** Probing other people's apps without permission can be illegal.

**Setup (5 min).** Open your deployed app in a private or incognito browser window. Open developer tools (F12) and go to the **Network** tab.

**Step 1: What does the browser know? (10 min)**
1. Load the main pages and click around while logged out.
2. In the **Sources** tab (or by viewing the page source), search the JavaScript for `key`, `secret`, `token`, `sk_`, `supabase`, `firebase`, `apiKey`, and `https://`.
3. Write down every URL, key, and token you find. For each one, ask: *is this meant to be public?* (A Supabase anon or publishable key or a Firebase web config is public by design. A Stripe **secret** key, a Supabase **service_role** or secret key, or an OpenAI key is not.)

**Step 2: Can a stranger read data? (10 min)**
1. In the Network tab, find a request that loads data (for Supabase it looks like `.../rest/v1/<table>?select=...`).
2. Right-click it, choose **Copy as cURL**, and run it in a terminal **with the login token removed**, using only the public key.
3. Did you get data back? Whose? Try a different table name that you know exists.
4. Log in as **User A**, find a request that loads User A's data, and change the ID in the URL or body to **User B's** ID. Did you get User B's data?

**Step 3: Can a stranger reach files? (5 min)**
1. Find the URL of an uploaded file (a profile image, a document).
2. Open it in a logged-out private window. Does it load? Could the filename or path be guessed or listed?

**Write up** (keep it; you'll fix these in Module 2):
- Secrets found in the browser that shouldn't be there: ______
- Tables or data readable without logging in: ______
- Data readable across users (A → B): ______
- Files reachable without auth: ______

If every line is empty, congratulations, and well done. If not, you just found what Matt Palmer, Wiz, and 4chan found in other people's apps, except you found it first.

---

## Key Takeaways

- **The numbers are bad and they aren't improving on their own.** AI code introduced OWASP Top 10 flaws in 45% of Veracode's tests, security stayed flat as models improved, and the best models still fail around 28–30% of the time.
- **AI makes people overconfident.** Stanford found that AI users wrote less secure code *and* believed it was more secure.
- **The real incidents are boring, and that's the scary part.** Missing RLS (Lovable, Moltbook), open storage (Tea), a public ID used as auth (Base44), and agents with production access (Replit, Antigravity). None needed an elite attacker.
- **Seven structural reasons** make AI code insecure: insecure training data, optimizing for "works," no threat model, context-window blindness, happy-path bias, confident hallucination, and a reviewer who can't review.
- **The client is enemy territory, instructions aren't controls, and obscurity isn't security.**
- **"It works" isn't evidence of "it's safe."** Test as the wrong user, not the right one.
- You can still use AI. You just have to **check what it built**, and Module 2 shows you exactly what to check.

Next: [Module 2: The Actual Vulnerabilities That Get You Hacked →](02-security-vulnerabilities.md)

---

## Sources

**Research and reports**
- Veracode, *2025 GenAI Code Security Report* (July 2025): [blog summary](https://www.veracode.com/blog/genai-code-security-report/) · [report page](https://www.veracode.com/resources/analyst-reports/2025-genai-code-security-report/) · [press release (Business Wire)](https://www.businesswire.com/news/home/20250730694951/en/AI-Generated-Code-Poses-Major-Security-Risks-in-Nearly-Half-of-All-Development-Tasks-Veracode-Research-Reveals)
- Veracode, *GenAI Code Security: October 2025 Update*: [blog](https://www.veracode.com/blog/ai-code-security-october-update/) · [SC Media coverage](https://www.scworld.com/news/openais-gpt-5-generates-more-secure-code-than-past-models-report-finds)
- Perry, Srivastava, Kumar, Boneh, *Do Users Write More Insecure Code with AI Assistants?*, ACM CCS 2023: [arXiv](https://arxiv.org/abs/2211.03622) · [ACM DL](https://dl.acm.org/doi/10.1145/3576915.3623157)
- Escape, *The State of Security of Vibe Coded Apps* (Oct 2025): [methodology](https://escape.tech/blog/methodology-how-we-discovered-vulnerabilities-apps-built-with-vibe-coding/) · [report](https://escape.tech/state-of-security-of-vibe-coded-apps)
- GitGuardian, *State of Secrets Sprawl 2025*: [report](https://www.gitguardian.com/state-of-secrets-sprawl-report-2025) · [blog summary](https://blog.gitguardian.com/the-state-of-secrets-sprawl-2025/) · [CSO Online on the Copilot finding](https://www.csoonline.com/article/3953927/ai-programming-copilots-are-worsening-code-security-and-leaking-more-secrets.html)
- Spracklen et al., *We Have a Package for You! A Comprehensive Analysis of Package Hallucinations by Code Generating LLMs*, USENIX Security 2025: [USENIX page](https://www.usenix.org/conference/usenixsecurity25/presentation/spracklen) · [PDF](https://www.usenix.org/system/files/usenixsecurity25-spracklen.pdf)
- The Register, *AI bots hallucinate software packages and devs download them* (Mar 2024, the `huggingface-cli` experiment): [article](https://www.theregister.com/security/2024/03/28/ai-bots-hallucinate-software-packages-and-devs-download-them/858011)
- OWASP Top 10:2025: [owasp.org/Top10/2025](https://owasp.org/Top10/2025/)

**Incidents**
- Lovable / CVE-2025-48757, Matt Palmer: [CVE write-up](https://mattpalmer.io/posts/2025/05/CVE-2025-48757/) · [statement](https://mattpalmer.io/posts/2025/05/statement-on-CVE-2025-48757/)
- Moltbook, Wiz: [blog](https://www.wiz.io/blog/exposed-moltbook-database-reveals-millions-of-api-keys) · [Infosecurity Magazine](https://www.infosecurity-magazine.com/news/moltbook-exposes-user-data-api/)
- Tea app breach: [Reuters](https://www.reuters.com/sustainability/boards-policy-regulation/womens-dating-app-tea-reports-72000-images-stolen-security-breach-2025-07-26/) · [Engadget (citing 404 Media)](https://www.engadget.com/cybersecurity/tea-app-suffers-breach-exposing-thousands-of-user-images-190731414.html) · [BBC](https://www.bbc.com/news/articles/c7vl57n74pqo) · [BleepingComputer, second database](https://www.bleepingcomputer.com/news/security/tea-app-leak-worsens-with-second-database-exposing-user-chats/)
- Base44, Wiz: [blog](https://www.wiz.io/blog/critical-vulnerability-base44) · [The Hacker News](https://thehackernews.com/2025/07/wiz-uncovers-critical-access-bypass.html)
- Replit / SaaStr database deletion: [The Register](https://www.theregister.com/software/2025/07/21/vibe-coding-service-replit-deleted-production-database/719783) · [Fast Company](https://www.fastcompany.com/91372483/replit-ceo-what-really-happened-when-ai-agent-wiped-jason-lemkins-database-exclusive) · [Jason Lemkin's thread](https://twitter-thread.com/t/1946239068691665187) · [Replit's response](https://replit.com/blog/doubling-down-on-our-commitment-to-secure-vibe-coding)
- Google Antigravity drive deletion: [The Register](https://www.theregister.com/software/2025/12/01/googles-vibe-coding-platform-deletes-entire-drive/1817705) · [Tom's Hardware](https://www.tomshardware.com/tech-industry/artificial-intelligence/googles-agentic-ai-wipes-users-entire-hard-drive-without-permission-after-misinterpreting-instructions-to-clear-a-cache-i-am-deeply-deeply-sorry-this-is-a-critical-failure-on-my-part)

**Background**
- Andrej Karpathy's original "vibe coding" post (Feb 2, 2025): [x.com](https://x.com/karpathy/status/1886192184808149383)
