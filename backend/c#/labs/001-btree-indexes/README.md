---
title: Speeding up search with B-tree indexes
track: backend/c#
difficulty: beginner
duration: 4h
requires: []
tags: [postgresql, indexes, btree, explain, performance, npgsql, docker]
---

# Speeding up search with B-tree indexes

## The problem

You have just joined the team behind **ShopLane**, a small online shop with
about 200 000 customers and 2 million orders.

Two complaints landed in your first week:

> **Customer support:** "When a customer calls, we look up their orders by
> email. The page freezes for a moment on every search. During the Black
> Friday rush it took several seconds."

> **Finance:** "Our dashboard shows *how many orders were cancelled this
> quarter*. Every widget on it is slow, and it reloads every minute."

The database has only primary keys — nobody ever added an index. In this lab
you will build a copy of the shop from scratch, reproduce both slow queries,
prove *why* they are slow, and make them fast with B-tree indexes. Then you
will find out what those indexes cost.

## Learning goals

After this lab you will be able to:

- Run PostgreSQL in Docker and talk to it from C# with Npgsql
- Load millions of rows quickly and repeatably
- Measure query latency from C# properly (warm-up, median, p95)
- Read `EXPLAIN (ANALYZE, BUFFERS)` output and spot a sequential scan, a
  filter that throws rows away, and the number of pages a query touched
- Choose the right column(s) for a B-tree index, including the **column order**
  of a composite index
- Explain why PostgreSQL does **not** index foreign keys for you
- Describe what indexes cost: disk space and slower writes

## Before you start

This is the first lab of the track, so nothing needs to exist yet. You need
the tools listed in the [track README](../README.md) and your own branch
`trainee/<your-name>`.

All your code goes into `backend/c#/workspace/`.

## Tasks

Do the tasks in order. After each one, ask the mentor for a review and wait
for **Approved** before you move on.

---

### Task 1 — PostgreSQL in Docker

**Goal:** have a PostgreSQL database you can start, stop, and throw away with
one command.

**What to do**

- Create the `workspace/` directory.
- Describe a PostgreSQL 17 service in a Docker Compose file inside the
  workspace.
- Requirements:
  - pin the exact image major version (no `latest`);
  - a dedicated database, user, and password for the shop (not `postgres`);
  - data survives `docker compose down` / `up` (but can be wiped on purpose);
  - a health check, so you can wait until the database is ready;
  - the host port must not clash with a PostgreSQL you may already have.

**Done when**

- `docker compose up -d --wait` reports the service as *healthy*;
- you can open `psql` *inside the container* and `SELECT version();` shows 17.x.

**Review:** your Compose file and the `SELECT version();` output.

---

### Task 2 — A console app that talks to the database

**Goal:** a .NET console app in the workspace that can connect to the shop
database.

**What to do**

- Create a .NET console project in the workspace (call it `OrderSearch` or
  similar) and add the **Npgsql** driver.
- The app must read the connection string from configuration or an
  environment variable. It may have a local-development default, but no
  password should be required to live in the C# source.
- Use a single `NpgsqlDataSource` for the whole app.
- The app takes a command as its first argument. Implement the first one:
  `version` — print the server version.

**Done when** `dotnet run -- version` prints `PostgreSQL 17.x …`.

**Review:** the project file, `Program.cs`, and the output.

---

### Task 3 — The shop schema

**Goal:** create the two tables the shop needs, from code.

**What to do**

- Keep schema changes as numbered `.sql` files (migrations) in the project,
  for example `Sql/001_schema.sql`.
- Add an app command `migrate` that applies the migration files in order.
  Running it twice must not fail.
- Tables:
  - **customers** — id, email, full name, registration time;
  - **orders** — id, customer (foreign key to customers), status, total
    amount, creation time.
- Pick sensible types (money is not `float`; time has a time zone). Every
  column is required.
- **Only primary keys and the foreign key.** No other indexes or unique
  constraints yet — that's what the lab is about.

**Done when** `dotnet run -- migrate` works twice in a row, and `\d orders` in
psql shows the expected columns and exactly one index (the primary key).

**Review:** the migration file, the `migrate` command, and the `\d orders` /
`\d customers` output.

---

### Task 4 — Realistic data

**Goal:** fill the database with enough data to make the problem real.

**What to do**

- Add an app command `seed` that loads:
  - **200 000 customers** with unique, realistic emails
    (e.g. `anna.silva.12345@yahoo.com`);
  - **2 000 000 orders** spread over the last two years, assigned to random
    customers. Newer orders have bigger ids, like in a real shop.
  - status distribution: delivered 80 %, cancelled 8 %, shipped 5 %,
    paid 4 %, pending 3 %;
  - totals between 5 and 505.
- Requirements:
  - running `seed` again replaces the data instead of adding to it;
  - the data is **deterministic** (same rows every run) so your measurements
    are comparable;
  - the whole seed finishes in **under a minute** — single-row `INSERT`s will
    not get there, find the PostgreSQL bulk-loading mechanism;
  - after loading, the planner's statistics are up to date;
  - print how long the seed took.

**Done when** `dotnet run -- seed` finishes in < 60 s, and in psql you can
show the row counts and the status distribution in percent.

**Review:** the seed code, the seed time, and the counts/distribution query
with its output. **Write down the seed time** — you'll need it in Task 9.

---

### Task 5 — The slow queries and a benchmark

**Goal:** reproduce both complaints and measure them reliably from C#.

**What to do**

- Add an `OrderRepository` class with two methods:
  - **orders by email** — the 50 most recent orders of the customer with the
    given email (support's search);
  - **count by status and period** — how many orders with a given status
    were created in `[from, to)` (finance's widget).
- Both queries must be parameterized. No string concatenation of values.
- Add an app command `bench` that measures both methods:
  - several warm-up runs that are not measured;
  - 100 measured runs each, using **different** realistic inputs per run
    (real emails from the database; 90-day windows for status `cancelled`);
  - print the **median** and **p95** latency in milliseconds and the average
    number of rows/the count returned.

**Done when** `dotnet run -- bench` prints median and p95 for both queries,
and the numbers are stable (±20 %) across three runs.

**Review:** the repository, the benchmark code, and the output of three runs.
**Fill in the "Baseline" column of the results table.**

---

### Task 6 — Ask the database why

**Goal:** find out *what PostgreSQL actually does* for each query.

**What to do**

- Run both queries in psql with `EXPLAIN (ANALYZE, BUFFERS)`, using real
  values (a real email, a 90-day window).
- For each plan, answer in your own words:
  - Which node reads the `orders` table, and how?
  - How many rows did it *read* vs. how many it *kept*?
  - How many 8 kB pages (buffers) did the whole query touch? Compare that to
    the size of the `orders` table in pages.
  - Which part of the plan takes most of the time?

**Done when** you can explain each plan line by line to the mentor.

**Review:** both plans and your answers. Record the **buffers** in the results
table.

---

### Task 7 — Fix the support search

**Goal:** get "orders by email" under **2 ms median**.

**What to do**

- *Before* changing anything, write down which index(es) you think are
  needed and why.
- Add indexes through a new migration file (`002_…sql`), not by hand.
- Add **one index at a time**. After each: run `bench`, look at the new plan,
  and write down what changed. Keep going until you hit the goal.
- Your final schema should also stop two customers from registering with the
  same email.

**Done when** the median for "orders by email" is < 2 ms and the plan has no
sequential scan of `orders` or `customers`.

**Review:** the migration, the bench output after *each* index, and the final
plan. Was your prediction right?

---

### Task 8 — Fix the finance dashboard

**Goal:** get "cancelled orders per quarter" under **5 ms median** and touch
as few pages as possible.

**What to do**

- Try **at least three** index designs for this query, one at a time, and
  drop the losers. At least two of them must be composite indexes containing
  the same columns in a **different order**.
- For each design record: index size, `bench` median, and buffers from
  `EXPLAIN (ANALYZE, BUFFERS)`.
- Keep the winner in a migration.

**Done when** the median is < 5 ms, you have a comparison table of your
designs, and you can explain why the winner wins.

**Review:** your comparison table and the final plan.

---

### Task 9 — What do indexes cost?

**Goal:** find the price you paid for the speed.

**What to do**

- Measure the size of the `orders` table and of every index on it. Compare
  the total index size with the table size.
- Run `seed` again **with your indexes in place** and compare the seed time
  to the one from Task 4.
- Measure how long it takes to create your indexes on an already loaded
  table (drop them, time `CREATE INDEX`, put them back).
- From these numbers, propose the fastest way to bulk-load a large table that
  needs indexes.

**Done when** the cost rows of the results table are filled in and you have a
one-paragraph recommendation.

**Review:** the numbers and the recommendation.

---

## Results table

Copy this table into `workspace/RESULTS.md` and fill it in as you go.

| Measurement                                  | Baseline | After Task 7 | After Task 8 |
|----------------------------------------------|----------|--------------|--------------|
| Orders by email — median / p95 (ms)          |          |              |              |
| Orders by email — buffers                    |          |              |              |
| Cancelled per quarter — median / p95 (ms)    |          |              |              |
| Cancelled per quarter — buffers              |          |              |              |
| Total size of indexes on `orders` (MB)       |          |              |              |

| Cost                                         | Value |
|----------------------------------------------|-------|
| `orders` table size (MB)                     |       |
| Seed time — primary keys only (s)            |       |
| Seed time — with your indexes (s)            |       |
| Time to create your indexes after load (s)   |       |

Typical values on a laptop: baseline 40–400 ms for both queries; after the
lab well under 2 ms and 5 ms. Your hardware will differ — the *ratios* matter.

## Reflection questions

1. An index on `customers.email` alone barely helped "orders by email". Why?
   What does that tell you about foreign keys in PostgreSQL?
2. You have an index on `(status, created_at)`. Which of these queries can
   use it efficiently, and why?
   `WHERE status = 'paid'` · `WHERE created_at > now() - interval '1 day'` ·
   `WHERE status = 'paid' ORDER BY created_at DESC LIMIT 10`
3. In Task 8, an index on `status` alone did almost nothing even though only
   8 % of orders are cancelled. Why didn't it help more? When would an index
   on `status` be worth having?
4. A teammate proposes: "Let's just index every column, just in case." Answer
   them using your own numbers.
5. Why does the benchmark warm up and report median and p95 instead of
   running each query once?

## Going further

Optional, if you finish early:

- Support types emails in mixed case. Make `lower(email) = lower(@email)`
  fast and keep the uniqueness guarantee case-insensitive.
- Only `pending` orders are ever polled by the warehouse. Look up **partial
  indexes** and compare the size with a full index.
- Make the finance query also return `sum(total)` and look at
  `Heap Fetches`. Then read about `INCLUDE` columns.
- Adding an index locks writes on a busy table. Find out what
  `CREATE INDEX CONCURRENTLY` does and what it costs.

Further reading:

- PostgreSQL docs — [Indexes](https://www.postgresql.org/docs/17/indexes.html)
  and [Using EXPLAIN](https://www.postgresql.org/docs/17/using-explain.html)
- [Use The Index, Luke](https://use-the-index-luke.com/) by Markus Winand
- Npgsql docs — [COPY](https://www.npgsql.org/doc/copy.html)
