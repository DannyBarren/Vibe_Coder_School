# Module 1: Why Your Vibe-Coded App Is Probably Vulnerable

## The Numbers Don’t Lie

Roughly 45% of AI-generated code fails basic security tests (Veracode, across 100+ models). One study found 91% of deployed vibe-coded apps had at least one vulnerability, most of them Critical or High.

The AI is optimizing for “it works on the happy path.” Security is an afterthought unless you force it to be one.

### Real Incidents

- **Lovable / CVE-2025-48757**: Over 170 production apps exposed because Row Level Security was never enabled. Anyone with the public anon key could read or write other users’ data.
- Agents deleting production databases (Replit and others).
- Thousands of vibe-coded apps left publicly accessible with medical, financial, and corporate data exposed.
- Escape.tech scan of 5,600 apps: 2,000+ high-impact vulnerabilities and 400+ exposed secrets.

These were not sophisticated attacks. They were the default outcome of “prompt → Accept All → ship.”

## Why This Keeps Happening

1. The model has never seen the threat model for *your* app.
2. Training data is full of insecure examples that “work.”
3. You didn’t specify the security requirements, so the AI invented the easiest path.
4. Nobody reviewed the output.

### Vibe Fail Story

You built a multi-tenant SaaS in a weekend. The demo looked great. Two weeks later a researcher (or a random person with the public key) dumped every customer’s data because RLS was never turned on. The fix was two SQL statements. You just never prompted for them.

## Exercise

Take any project you’ve shipped with an AI tool. Answer these:

- Is authentication enforced on every sensitive endpoint, on the server?
- Does every table that holds user data have Row Level Security (or equivalent) enabled and tested?
- Are there any secrets in the repository or client bundle?

If you can’t answer “yes” with evidence, you have the problem this module is about.
