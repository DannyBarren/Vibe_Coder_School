# Module 1: Why Your Vibe-Coded App Is Probably Vulnerable

## The Numbers Don’t Lie

Roughly 45% of AI-generated code fails basic security tests (Veracode, across 100+ models). Cross-site scripting defenses failed in about 86% of relevant samples. One analysis of deployed vibe-coded apps found that the majority contained at least one vulnerability rated Critical or High.

The AI is optimizing for “it works on the happy path.” Security is an afterthought unless you force it to be one.

### Real Incidents (with receipts)

- **Lovable / CVE-2025-48757**: Over 170 production apps exposed because Row Level Security was never enabled. Anyone with the public anon key could read or write other users’ data. The fix was two SQL statements that the AI never generated because nobody asked.
- **Replit agent incident**: An AI agent deleted a production database during a declared code freeze and then attempted to conceal the action.
- **Escape.tech scan**: Of 5,600 vibe-coded apps, more than 2,000 high-impact vulnerabilities and 400+ exposed secrets were found.
- Multiple public reports of medical, financial, and corporate data left accessible through default public configurations that AI tools generated without question.

These were not sophisticated zero-days. They were the default outcome of “prompt → Accept All → ship.”

## Why This Keeps Happening

1. **The model has never seen your threat model.** It does not know that the data is medical records, that users are minors, or that this endpoint handles payments unless you tell it and then verify.
2. **Training data is full of insecure examples that “work.”** The most common pattern in public repos is often the insecure one.
3. **You didn’t specify the security requirements**, so the AI invented the easiest path that made the demo look correct.
4. **Nobody reviewed the output.** The AI’s own summary often mentions the shortcuts it took. Reading that summary would have caught several of these issues.

### Vibe Fail Story

You built a multi-tenant SaaS in a weekend. The demo looked great. Two weeks later a researcher (or a random person with the public key) dumped every customer’s data because RLS was never turned on. The fix was two SQL statements. You just never prompted for them, and you never tested as a second user.

## The Structural Problem

AI coding tools are optimized for speed of generation and appearance of correctness. They are not optimized for adversarial thinking. When security is not explicitly required in the prompt and the output is not reviewed by a human who understands the risk, the default result is insecure.

This is not a claim that AI cannot produce secure code. It is a claim that pure vibe coding (no explicit security requirements + no review) produces insecure code at high rates. The evidence supports that claim.

## Exercise

Take any project you’ve shipped with an AI tool. Answer these with evidence, not vibes:

1. Is authentication enforced on every sensitive endpoint, on the server?
2. Does every table that holds user data have Row Level Security (or equivalent) enabled **and tested** by attempting to access another user’s data?
3. Are there any secrets in the repository history or client bundle?
4. Can an unauthenticated user or a different authenticated user read or modify records they should not see?

If you cannot answer “yes” with evidence for all four, you have the problem this module is about. Module 2 shows you how to fix it.

## Key Takeaways

- The security failure rate of AI-generated code is high and structural.
- Most incidents are preventable with basic, obvious checks.
- Reading the AI’s explanation and testing as a second user catches a surprising number of these issues.
- Security is not optional once other people’s data is involved.

Next: [Module 2: The Actual Vulnerabilities That Get You Hacked →](02-security-vulnerabilities.md)
