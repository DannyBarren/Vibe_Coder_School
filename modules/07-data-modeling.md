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
*Illustrative composite.*

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

And in July 2025, a Replit AI agent deleted a user's production database during a code freeze, then claimed a rollback was impossible; it wasn't ([Module 1 has the full timeline](01-security-reality.md#29-anatomy-of-an-incident-2-replit-and-the-saastr-database)). Know your own recovery options; don't ask the thing that just deleted your data whether it can be recovered.

**A backup you haven't restored is a rumor.**

Backup checklist:

- [ ] Automated backups enabled, with retention of at least 7 days (longer for important data).
- [ ] Point-in-time recovery if your provider offers it.
- [ ] At least one copy stored somewhere other than the primary provider/account (e.g., a scheduled `pg_dump` to separate storage) for important apps.
- [ ] A **restore drill** at least quarterly. [Section 14](#14-a-backup-restore-drill-you-can-actually-run) has a script and the full procedure.
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
4.35 * 100           // 434.99999999999994  (so Math.floor gives 434 cents, not 435)
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

### 11. Worked example: from a JSON blob to real tables

*(Illustrative, and extremely common.)* Version 1 of a project tracker, as the AI first built it: each project stores its tasks inside a JSON column.

```sql
-- v1: "just put it in JSON"
CREATE TABLE projects (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id    uuid NOT NULL,
  name        text NOT NULL,
  data        jsonb NOT NULL DEFAULT '{}'::jsonb,  -- {"tasks":[{"title","done","due"}], "color": ...}
  created_at  timestamptz NOT NULL DEFAULT now()
);
```

It worked for months. Then came the requests: "show all overdue tasks across my projects," "let two people edit tasks at once without overwriting each other," "assign tasks to teammates." Every one of those is painful in a JSON blob: no constraints, no indexes on individual tasks, and every edit rewrites the whole array (so two simultaneous edits means one silently wins).

Time to evolve the schema, using expand/contract so the live app never breaks.

**Step 0: Pre-flight.** Before migrating, find the data your migration can't handle. Real JSON blobs written by months of AI-generated code are never as clean as you think:

```sql
-- Pre-flight: find data the backfill can't convert (run BEFORE migrating)
SELECT p.id, p.name, t.value->>'due' AS bad_due
FROM projects p
CROSS JOIN LATERAL jsonb_array_elements(
  CASE WHEN jsonb_typeof(p.data->'tasks') = 'array' THEN p.data->'tasks' ELSE '[]'::jsonb END
) AS t(value)
WHERE t.value->>'due' IS NOT NULL
  AND t.value->>'due' !~ '^\d{4}-\d{2}-\d{2}$';

-- Projects whose "tasks" exists but isn't an array at all
SELECT id, name, jsonb_typeof(data->'tasks') AS tasks_type
FROM projects
WHERE data ? 'tasks' AND jsonb_typeof(data->'tasks') <> 'array';

-- Tasks whose "done" isn't a real JSON boolean (e.g. "yes", 1, or "true" as a string)
SELECT p.id, p.name, t.value->'done' AS bad_done
FROM projects p
CROSS JOIN LATERAL jsonb_array_elements(
  CASE WHEN jsonb_typeof(p.data->'tasks') = 'array' THEN p.data->'tasks' ELSE '[]'::jsonb END
) AS t(value)
WHERE jsonb_typeof(t.value->'done') NOT IN ('boolean', 'null');
```

On a realistic dataset this finds things like `"due": "next friday"`, `"done": "yes"`, and a project where `tasks` is an object instead of an array. Decide what to do with each (here: unparseable dates become NULL, anything that isn't a real boolean counts as not done, the odd project gets fixed by hand) *before* you migrate, not halfway through.

**Step 1: Expand.** Add the new table. Purely additive, so the running code doesn't notice:

```sql
-- 0007_create_tasks.sql  (expand: purely additive, safe while old code runs)
CREATE TABLE tasks (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id  uuid NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  title       text NOT NULL CHECK (length(title) BETWEEN 1 AND 500),
  done        boolean NOT NULL DEFAULT false,
  due_date    date,
  position    integer NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_tasks_project_position ON tasks (project_id, position);
```

**Step 2: Backfill**, in a transaction, with a verification that rolls everything back if the numbers don't match:

```sql
-- 0008_backfill_tasks.sql  (copy JSON tasks into the new table, all-or-nothing)
BEGIN;

INSERT INTO tasks (project_id, title, done, due_date, position)
SELECT
  p.id,
  COALESCE(NULLIF(trim(t.value->>'title'), ''), '(untitled)'),
  CASE WHEN jsonb_typeof(t.value->'done') = 'boolean'
       THEN (t.value->'done')::boolean ELSE false END,  -- "yes", 1, missing → false (found in pre-flight)
  CASE WHEN t.value->>'due' ~ '^\d{4}-\d{2}-\d{2}$'
       THEN (t.value->>'due')::date END,          -- unparseable dates become NULL (found in pre-flight)
  t.position::int
FROM projects p
CROSS JOIN LATERAL jsonb_array_elements(
  CASE WHEN jsonb_typeof(p.data->'tasks') = 'array' THEN p.data->'tasks' ELSE '[]'::jsonb END
) WITH ORDINALITY AS t(value, position);

-- Verify inside the transaction: counts must match, or we roll back
DO $$
DECLARE in_json bigint; in_table bigint;
BEGIN
  SELECT COALESCE(sum(jsonb_array_length(data->'tasks')), 0) INTO in_json
  FROM projects WHERE jsonb_typeof(data->'tasks') = 'array';
  SELECT count(*) INTO in_table FROM tasks;
  IF in_json <> in_table THEN
    RAISE EXCEPTION 'Backfill mismatch: % tasks in JSON, % in table', in_json, in_table;
  END IF;
END $$;

COMMIT;
```

A nice side effect of the built-in check: if someone (or some agent) accidentally runs the backfill twice, the counts no longer match, the transaction raises an error, and nothing is inserted. The guard rail is part of the migration itself. (The date regex only checks the *shape* `YYYY-MM-DD`; an impossible date like `2026-13-45` would still fail the cast, which is exactly what the pre-flight step is for.)

**Step 3: Dual-write, then switch reads.** Deploy code that writes tasks to *both* places, so nothing created during the transition is lost. If new tasks were created between the backfill and this deploy, re-sync them (or do the backfill right after the dual-write deploy). Then deploy code that reads from the `tasks` table. Watch errors and compare a few projects by hand.

**Step 4: Stop writing JSON tasks.** Once reads have been on the new table for a while without problems, remove the JSON write path.

**Step 5: Contract.** Only now remove the old data, and only what you actually migrated:

```sql
-- 0009_drop_json_tasks.sql  (contract: only after the new code has been live and verified)
-- Only remove arrays we actually migrated; anything odd stays for a human to look at.
UPDATE projects SET data = data - 'tasks'
WHERE jsonb_typeof(data->'tasks') = 'array';
```

Note the `WHERE`: without it, this line would strip `tasks` from *every* project, silently destroying the odd non-array data the pre-flight found. "Delete everything that matches a key" is the kind of line an AI writes without blinking. Read it twice. Take a backup first.

Five steps instead of one "migrate everything" prompt. Each step is reversible until the last, and the app works throughout.

### 12. ORM vs. raw SQL

AI tools will pick an ORM (Prisma, Drizzle, TypeORM, SQLAlchemy, Django ORM, ActiveRecord) or raw SQL more or less at random. Both are fine. Mixing four approaches in one codebase is not.

| | ORM / query builder | Raw SQL |
|---|---|---|
| **Good at** | Type-safe CRUD, migrations tooling, relationships, less boilerplate | Complex reports, bulk operations, window functions, performance tuning |
| **Watch out for** | N+1 queries hidden behind innocent-looking property access; generated migrations that drop and recreate columns; "magic" you don't understand | String concatenation (SQL injection; see [Module 2](02-security-vulnerabilities.md)); no type safety unless you add it; hand-mapping rows |
| **AI pitfalls** | Hallucinated ORM methods and options that don't exist in your version | Mixing dialects (MySQL syntax in Postgres), forgetting parameters |

A sensible default for most small apps:

- Use **one** ORM or query builder for everyday reads and writes, and its migration tool for schema changes.
- Drop to **parameterized** raw SQL for the handful of queries where it's clearer or faster: reports, backfills, bulk updates. Keep them in the data layer ([Module 5](05-architecture.md)), not sprinkled in route handlers.
- Learn enough SQL to read what your ORM generates. Most ORMs can log their queries; turn that on in development once in a while and look. That's how you spot the N+1 that makes your dashboard take six seconds.

```ts
// BAD: string-built SQL (injection risk, and breaks on names like O'Brien)
await db.query(`SELECT * FROM customers WHERE email = '${email}'`);

// GOOD: parameterized (the driver handles escaping)
await db.query("SELECT id, name FROM customers WHERE email = $1", [email]);
```

### 13. Seeding and test data

**Seed data** is the known, fake data you load into local, preview, and test databases so the app has something to show and your tests have something to work with.

```sql
-- seed.sql — local/dev/test only. Idempotent: running it twice changes nothing.
INSERT INTO customers (email, name) VALUES
  ('alice@example.com', 'Alice Example'),
  ('bob@example.com',   'Bob Example')
ON CONFLICT (email) DO NOTHING;

-- Fixed IDs make seed rows easy to reference in tests and safe to re-insert
INSERT INTO products (id, name, price_cents, active) VALUES
  ('00000000-0000-0000-0000-000000000101', 'Starter Kit',      1999, true),
  ('00000000-0000-0000-0000-000000000102', 'Family Kit',       4999, true),
  ('00000000-0000-0000-0000-000000000103', 'Discontinued Kit',  999, false)  -- edge case on purpose
ON CONFLICT (id) DO NOTHING;
```

Seed rules:

- **Idempotent.** Running the seed twice must not create duplicates (`ON CONFLICT DO NOTHING`, fixed IDs for reference data).
- **Include edge cases on purpose:** an inactive product, an expired coupon, a user with no orders, a name with an apostrophe, a long name with emoji. Seeds are the cheapest edge-case tests you'll ever write.
- **Never** seed production with fake users, and never run a seed script with a production connection string. Put a guard at the top of seed scripts that refuses to run if the environment is production.
- **No real customer data in seeds or test databases.** If you need realistic volume, generate it (libraries like Faker exist for most languages) or use properly anonymized data.
- Reference data that production genuinely needs (plans, countries, permission roles) belongs in **migrations**, not seeds, so every environment gets it the same way.

### 14. A backup restore drill you can actually run

Section 8 said "a backup you haven't restored is a rumor." Here's a script that turns the rumor into evidence. It dumps a database, restores it into a separate scratch database, compares row counts, and times the whole thing:

```bash
#!/usr/bin/env bash
# restore-drill.sh — dump a database, restore it into a NEW scratch database, and check it.
set -euo pipefail

: "${SOURCE_URL:?Set SOURCE_URL to a READ-ONLY connection string for the database to back up}"
: "${DRILL_URL:?Set DRILL_URL to an EMPTY scratch database (never production)}"
TABLES="${TABLES:-customers orders order_items}"   # tables to compare
NEWEST_TABLE="${NEWEST_TABLE:-orders}"             # table with a created_at column

if [[ "$DRILL_URL" == "$SOURCE_URL" ]]; then
  echo "Refusing: DRILL_URL is the same as SOURCE_URL." >&2
  exit 1
fi

STAMP=$(date -u +%Y%m%dT%H%M%SZ)
FILE="backup-$STAMP.dump"
start=$(date +%s)

echo "1/4 Dumping source..."
pg_dump --format=custom --no-owner --no-privileges --file="$FILE" "$SOURCE_URL"

echo "2/4 Restoring into the drill database..."
pg_restore --no-owner --no-privileges --exit-on-error --dbname="$DRILL_URL" "$FILE"

echo "3/4 Comparing row counts (a busy database may drift slightly; big gaps are a problem)..."
fail=0
for table in $TABLES; do
  src=$(psql "$SOURCE_URL" -tAc "SELECT count(*) FROM $table")
  dst=$(psql "$DRILL_URL" -tAc "SELECT count(*) FROM $table")
  printf '  %-14s source=%-8s restored=%s\n' "$table" "$src" "$dst"
  [[ "$src" == "$dst" ]] || fail=1
done

echo "4/4 Newest record in the restored copy:"
psql "$DRILL_URL" -tAc "SELECT max(created_at) FROM $NEWEST_TABLE"

echo "Drill took $(( $(date +%s) - start ))s. Backup file: $FILE ($(du -h "$FILE" | cut -f1))"
[[ $fail -eq 0 ]] && echo "PASS" || echo "FAIL: row counts differ"
exit $fail
```

Usage:

```bash
SOURCE_URL="postgres://readonly_user:...@host/app" \
DRILL_URL="postgres://user:...@localhost/restore_drill" \
TABLES="customers orders order_items" NEWEST_TABLE=orders \
./restore-drill.sh
```

Important details:

- Use a **read-only** user for `SOURCE_URL`, and run the script yourself, not via an AI agent ([Module 8](08-deployment.md)).
- `DRILL_URL` must be an **empty scratch database**. The script refuses to restore into the source, but it can't know whether some other URL is production. Double-check.
- Your `pg_dump` version must be the **same or newer** than the server's major version. A version mismatch is exactly what silently broke GitLab's backups in 2017 (section 8).
- The dump file contains all your data, including personal data. Store and delete it accordingly.
- This tests *your own* dump. Also practice your provider's restore (snapshots or point-in-time recovery) at least once, because that's what you'll use in a real emergency. Then point a staging copy of the app at the restored database and click around.
- Run it on a schedule (monthly or quarterly), and write down the time it took. That's your real-world recovery time.

### 15. Multi-tenant data: keeping customers' data apart

If your app has **organizations, teams, or workspaces** (B2B SaaS, agencies, anything where several users share data), you're building a **multi-tenant** app. The most common approach for small and mid-size apps is shared tables with a `tenant_id` (often called `org_id` or `workspace_id`) on every tenant-owned row.

```sql
CREATE TABLE organizations (
  id    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name  text NOT NULL
);

CREATE TABLE memberships (
  org_id   uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  user_id  uuid NOT NULL,
  role     text NOT NULL CHECK (role IN ('owner', 'admin', 'member')),
  PRIMARY KEY (org_id, user_id)
);

CREATE TABLE org_projects (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id      uuid NOT NULL REFERENCES organizations(id),
  slug        text NOT NULL,
  name        text NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now(),
  UNIQUE (org_id, slug),   -- slugs are unique per organization, not globally
  UNIQUE (org_id, id)      -- lets child tables reference (org_id, id) together
);
CREATE INDEX idx_org_projects_org_created ON org_projects (org_id, created_at DESC);

CREATE TABLE org_tasks (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id      uuid NOT NULL,
  project_id  uuid NOT NULL,
  title       text NOT NULL,
  -- A task can only point at a project in the SAME organization:
  FOREIGN KEY (org_id, project_id) REFERENCES org_projects (org_id, id) ON DELETE CASCADE
);
CREATE INDEX idx_org_tasks_org_project ON org_tasks (org_id, project_id);
```

The rules that make it work:

- **Every tenant-owned table gets `org_id NOT NULL`.** Even child tables like tasks, so every query can filter by tenant directly.
- **Uniqueness is per tenant:** `UNIQUE (org_id, slug)`, not `UNIQUE (slug)`. Two companies can both have a project called "launch."
- **Composite foreign keys** (`(org_id, project_id) REFERENCES org_projects (org_id, id)`) make it *impossible* for a task in one organization to point at another organization's project. The database rejects the cross-tenant insert outright.
- **Index by tenant first:** `(org_id, created_at)`, `(org_id, project_id)`. Nearly every query filters by tenant.
- **Every query is scoped by tenant**, enforced in one place: a repository layer that requires `orgId` as an argument, and ideally database-level Row Level Security as a backstop. How to set up and test RLS properly is covered in [Module 2](02-security-vulnerabilities.md); the four-case auth tests from [Module 4](04-testing.md) apply per tenant too.
- **The tenant comes from the session**, via a membership check, never from a request parameter the user can change.

Other models exist (a separate schema or a separate database per tenant). They give stronger isolation and are sometimes required by enterprise customers or regulation, at the cost of much more operational work. If someone asks for them, that's a good moment for an ADR ([Module 5](05-architecture.md)) and possibly an experienced engineer ([Module 9](09-tech-debt.md)).

### 16. Schema walkthroughs: three apps you might actually build

Theory is easier to apply when you've seen it on real shapes. Here are three of the most common vibe-coded app types, with the decisions that matter called out. (Each is a starting point, not a final design.)

#### A. SaaS with teams

You've already seen it: section 15 *is* this schema (organizations, memberships with roles, tenant-scoped projects and tasks with composite foreign keys). Two additions most SaaS apps need soon:

- **Invites** as their own table (`org_id`, `email`, `role`, `token`, `status`, `expires_at`), with a partial unique index so the same email can't have two pending invites to one org: `CREATE UNIQUE INDEX ON invites (org_id, lower(email)) WHERE status = 'pending';`. That index is the database-level backstop for the double-click problem from [Module 5](05-architecture.md)'s worked refactor.
- **Billing state on the organization, not the user** (`plan`, `stripe_customer_id`, `seats`). The first time a company asks "can we move our subscription to a different admin?", you'll be glad.

#### B. A marketplace

Buyers purchase listings from many sellers, sometimes several sellers in one checkout. The traps are about **money and history**:

```sql
-- Marketplace: buyers order from many sellers in one checkout
CREATE TABLE sellers (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id          uuid NOT NULL UNIQUE,            -- one seller profile per user
  display_name     text NOT NULL,
  payout_account   text,                            -- e.g. a Stripe Connect account ID
  created_at       timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE listings (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  seller_id     uuid NOT NULL REFERENCES sellers(id),
  title         text NOT NULL,
  price_cents   integer NOT NULL CHECK (price_cents > 0),
  currency      char(3) NOT NULL DEFAULT 'USD',
  stock         integer NOT NULL CHECK (stock >= 0),  -- can never go negative
  status        text NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'active', 'archived')),
  created_at    timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_listings_seller ON listings (seller_id);

CREATE TABLE orders (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  buyer_id      uuid NOT NULL,
  status        text NOT NULL DEFAULT 'pending'
                CHECK (status IN ('pending', 'paid', 'refunded', 'cancelled')),
  total_cents   integer NOT NULL CHECK (total_cents >= 0),
  created_at    timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_orders_buyer_created ON orders (buyer_id, created_at DESC);

CREATE TABLE order_items (
  id                uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id          uuid NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  listing_id        uuid NOT NULL REFERENCES listings(id),
  seller_id         uuid NOT NULL REFERENCES sellers(id),  -- copied: who gets paid for this line
  title_snapshot    text NOT NULL,                         -- what the buyer saw at checkout
  unit_price_cents  integer NOT NULL CHECK (unit_price_cents > 0),
  quantity          integer NOT NULL CHECK (quantity > 0)
);
CREATE INDEX idx_order_items_order ON order_items (order_id);
CREATE INDEX idx_order_items_seller ON order_items (seller_id);

CREATE TABLE payouts (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  seller_id     uuid NOT NULL REFERENCES sellers(id),
  amount_cents  integer NOT NULL CHECK (amount_cents > 0),
  status        text NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'paid', 'failed')),
  period_start  date NOT NULL,
  period_end    date NOT NULL,
  CHECK (period_end > period_start),
  UNIQUE (seller_id, period_start)   -- you can't pay the same seller twice for one period
);
```

The decisions worth noticing:

- **`order_items` copies the price and title** at checkout (`unit_price_cents`, `title_snapshot`). If a seller changes the price tomorrow, last month's orders must still show what the buyer actually paid. Joining to the live `listings.price_cents` is one of the most common marketplace bugs: suddenly every historical receipt is wrong.
- **`order_items.seller_id`** makes "what do we owe each seller?" a simple `GROUP BY seller_id` instead of a join puzzle.
- **`stock CHECK (stock >= 0)`** means two buyers racing for the last item can't both win. Decrement with `UPDATE listings SET stock = stock - 1 WHERE id = $1 AND stock >= 1 RETURNING stock`; zero rows back means it's sold out. Even if some code path forgets the `WHERE`, the constraint rejects the negative stock.
- **`UNIQUE (seller_id, period_start)` on payouts** makes "we accidentally paid everyone twice because the payout job ran twice" impossible. That one constraint can save a small marketplace thousands of dollars on its worst day.
- **Statuses are constrained** to known values, so a typo like `'payed'` can't sneak in and make orders invisible to every query filtering on `'paid'`.

#### C. A booking app

Salons, tutors, meeting rooms, rental gear: anything with a calendar. The hard part is the one every demo skips: **two people booking the same slot at the same time.** See section 17 for the schema and the concurrency fix, which is the main event here. Two more booking decisions:

- **Store bookings as real instants** (`timestamptz`, or a `tstzrange` of them), and store the **resource's time zone** separately to display "2:00 PM" correctly for the salon, whatever time zone the customer's laptop is in.
- **Cancel by status, don't delete.** A cancelled booking is history you'll need for refunds, no-show policies, and disputes (section 9).

### 17. The double-booking bug, and the fix that actually works

*(Illustrative composite.)* A yoga studio's booking app, built in a weekend. The booking endpoint does what you'd do by hand:

```ts
// BAD: check, then insert. Two requests can both pass the check.
const clash = await db.booking.findFirst({
  where: { resourceId, start: { lt: end }, end: { gt: start }, status: "confirmed" },
});
if (clash) throw new SlotTakenError();
await db.booking.create({ data: { resourceId, customerId, start, end } });
```

It passes every test, because tests send one request at a time. Then the studio announces a popular workshop at noon. Two customers tap "Book" within the same 50 milliseconds. Both requests run the `findFirst`, both see no clash, both insert. Two people show up for one spot; one of them took a day off work for it. Multiply by every popular slot, and the owner spends Saturday issuing refunds and apologies. This is the same read → decide → write race from [Module 3](03-reading-code.md), and no amount of application code "checking harder" fixes it, because the gap between the check and the insert is always there.

**The fix: make overlapping bookings impossible in the database.** Postgres has a constraint built for exactly this, the **exclusion constraint**: "no two rows may have the same resource *and* overlapping time ranges."

```sql
-- Booking: rooms, chairs, tutors, rental cars... anything with a calendar
CREATE EXTENSION IF NOT EXISTS btree_gist;  -- lets one GiST index mix "=" and "overlaps"

CREATE TABLE resources (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name       text NOT NULL,
  time_zone  text NOT NULL DEFAULT 'America/New_York'  -- where the resource physically is
);

CREATE TABLE bookings (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  resource_id  uuid NOT NULL REFERENCES resources(id),
  customer_id  uuid NOT NULL,
  during       tstzrange NOT NULL,                     -- [start, end) as real instants
  status       text NOT NULL DEFAULT 'confirmed' CHECK (status IN ('confirmed', 'cancelled')),
  created_at   timestamptz NOT NULL DEFAULT now(),
  CHECK (lower(during) < upper(during)),
  -- No two confirmed bookings for the same resource may overlap. Enforced by the database.
  CONSTRAINT no_double_booking EXCLUDE USING gist (
    resource_id WITH =,
    during WITH &&
  ) WHERE (status = 'confirmed')
);
```

Here's what happens when two transactions try to book the same hour at the same moment. The first inserts successfully. The second **waits** for the first to finish, then fails:

```text
ERROR:  conflicting key value violates exclusion constraint "no_double_booking"
DETAIL:  Key (resource_id, during)=(..., ["2026-11-05 14:00:00-05","2026-11-05 15:00:00-05"))
         conflicts with existing key (resource_id, during)=(..., ["2026-11-05 14:00:00-05","2026-11-05 15:00:00-05")).
```

And back-to-back bookings (14:00-15:00 and 15:00-16:00) are allowed, because `[)` ranges include the start but not the end. Your application code catches the error and shows a friendly message. The error has SQLSTATE code `23P01` (exclusion violation), so you can check for it precisely:

```ts
try {
  await db.$executeRaw`
    INSERT INTO bookings (resource_id, customer_id, during)
    VALUES (${resourceId}::uuid, ${customerId}::uuid, tstzrange(${start}::timestamptz, ${end}::timestamptz, '[)'))`;
} catch (err) {
  if (isPgError(err, "23P01")) throw new SlotTakenError("Sorry, someone just booked that slot.");
  throw err;
}
```

(How you read the SQLSTATE code depends on your driver: `err.code` with node-postgres, and for Prisma raw queries it's on the error's `meta`. Write `isPgError` once in your data layer.)

**The alternative when you can't use a constraint:** lock the parent row inside a transaction, so concurrent bookings for the same resource take turns: `BEGIN; SELECT ... FROM resources WHERE id = $1 FOR UPDATE;` then check for overlaps, insert, and `COMMIT`. The second transaction waits at the `FOR UPDATE` until the first commits, and then sees its booking. It works, but every code path that books must remember to do it. The constraint can't be forgotten.

The general lesson: **any rule of the form "there must never be two X" belongs in the database** (a unique constraint, an exclusion constraint, or a check), with the app's check kept only to show a nice message early.

### 18. Money and time, in code

Section 10 listed the rules. Here they are as code you can copy, with the bugs they prevent.

**Money in JavaScript/TypeScript: integer cents, format at the edge.**

```js
// money.js — integer cents everywhere; convert only for display
export function formatMoney(cents, currency = "USD", locale = "en-US") {
  return new Intl.NumberFormat(locale, { style: "currency", currency }).format(cents / 100);
}

export function applyPercentOff(cents, percentOff) {
  return Math.round((cents * (100 - percentOff)) / 100); // one deliberate rounding step
}

// Split an amount into n parts that always add back up to the total
export function splitCents(totalCents, parts) {
  const base = Math.floor(totalCents / parts);
  const remainder = totalCents - base * parts;
  return Array.from({ length: parts }, (_, i) => base + (i < remainder ? 1 : 0));
}
```

```js
formatMoney(1999 * 3)                  // "$59.97"
applyPercentOff(1999, 10)              // 1799, displayed as "$17.99"
splitCents(10000, 3)                   // [3334, 3333, 3333], adds up to 10000
formatMoney(123456, "EUR", "de-DE")    // "1.234,56 €"

// The float version of the same split loses a cent:
(100 / 3).toFixed(2) * 3               // 99.99
```

That "lost cent" is not academic. Split a $100 group booking three ways with floats, charge each person $33.33, and you've collected $99.99. Do that across thousands of transactions and your books stop reconciling with your payment provider, and someone spends a week finding out why. (`formatMoney` divides by 100, which is right for dollars and euros; currencies like the Japanese yen have no minor unit, so a multi-currency app needs a per-currency exponent.)

**Money in Python: `Decimal`, constructed from strings.**

```python
from decimal import Decimal, ROUND_HALF_UP

print(round(2.675, 2))                     # 2.67 (!) because 2.675 is really 2.67499999...
print(Decimal("2.675").quantize(Decimal("0.01"), rounding=ROUND_HALF_UP))  # 2.68

def tax_cents(subtotal_cents: int, rate: str) -> int:
    """Tax on an amount in cents, rounded half-up to a whole cent."""
    tax = Decimal(subtotal_cents) * Decimal(rate)
    return int(tax.quantize(Decimal("1"), rounding=ROUND_HALF_UP))

print(tax_cents(1999, "0.0825"))           # 165  ($1.649175 rounds to $1.65)
```

`Decimal(2.675)` (from a float) would inherit the float's error; always build `Decimal` from strings or integers.

**Time in Postgres: store instants, convert at the edge.**

```sql
-- One stored instant, shown in two places:
SELECT timestamptz '2026-03-08 14:00:00+00' AT TIME ZONE 'America/New_York' AS new_york,
       timestamptz '2026-03-08 14:00:00+00' AT TIME ZONE 'Asia/Tokyo'       AS tokyo;
--       new_york       |        tokyo
-- 2026-03-08 10:00:00  | 2026-03-08 23:00:00
```

**The "every day at 9 a.m." bug, made visible.** Daylight saving time started in the US on March 8, 2026. Ask Postgres what instant "9 a.m. in New York" is on three consecutive days:

```sql
SET TIME ZONE 'UTC';
SELECT d::date AS day,
       (d::date + time '09:00') AT TIME ZONE 'America/New_York' AS nine_am_in_new_york
FROM generate_series(date '2026-03-07', date '2026-03-09', interval '1 day') AS d;
--     day     |  nine_am_in_new_york
-- 2026-03-07  | 2026-03-07 14:00:00+00
-- 2026-03-08  | 2026-03-08 13:00:00+00
-- 2026-03-09  | 2026-03-09 13:00:00+00
```

A reminder job hardcoded to run at 14:00 UTC sends "your 9 a.m. reminder" at 10 a.m. for the whole summer. Schedule by the user's time zone name, and compute the instant per day.

**Why `timestamp` (without time zone) bites:** it stores the wall-clock text and forgets where it came from.

```sql
SET TIME ZONE 'UTC';
CREATE TEMP TABLE clocks (naive timestamp, aware timestamptz);
INSERT INTO clocks VALUES ('2026-10-17 18:30:00', '2026-10-17 18:30:00');
SET TIME ZONE 'America/New_York';
SELECT naive, aware FROM clocks;
--        naive        |         aware
-- 2026-10-17 18:30:00 | 2026-10-17 14:30:00-04
```

The `timestamptz` value correctly says "this happened at 2:30 p.m. New York time." The `timestamp` value still says 18:30, and nothing in the database knows it was UTC. Move servers, change a session setting, or let a second service write to the table, and your timestamps silently mean different things.

**Time in JavaScript: format with the user's zone.**

```js
const instant = new Date("2026-03-08T14:00:00Z"); // what the API returns (UTC)
const show = (timeZone) =>
  new Intl.DateTimeFormat("en-US", { timeZone, dateStyle: "medium", timeStyle: "short" }).format(instant);

show("America/New_York");    // "Mar 8, 2026, 10:00 AM"
show("America/Los_Angeles"); // "Mar 8, 2026, 7:00 AM"
show("Asia/Tokyo");          // "Mar 8, 2026, 11:00 PM"
```

Store the user's time zone name (from their profile, or `Intl.DateTimeFormat().resolvedOptions().timeZone` in the browser as a default), and pass it explicitly. Never let "whatever time zone the server happens to be in" decide what your users see.

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
2. Run the drill from [section 14](#14-a-backup-restore-drill-you-can-actually-run) for real, then point a staging copy of your app at the restored database and check that recent data is there. Write down the total time: that's your best-case recovery time.

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

**Next up:** [Module 8: Deployment, Environments & Production Reality](08-deployment.md) · [Back to course outline](../course-outline.md)
