# Module 7: Data Modeling & Persistence

> Code is a renter. Data is the landlord.
> You can rewrite your whole app in a weekend. You can't un-corrupt three years of customer records.

Here's an uncomfortable truth about software: **your code is disposable; your data is not.** You can delete a bad feature, roll back a bad deploy, or rewrite the frontend in whatever framework is trendy this month. But if the data is wrong, missing, duplicated, or gone, there may be no undo button at all.

AI tools treat the database like a junk drawer. Need to store something? Add a column. Different shape? Stuff it in a JSON blob. Need to change the schema? Edit the table directly in the production dashboard. It all works, right up until it doesn't, and then it fails in the most expensive way possible.

This module gives you the data fundamentals that every app relies on: how to structure tables, how to make the database enforce your rules, how to change the schema without breaking things, and how to make sure you can recover when (not if) something goes wrong.

---

## Learning Objectives

By the end of this module you will be able to:

1. Model data as tables and relationships (one-to-many, many-to-many) instead of one giant blob.
2. Apply basic normalization so each fact is stored once.
3. Use primary keys, foreign keys, and constraints so the database rejects bad data even when your code doesn't.
4. Add indexes where they matter and recognize when they're missing.
5. Change your schema safely with migrations, and never hand-edit production.
6. Use transactions so multi-step changes happen completely or not at all.
7. Set up backups and, crucially, **test restoring them**.
8. Choose between soft and hard deletes on purpose.
9. Avoid the classic money and time-zone traps.

---

## Vibe Fail: The Orders Table With 60 Columns

*(Illustrative story; every piece of it is common.)*

Sam's meal-kit startup ran on a single `orders` table the AI designed in the first week. It had 60 columns: `customer_name`, `customer_email`, `customer_address`, `item1_name`, `item1_price`, `item2_name`... up to `item8_price`, plus a `notes` JSON column that held "everything else."

Problems arrived in order:

1. **A customer changed their email.** It was stored on every order, so now they had two emails across 140 orders, and the "my orders" page showed half of them.
2. **Someone ordered nine items.** There was no `item9`. The AI's fix stuffed the extra items into `notes`. Revenue reports now silently undercounted.
3. **Prices were stored as floats.** A $19.99 kit times three sometimes became $59.97000000000001, and the accounting export didn't match Stripe.
4. **Duplicate orders appeared** because a double-click submitted twice, and nothing in the database prevented it.
5. **Then the AI "cleaned up" the schema** by running a migration it wrote directly against production, renaming `customer_email` to `email`. The live app, still using the old name, broke for every user for 40 minutes.
6. **They went to restore from backup.** Backups had been "enabled" on the hosting dashboard, on a plan that kept them for 24 hours. The bad migration had been run 30 hours earlier, during a previous cleanup attempt.

Not one of these problems needed a database expert. They needed about an hour of data modeling at the start, and one tested restore.

---

## Core Concepts

### 1. Tables, rows, and relationships

A relational database (Postgres, MySQL, SQLite, and what sits underneath Supabase, Neon, PlanetScale, and many others) stores data in **tables**. Each table represents one kind of thing: users, orders, products. Each **row** is one instance. Each **column** is one attribute.

The power comes from **relationships** between tables:

- **One-to-many:** one customer has many orders. Each order belongs to one customer. The `orders` table has a `customer_id` column pointing to `customers`.
- **Many-to-many:** an order contains many products, and a product appears in many orders. You need a **join table** (`order_items`) with `order_id` and `product_id`, plus anything specific to that pairing (quantity, price at time of purchase).
- **One-to-one:** a user has one profile. Often just columns on the same table; sometimes split for security or size.

```text
customers 1 ──── * orders 1 ──── * order_items * ──── 1 products
```

Sam's 60-column table, modeled properly:

```sql
CREATE TABLE customers (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  email       text NOT NULL UNIQUE,
  name        text NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE products (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name         text NOT NULL,
  price_cents  integer NOT NULL CHECK (price_cents >= 0),
  active       boolean NOT NULL DEFAULT true
);

CREATE TABLE orders (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id      uuid NOT NULL REFERENCES customers(id),
  status           text NOT NULL CHECK (status IN ('pending','paid','shipped','cancelled')),
  idempotency_key  text NOT NULL UNIQUE,
  created_at       timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE order_items (
  order_id          uuid NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id        uuid NOT NULL REFERENCES products(id),
  quantity          integer NOT NULL CHECK (quantity > 0),
  unit_price_cents  integer NOT NULL CHECK (unit_price_cents >= 0), -- price at time of purchase
  PRIMARY KEY (order_id, product_id)
);
```

Now a customer can change their email once. An order can have 1 item or 100. Prices are exact. Duplicate submissions are blocked by the `UNIQUE` idempotency key.

### 2. Normalization basics: store each fact once

**Normalization** is a formal topic with numbered "normal forms," but the everyday version fits in a sentence: **each fact should be stored in exactly one place.** (Yes, this is the single-source-of-truth idea from [Module 5](05-architecture.md), applied to data.)

Signs you need to normalize:

- Repeating column groups (`item1`, `item2`, `item3`...).
- The same information copied across many rows (customer email on every order).
- Comma-separated lists in a column (`tags = "red,large,sale"`).
- Updating one fact requires updating many rows.

**When duplication is OK:** when you're deliberately capturing a snapshot. `unit_price_cents` on `order_items` is a copy of the product price, on purpose, because the order should remember what the customer actually paid even if the product price changes later. That's a conscious decision, not an accident. (Same for shipping addresses on orders.)

**What about JSON columns?** Postgres `jsonb` is genuinely useful for truly flexible, rarely-queried data: third-party webhook payloads, user preferences, feature-specific settings. It's a bad home for anything you'll filter, join, sum, or enforce rules on. "Just put it in JSON" has a shelf life, and it expires the day someone asks for a report.

### 3. Primary and foreign keys

A **primary key** uniquely identifies each row. Every table needs one. Common choices: auto-incrementing integers (`bigint generated always as identity`) or UUIDs. UUIDs are nice for public-facing IDs because they're not guessable or sequential (though unguessable IDs are **not** a substitute for permission checks; see [Module 2](02-security-vulnerabilities.md)).

A **foreign key** says "this column must match a row in that other table."

```sql
customer_id uuid NOT NULL REFERENCES customers(id)
```

With that in place, the database refuses to create an order for a customer who doesn't exist, and refuses to delete a customer who still has orders (unless you've said what should happen). Without it, you get **orphans**: orders pointing to customers who were deleted, which crash your UI and corrupt your reports.

Foreign key options worth knowing:

- `ON DELETE CASCADE`: deleting the parent deletes the children (good for `order_items` when an order is deleted; terrifying on `customers → orders`).
- `ON DELETE RESTRICT` (or the default `NO ACTION`): refuse to delete the parent while children exist. Safe default.
- `ON DELETE SET NULL`: keep the child, clear the reference.

Many AI-generated schemas, especially those created through ORMs or dashboards, skip foreign keys entirely. Check yours.

### 4. Constraints: the last line of defense

Your code has validation. Your API has validation. Your frontend has validation. And one day, a bug, a script, a migration, an admin tool, a second app, or an AI agent with database access will bypass all of it. **Constraints are the database's own validation, and they apply no matter who's writing.**

```sql
email        text NOT NULL UNIQUE,                         -- required, no duplicates
quantity     integer NOT NULL CHECK (quantity > 0),        -- no zero or negative quantities
status       text NOT NULL CHECK (status IN ('pending','paid','shipped','cancelled')),
starts_at    timestamptz NOT NULL,
ends_at      timestamptz NOT NULL,
CHECK (ends_at > starts_at),                               -- no time travel
UNIQUE (team_id, slug)                                     -- slug unique within a team
```

| Constraint | Prevents |
|---|---|
| `NOT NULL` | Missing required data |
| `UNIQUE` | Duplicates (emails, slugs, idempotency keys) |
| `CHECK` | Invalid values (negative prices, unknown statuses) |
| `FOREIGN KEY` | References to things that don't exist |
| `PRIMARY KEY` | Duplicate or missing row identity |

**Hard truth:** if a rule is important enough to write in your code, it's probably important enough to enforce in the database too. Code has bugs. Constraints don't get tired.

When a constraint rejects an insert, handle the error gracefully (e.g., a unique violation on email becomes "That email is already registered") rather than swallowing it ([Module 6](06-observability.md)).

### 5. Indexes: why your app got slow at 10,000 rows

An **index** is like the index at the back of a book: it lets the database find rows without reading every page. Without one, a query like `WHERE customer_id = $1` scans the whole table. Fine with 100 rows. Painful with 1,000,000.

```sql
-- You almost always want indexes on:
CREATE INDEX idx_orders_customer_id ON orders (customer_id);   -- foreign keys you filter by
CREATE INDEX idx_orders_created_at ON orders (created_at DESC); -- columns you sort or range-filter by
-- UNIQUE constraints and primary keys create indexes automatically.
```

Note: Postgres does **not** automatically index foreign key columns. That's a very common reason a vibe-coded app slows to a crawl as it grows.

Guidelines:

- Index columns used in `WHERE`, `JOIN`, and `ORDER BY` on large tables.
- Composite indexes (`(team_id, created_at)`) help queries that filter by the first and sort by the second.
- Don't index everything. Each index slows down writes and uses storage.
- Use `EXPLAIN ANALYZE` on slow queries to see whether they use an index or scan the whole table. ("Seq Scan" on a big table is your clue.)

**Also watch for N+1 queries:** code that fetches a list, then runs one more query per item. Fetching 50 orders and then 50 separate queries for their customers = 51 queries. AI writes this constantly, especially with ORMs. Use joins or the ORM's "include/preload" feature.

### 6. Migrations: never hand-edit production

A **migration** is a versioned script that changes your schema: create a table, add a column, add an index. Migrations live in your repo, run in order, and are applied the same way to every environment (local, staging, production).

```text
migrations/
  0001_create_customers.sql
  0002_create_orders.sql
  0003_add_order_status.sql
  0004_index_orders_customer_id.sql
```

Every major tool has a migration system: Prisma Migrate, Drizzle Kit, Supabase CLI migrations, Rails, Django, Alembic (SQLAlchemy), Knex, Flyway, and more. Use the one that matches your stack.

**The rules:**

1. **Never change production's schema by hand** (dashboard table editors, ad-hoc SQL consoles, or an agent with prod credentials). If it's not in a migration, it's not reproducible, not reviewable, and staging no longer matches production.
2. **Never edit a migration that has already run** anywhere shared. Write a new one.
3. **Review the generated SQL.** ORMs sometimes generate a "drop column and create a new one" when you meant "rename," which deletes the data.
4. **Test migrations on a copy of real-ish data** (staging) before production.
5. **Make changes backward-compatible** when the app is live. Old code and new code will run at the same time during a deploy (more in [Module 8](08-deployment.md)).

**The expand/contract pattern** for risky changes (e.g., renaming `customer_email` to `email`):

1. **Expand:** add the new column `email`. Deploy code that writes to both columns.
2. **Backfill:** copy existing data from `customer_email` to `email`.
3. **Switch:** deploy code that reads from `email`.
4. **Contract:** once nothing uses `customer_email`, drop it in a later migration.

Four steps instead of one. Zero downtime instead of Sam's 40 minutes.

**Destructive operations deserve extra suspicion:** `DROP TABLE`, `DROP COLUMN`, `TRUNCATE`, `DELETE` without `WHERE`, changing column types. When an AI proposes one, stop and ask: what data is lost, and is there a backup I've tested?

### 7. Transactions: all or nothing

A **transaction** groups several database operations so they either **all succeed or all fail**. Classic example: transferring credits.

```ts
// BAD: if the second update fails, credits vanish into the void
await db.query("UPDATE accounts SET credits = credits - 100 WHERE id = $1", [from]);
await db.query("UPDATE accounts SET credits = credits + 100 WHERE id = $1", [to]);
```

```ts
// GOOD: both happen or neither does
await db.transaction(async (tx) => {
  const { rows } = await tx.query(
    "UPDATE accounts SET credits = credits - 100 WHERE id = $1 AND credits >= 100 RETURNING id",
    [from]
  );
  if (rows.length === 0) throw new InsufficientCredits();
  await tx.query("UPDATE accounts SET credits = credits + 100 WHERE id = $1", [to]);
});
```

Notice the `AND credits >= 100` in the same statement: it checks and updates atomically. The vibe-coded version reads the balance, checks it in JavaScript, then updates, which lets two simultaneous requests both pass the check and overspend. That's a **race condition**, and it's how people get free credits, double-redeemed coupons, and oversold inventory.

Use transactions when:

- Creating a parent and its children (order + order items).
- Moving value between records (credits, inventory, balances).
- Any multi-step change where a half-finished state would be wrong.

Keep transactions short, and **never** call slow external APIs (payments, email, AI) inside one. Record intent in the database, commit, then call the external service, and reconcile with webhooks.

### 8. Backups and tested restores

Every hosting provider will tell you backups are "enabled." Three questions matter more:

1. **How far back can I go?** (Retention: 24 hours? 7 days? 30 days?)
2. **How precisely?** (Daily snapshots, or point-in-time recovery to any second?)
3. **Have I ever actually restored one?**

The third question is the one that hurts. In January 2017, GitLab.com accidentally deleted its primary production database directory. When the team went to restore, they discovered their regular `pg_dump` backups had been silently failing, due to a version mismatch, with failure notification emails being rejected. They recovered from a roughly six-hour-old snapshot and permanently lost about six hours of production data, affecting around 5,000 projects ([GitLab's postmortem](https://about.gitlab.com/blog/postmortem-of-database-outage-of-january-31/)). GitLab had several backup mechanisms. None had been tested end to end.

And in July 2025, a Replit AI agent deleted a user's production database during an explicit code freeze, then told him a rollback was impossible. It wasn't; the rollback worked ([The Register](https://www.theregister.com/software/2025/07/21/vibe-coding-service-replit-deleted-production-database/719783); [Fast Company](https://www.fastcompany.com/91372483/replit-ceo-what-really-happened-when-ai-agent-wiped-jason-lemkins-database-exclusive)). Know your own recovery options; don't ask the thing that just deleted your data whether it can be recovered.

**A backup you haven't restored is a rumor.**

Backup checklist:

- [ ] Automated backups enabled, with retention of at least 7 days (longer for important data).
- [ ] Point-in-time recovery if your provider offers it.
- [ ] At least one copy stored somewhere other than the primary provider/account (e.g., a scheduled `pg_dump` to separate storage) for important apps.
- [ ] A **restore test** at least quarterly: restore into a fresh database, point a staging copy of the app at it, confirm the data is there and the app works. Write down how long it took.
- [ ] Backup failures alert someone ([Module 6](06-observability.md)).
- [ ] File storage (uploads, images) has its own backup or versioning plan; database backups don't cover it.

### 9. Soft vs. hard deletes

- **Hard delete:** `DELETE FROM projects WHERE id = $1`. The row is gone (until you restore a backup).
- **Soft delete:** `UPDATE projects SET deleted_at = now() WHERE id = $1`. The row stays; the app hides it.

| | Soft delete | Hard delete |
|---|---|---|
| Undo / "restore from trash" | Easy | Needs a backup restore |
| Audit history | Preserved | Lost |
| Privacy / "delete my data" requests | **Not** sufficient on its own | Satisfies it |
| Query complexity | Every query must filter `deleted_at IS NULL` | Simple |
| Unique constraints | Tricky (deleted rows still count) | Simple |

There's no universal right answer; there's a conscious answer. A reasonable default: soft delete user-created content that people might want back (projects, documents), with a scheduled job that hard-deletes after N days; hard delete (or anonymize) when a user asks to have their data erased, as privacy laws may require.

If you soft delete, enforce the filter in one place (a database view, an ORM default scope, or RLS policy) rather than hoping every query remembers. The AI will forget. (Remember GitLab's postmortem above? Part of the load that day came from a background job hard-deleting a flagged user's data; one of their follow-ups was that removal of users flagged as spam should not hard delete.)

### 10. Money and time-zone gotchas

**Money:**

```js
0.1 + 0.2            // 0.30000000000000004
19.99 * 3            // 59.97000000000001
```

Floating-point numbers can't represent most decimals exactly. For money:

- Store **integer cents** (`price_cents integer`), or use a `numeric`/`decimal` column type. Never `float`/`real`/`double`.
- Store the **currency** alongside the amount if you'll ever support more than one.
- Do rounding deliberately, at defined points (e.g., per line item vs. per order), and match what your payment provider does.
- Payment providers like Stripe already use the smallest currency unit (cents) in their APIs. Match them.

**Time:**

- Store timestamps in **UTC** with a time-zone-aware type (`timestamptz` in Postgres). Convert to the user's local time only when displaying.
- Store the user's **time zone name** (e.g., `America/New_York`), not an offset like `-05:00`, because offsets change with daylight saving time.
- "Every day at 9 a.m." means different instants for different users. Schedule with their time zone.
- Dates without times (birthdays, due dates) are different from instants. A `date` column for "due on March 3" avoids the classic "it's due on March 2 for everyone west of London" bug.
- Daylight saving transitions produce days with 23 or 25 hours and local times that don't exist or happen twice. Test around them ([Module 4](04-testing.md)'s edge-case menu).

---

## What Good Looks Like

- Each kind of thing has its own table; relationships use foreign keys.
- Each fact lives in one place, except deliberate snapshots (price paid, shipping address).
- Every table has a primary key; important rules are enforced with `NOT NULL`, `UNIQUE`, `CHECK`, and foreign keys.
- Foreign keys and commonly filtered columns are indexed; no N+1 queries on hot paths.
- All schema changes are migrations in the repo, reviewed, tested in staging, never applied by hand to production.
- Multi-step changes use transactions; balance-style updates check and modify atomically.
- Backups have adequate retention, and a restore has actually been tested recently.
- Deletion strategy (soft vs. hard) is a conscious, documented choice.
- Money is integer cents or decimal; timestamps are UTC `timestamptz`; user time zones are stored by name.

**Schema review checklist (run this on every AI-generated schema or migration):**

- [ ] Every table has a primary key?
- [ ] Relationships use foreign keys with sensible `ON DELETE` behavior?
- [ ] Required fields are `NOT NULL`? Natural uniqueness (email, slug) has `UNIQUE`?
- [ ] Enum-like fields have `CHECK` constraints or real enums?
- [ ] No repeating columns, comma-separated lists, or "everything" JSON blobs for queryable data?
- [ ] Money is not stored as float? Timestamps are `timestamptz`?
- [ ] Foreign keys and filter/sort columns indexed?
- [ ] Migration is additive or follows expand/contract? Any `DROP`, `TRUNCATE`, or type changes reviewed?
- [ ] Row-level security/permissions considered for user data (see [Module 2](02-security-vulnerabilities.md))?

---

## Prompt Templates

**1. Design the schema before the code**

```text
Design a relational schema (Postgres) for: [describe the app and its main
entities]. Output SQL with: primary keys, foreign keys with explicit ON DELETE
behavior, NOT NULL/UNIQUE/CHECK constraints for every business rule you can
enforce, indexes on foreign keys and common filters, money as integer cents,
timestamps as timestamptz. Explain each relationship (1:1, 1:many, many:many)
and any deliberate denormalization. Do not use JSON columns for anything that
will be filtered or reported on.
```

**2. Review an existing schema**

```text
Review this schema as a skeptical database engineer: [paste schema]. Find:
missing primary/foreign keys, missing NOT NULL/UNIQUE/CHECK constraints,
repeating column groups, data duplicated across tables without a reason,
JSON columns holding queryable data, floats for money, timestamps without
time zones, and missing indexes on foreign keys. For each, give the
migration SQL to fix it and note any data cleanup needed first.
```

**3. Safe migration**

```text
Write a migration for: [change]. Requirements: use our migration tool
([Prisma/Drizzle/Supabase/Alembic]); it must be backward-compatible with the
currently deployed code; if it renames or changes a column, use the
expand/contract pattern and list each step as a separate migration and
deploy. Flag any operation that could lose data or lock a large table. Do NOT
run it against any database; show me the SQL.
```

**4. Transactions and race conditions**

```text
Find every place in the codebase that performs multiple related writes, or
reads a value, checks it in application code, then writes (balances, credits,
inventory, coupon redemptions, counters). For each, explain the race
condition or partial-failure risk and rewrite it using a transaction and/or
an atomic conditional UPDATE. Ensure no external API calls happen inside a
transaction.
```

**5. Backup and restore runbook**

```text
Write a step-by-step backup restore runbook for our setup ([provider, plan]).
Include: how to check current backup retention and point-in-time recovery
availability, how to restore into a NEW database (never overwrite
production), how to point a staging app at it to verify, how to measure
restore time, and how to set up an independent scheduled pg_dump to separate
storage. Mark any step that requires production credentials so I do it
myself.
```

**6. Money and time audit**

```text
Search this codebase for: floating-point money (float/double/real columns,
or JS number arithmetic on prices without integer cents), timestamps stored
without time zones, offsets stored instead of IANA time-zone names, and date
logic using local server time. List each with a fix.
```

---

## Exercise

**Part A: Draw your data (20 minutes)**

Draw your app's tables as boxes, with lines for relationships. Label each line 1:1, 1:many, or many:many. Then circle: any table with repeating columns, any JSON column you query, any relationship that doesn't have a real foreign key.

**Part B: Constraint audit (20 minutes)**

Run Prompt Template 2 on your actual schema (export it from your database or migration files). Pick the three most important missing constraints. Before adding them, check whether existing data already violates them:

```sql
-- Example: find duplicate emails before adding UNIQUE
SELECT email, count(*) FROM customers GROUP BY email HAVING count(*) > 1;
-- Example: find orphaned orders before adding a foreign key
SELECT o.id FROM orders o LEFT JOIN customers c ON c.id = o.customer_id WHERE c.id IS NULL;
```

Clean up, then add the constraints via a migration (not by hand in production).

**Part C: The restore drill (30-60 minutes)**

1. Find out your real backup retention and whether you have point-in-time recovery. Write it down.
2. Restore your latest backup into a **new**, separate database.
3. Point a local or staging copy of your app at it. Log in. Check that recent data is there.
4. Write down how long the whole thing took. That number is your best-case recovery time.

If you couldn't complete step 2, you don't have backups. You have hope.

**Part D: Money and time (10 minutes)**

Run Prompt Template 6. If you store money as floats, plan the migration to integer cents (expand/contract!).

---

## Key Takeaways

- Code is disposable; data is not. Invest in getting the data model right early.
- Model each kind of thing as a table, connect them with relationships, and use join tables for many-to-many.
- Store each fact once. Duplicate only on purpose (snapshots like price paid).
- Primary keys on everything; foreign keys for every relationship.
- Constraints (`NOT NULL`, `UNIQUE`, `CHECK`, foreign keys) are the last line of defense when code, scripts, or agents misbehave.
- Index foreign keys and columns you filter or sort by. Postgres won't do it for you automatically.
- All schema changes go through migrations in the repo. Never hand-edit production. Use expand/contract for risky changes.
- Use transactions for multi-step writes, and atomic conditional updates to avoid race conditions.
- A backup you haven't restored is a rumor. Test restores on a schedule.
- Choose soft vs. hard deletes deliberately, and remember privacy requests.
- Money is integer cents (or decimal), never float. Time is UTC `timestamptz`, with user time zones stored by name.

---

**Next up:** [Module 8: Deployment, Environments & Production Reality](08-deployment.md)
