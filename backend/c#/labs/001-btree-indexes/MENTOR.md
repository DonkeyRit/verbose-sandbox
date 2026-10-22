# Mentor guide: Speeding up search with B-tree indexes

## Lab summary

The trainee builds a small shop database (200k customers, 2M orders) in
Docker, talks to it from a .NET console app with Npgsql, and reproduces two
slow queries: *orders by customer email* (a join) and *count of orders by
status in a period* (a KPI). Without indexes both are full table scans.

Key insights the trainee must reach:

1. A join needs an index on **both** sides of the lookup path — and
   PostgreSQL does **not** create an index for a foreign key.
2. In a composite B-tree index, **equality columns go first, range columns
   last**. `(status, created_at)` beats `(created_at, status)` by ~10× in
   pages read for the same data.
3. Indexes are not free: they cost disk (here, almost as much as the table)
   and slow bulk loads 1.5–3×. Loading first and indexing afterwards is much
   faster.

## Key concepts

- **Sequential scan** — reads every page of the table; cost grows with table
  size, not with result size.
- **B-tree index** — a sorted structure of `(key → row location)`; finds
  matching keys in `O(log n)` and then reads only the matching rows.
- **Buffers** — 8 kB pages touched by the query (`shared hit` = from
  PostgreSQL's cache, `read` = from the OS/disk). The best hardware-independent
  measure of work done.
- **Rows Removed by Filter** — rows read and then thrown away; a large number
  means the access path does not match the `WHERE` clause.
- **Selectivity** — the share of rows a condition keeps. Indexes pay off for
  selective conditions; for 8 % of rows scattered across the table, a plain
  index on `status` still touches nearly every page.
- **Composite index column order** — the index is sorted by the first column,
  then by the second within equal first values. Equality on the leading
  column + range on the next = one contiguous slice of the index.
- **Index-only scan** — if every column a query needs is in the index (and
  the visibility map is current), the table is not read at all.
- **Foreign keys are not indexed automatically** in PostgreSQL (unlike MySQL
  InnoDB). Unindexed FKs also make `DELETE` on the parent table slow.
- **Bulk loading** — `COPY` is an order of magnitude faster than row-by-row
  `INSERT`; maintaining indexes during the load is the main slowdown.
- **Planner statistics** — `ANALYZE` lets the planner estimate row counts;
  stale statistics after a bulk load lead to bad plans.

## Task guide

### Task 1 — PostgreSQL in Docker

**Expected outcome:** `workspace/compose.yaml` (or `docker-compose.yml`) with
a `postgres:17` service, named volume, health check, non-default host port.

**Review checklist**
- Image pinned to at least the major version (`postgres:17`, `postgres:17.x`).
- `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` set; not `postgres`.
- Named volume mounted at `/var/lib/postgresql/data`.
- Health check using `pg_isready` with the right user/db.
- Host port mapped (e.g. `55432:5432`) — avoids clashing with a local install.
- Trainee shows `docker compose up -d --wait` → *Healthy* and
  `docker compose exec db psql -U shop -d shop -c 'select version()'`.
- `workspace/` is on the `trainee/<name>` branch.

**Common mistakes**
- `image: postgres` / `postgres:latest` — non-reproducible.
- No volume → data lost on `down`; or a bind mount into the repo that ends up
  committed. Ask them to check `git status`.
- Running `psql` on the host (often not installed) instead of in the container.
- Port `5432:5432` clashing with a local PostgreSQL → "password authentication
  failed" against the *wrong* server. Very confusing for juniors.

**Guiding questions**
- "If I run `docker compose down` and `up` again, is the data still there?
  How do you wipe it on purpose?" (`down -v`)
- "How does `--wait` know the database is ready and not just started?"

**Hint ladder**
1. "Look at the *Environment Variables* and *healthcheck* sections of the
   official `postgres` image page on Docker Hub."
2. "You need four things in the service: `image`, `environment`, `ports`,
   `volumes` — plus a `healthcheck` using `pg_isready`."
3. Show the structure with blanks (keys only, no values).

**Reference**

```yaml
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: shop
      POSTGRES_PASSWORD: shop
      POSTGRES_DB: shop
    ports:
      - "55432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U shop -d shop"]
      interval: 2s
      timeout: 3s
      retries: 20
volumes:
  pgdata:
```

After approval, create `workspace/PROGRESS.md` (lab, task, date, open items).

---

### Task 2 — A console app that talks to the database

**Expected outcome:** console project with Npgsql, a single
`NpgsqlDataSource`, connection string from an env var (e.g. `SHOP_DB`) with a
local default, a `version` command.

**Review checklist**
- `NpgsqlDataSource` created once and disposed (`await using`), not a new
  `NpgsqlConnection` with a string in every method.
- Commands, readers, connections in `await using`.
- Connection string not built by concatenating user input.
- Async APIs used (`ExecuteScalarAsync`), not `.Result`.
- Unknown command → prints usage, doesn't crash.
- `bin/` and `obj/` not committed (add a `.gitignore` in the workspace, e.g.
  `dotnet new gitignore`).

**Common mistakes**
- Password hard-coded in a C# constant with no override.
- `using` missing → leaked connections (won't show yet, but call it out —
  it's a *must fix*).
- Creating an `NpgsqlDataSource` per query.

**Guiding questions**
- "Where would the password come from in production?"
- "What happens to the connection if an exception is thrown before `Close()`?"

**Hint ladder**
1. "Look at the Npgsql *Getting started* page — the first example uses a data
   source."
2. "`NpgsqlDataSource.Create(connectionString)` then `dataSource.CreateCommand(sql)`."
3. Show the reference `version` case only.

**Reference**

```csharp
using Npgsql;

var connectionString = Environment.GetEnvironmentVariable("SHOP_DB")
    ?? "Host=localhost;Port=55432;Username=shop;Password=shop;Database=shop";
await using var db = NpgsqlDataSource.Create(connectionString);

switch (args.FirstOrDefault())
{
    case "version":
    {
        await using var cmd = db.CreateCommand("SELECT version()");
        Console.WriteLine(await cmd.ExecuteScalarAsync());
        break;
    }
    default:
        Console.WriteLine("usage: migrate | seed | bench | version");
        break;
}
```

A local-dev default with a throwaway password is acceptable here; the env var
override is the point.

---

### Task 3 — The shop schema

**Expected outcome:** `Sql/001_schema.sql` copied to output, `migrate`
applies files in name order, idempotent.

**Review checklist**
- `bigint` ids (identity preferred over `serial`), `numeric(10,2)` for money,
  `timestamptz` for times, `text` for email/status/name, all `NOT NULL`.
- `REFERENCES customers (id)` on `orders.customer_id`.
- **No** index or `UNIQUE` on `email`, **no** index on `customer_id`. If the
  trainee added them, ask them to remove them and explain the lab will
  come back to it — otherwise there is no problem to observe.
- Idempotency: `CREATE TABLE IF NOT EXISTS` (or a migrations-history table —
  praise it, but don't require it).
- Files ordered explicitly (`.Order()`), not relying on file system order.
- `.sql` files copied to output (`CopyToOutputDirectory`) or embedded.

**Common mistakes**
- `float`/`double precision`/`money` for totals.
- `timestamp` without time zone.
- `varchar(50)` for emails — ask "what happens with a 51-character email?"
- Adding `UNIQUE (email)` "because it's obviously right". It *is* right — it
  just belongs to Task 7.
- Migration file not found at runtime (path relative to the working dir).

**Guiding questions**
- "What's `0.1 + 0.2` in a `double`? Do you want that in an invoice?"
- "If a customer in Tokyo places an order, what time is stored?"
- "Which indexes does `\d orders` show? Did you create any of them?" (the PK one)

**Hint ladder**
1. "Check the *Numeric types* and *Date/Time types* pages of the PostgreSQL docs."
2. "The app needs the `.sql` files next to the binary — look at
   `CopyToOutputDirectory` in the `.csproj`."
3. Show the reference schema.

**Reference**

```sql
CREATE TABLE IF NOT EXISTS customers (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email       text        NOT NULL,
    full_name   text        NOT NULL,
    created_at  timestamptz NOT NULL
);

CREATE TABLE IF NOT EXISTS orders (
    id           bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id  bigint        NOT NULL REFERENCES customers (id),
    status       text          NOT NULL,
    total        numeric(10,2) NOT NULL,
    created_at   timestamptz   NOT NULL
);
```

```xml
<ItemGroup>
  <None Include="Sql/*.sql" CopyToOutputDirectory="PreserveNewest" />
</ItemGroup>
```

```csharp
case "migrate":
    foreach (var file in Directory.GetFiles(Path.Combine(AppContext.BaseDirectory, "Sql"), "*.sql").Order())
    {
        await using var cmd = db.CreateCommand(await File.ReadAllTextAsync(file));
        await cmd.ExecuteNonQueryAsync();
        Console.WriteLine($"applied {Path.GetFileName(file)}");
    }
    break;
```

---

### Task 4 — Realistic data

**Expected outcome:** `seed` truncates and loads 200k customers + 2M orders
with binary `COPY`, fixed random seed, `VACUUM ANALYZE` at the end. Takes
~10–40 s.

**Review checklist**
- Uses `BeginBinaryImportAsync` / `COPY … FROM STDIN (FORMAT BINARY)`.
- `TRUNCATE orders, customers RESTART IDENTITY` (or equivalent) before
  loading — so customer ids are 1..200 000 and orders can reference them.
- `new Random(42)` (any fixed seed).
- Orders generated in time order (ids grow with `created_at`).
- Status distribution matches (ask for the `GROUP BY status` query with
  percentages — expect ±0.1 %).
- `ANALYZE` (ideally `VACUUM ANALYZE`) after the load. `VACUUM` also sets the
  visibility map, which enables index-only scans in Task 8.
- Emails unique (contain the customer number).
- `CompleteAsync()` called — otherwise the COPY is rolled back silently.

**Common mistakes**
- Row-by-row `INSERT` in a loop → many minutes. Let them start it, look at
  the projected time, then ask how to do it faster.
- Multi-row `INSERT` with string concatenation → SQL injection risk *and* slow.
- Forgetting `CompleteAsync()` → table empty, no error.
- `"VACUUM ANALYZE customers; VACUUM ANALYZE orders;"` in **one** command →
  `25001: VACUUM cannot be executed within a pipeline`. Npgsql sends
  multi-statement commands as a batch; run one statement per command.
- `Random` without seed → numbers drift between runs.
- Emails generated randomly without the customer number → duplicates.

**Guiding questions**
- "At your current speed, how long will 2 million rows take?"
- "Does PostgreSQL have a dedicated command for loading lots of data?"
- "How does the planner know how many rows a table has right after you
  loaded it?"

**Hint ladder**
1. "Look at PostgreSQL's `COPY` command, then at what Npgsql offers for it."
2. "Npgsql docs → *COPY* → *Binary COPY*: `BeginBinaryImportAsync`,
   `StartRowAsync`, `WriteAsync`, `CompleteAsync`."
3. Show the reference customer loop only; let them write the orders loop.

**Reference**

```csharp
using Npgsql;
using NpgsqlTypes;

public static class Seeder
{
    static readonly string[] FirstNames = ["anna", "ivan", "maria", "john", "olga", "peter", "elena", "david", "sofia", "alex"];
    static readonly string[] LastNames = ["smith", "ivanov", "garcia", "mueller", "kowalski", "rossi", "novak", "silva", "kim", "brown"];
    static readonly string[] Domains = ["gmail.com", "outlook.com", "yahoo.com", "mail.ru", "proton.me"];

    public static async Task SeedAsync(NpgsqlDataSource db, int customers, int orders)
    {
        await using (var cmd = db.CreateCommand("TRUNCATE orders, customers RESTART IDENTITY"))
            await cmd.ExecuteNonQueryAsync();

        var rnd = new Random(42);
        var now = DateTime.UtcNow;

        await using var conn = await db.OpenConnectionAsync();
        await using (var w = await conn.BeginBinaryImportAsync(
            "COPY customers (email, full_name, created_at) FROM STDIN (FORMAT BINARY)"))
        {
            for (var i = 1; i <= customers; i++)
            {
                var first = FirstNames[rnd.Next(FirstNames.Length)];
                var last = LastNames[rnd.Next(LastNames.Length)];
                await w.StartRowAsync();
                await w.WriteAsync($"{first}.{last}.{i}@{Domains[rnd.Next(Domains.Length)]}", NpgsqlDbType.Text);
                await w.WriteAsync($"{char.ToUpper(first[0])}{first[1..]} {char.ToUpper(last[0])}{last[1..]}", NpgsqlDbType.Text);
                await w.WriteAsync(now.AddDays(-rnd.Next(730, 1500)), NpgsqlDbType.TimestampTz);
            }
            await w.CompleteAsync();
        }

        // Orders are written in time order, like a real shop: newer orders get bigger ids.
        var start = now.AddDays(-730);
        var step = TimeSpan.FromDays(730).Ticks / orders;
        await using (var w = await conn.BeginBinaryImportAsync(
            "COPY orders (customer_id, status, total, created_at) FROM STDIN (FORMAT BINARY)"))
        {
            for (var i = 0; i < orders; i++)
            {
                await w.StartRowAsync();
                await w.WriteAsync((long)rnd.Next(1, customers + 1), NpgsqlDbType.Bigint);
                await w.WriteAsync(PickStatus(rnd), NpgsqlDbType.Text);
                await w.WriteAsync(Math.Round((decimal)(rnd.NextDouble() * 500 + 5), 2), NpgsqlDbType.Numeric);
                await w.WriteAsync(start.AddTicks(step * i), NpgsqlDbType.TimestampTz);
            }
            await w.CompleteAsync();
        }

        // VACUUM refuses to run inside a multi-statement batch, so one command per table.
        foreach (var table in new[] { "customers", "orders" })
        {
            await using var cmd = new NpgsqlCommand($"VACUUM ANALYZE {table}", conn);
            await cmd.ExecuteNonQueryAsync();
        }
    }

    static string PickStatus(Random rnd) => rnd.Next(100) switch
    {
        < 80 => "delivered",
        < 88 => "cancelled",
        < 93 => "shipped",
        < 97 => "paid",
        _ => "pending",
    };
}
```

```sql
SELECT status, count(*), round(100.0 * count(*) / sum(count(*)) OVER (), 1) AS pct
FROM orders GROUP BY status ORDER BY count(*) DESC;
```

Note: `created_at` values depend on "now", so timestamps shift between runs on
different days; the *distribution* stays the same, which is what matters.

---

### Task 5 — The slow queries and a benchmark

**Expected outcome:** `OrderRepository` with `FindByEmailAsync` and
`CountByStatusAsync`, a `Benchmark` helper with warm-up, median, p95.

**Review checklist**
- Parameters (`@email`, `@status`, `@from`, `@to`), never interpolated values.
  Interpolating values is a **must fix** (SQL injection).
- Orders-by-email query: `JOIN` customers, `WHERE c.email = @email`,
  `ORDER BY o.created_at DESC LIMIT 50`.
- Count query: half-open range `created_at >= @from AND created_at < @to`.
- Benchmark: warm-up runs excluded; `Stopwatch` around *only* the call;
  timings sorted; median = middle, p95 = 95th percentile.
- Inputs vary per run (different emails, sliding 90-day windows) — otherwise
  they measure a cache, not a search.
- Emails sampled from the DB *outside* the timed section.
- Results consumed (count rows), so the work isn't skipped.

**Common mistakes**
- Measuring once / averaging — one GC pause or cold cache dominates.
- Timing includes picking the random email (`ORDER BY random()` is itself a
  full scan).
- `BETWEEN` for the range — double-counts the boundary.
- Same email every iteration.
- Running the benchmark in `Debug`. Not dramatic for I/O-bound code, but
  suggest `-c Release` for habits.

**Guiding questions**
- "If one of your 100 runs hits a GC pause, how does it change the average?
  The median?"
- "Why does p95 matter to the support agent more than the average?"

**Hint ladder**
1. "Collect all timings into an array first, then compute statistics."
2. "Sort the array: the median is the middle element, p95 is at index
   `0.95 * n`."
3. Show the reference `Benchmark.RunAsync`.

**Reference**

```csharp
using Npgsql;

public record OrderRow(long Id, long CustomerId, string Status, decimal Total, DateTime CreatedAt);

public class OrderRepository(NpgsqlDataSource db)
{
    public const string ByEmailSql = """
        SELECT o.id, o.customer_id, o.status, o.total, o.created_at
        FROM orders o
        JOIN customers c ON c.id = o.customer_id
        WHERE c.email = @email
        ORDER BY o.created_at DESC
        LIMIT 50
        """;

    public const string CountByStatusSql = """
        SELECT count(*)
        FROM orders
        WHERE status = @status AND created_at >= @from AND created_at < @to
        """;

    public async Task<List<OrderRow>> FindByEmailAsync(string email)
    {
        await using var cmd = db.CreateCommand(ByEmailSql);
        cmd.Parameters.AddWithValue("email", email);

        var result = new List<OrderRow>();
        await using var reader = await cmd.ExecuteReaderAsync();
        while (await reader.ReadAsync())
            result.Add(new OrderRow(reader.GetInt64(0), reader.GetInt64(1), reader.GetString(2),
                reader.GetDecimal(3), reader.GetDateTime(4)));
        return result;
    }

    public async Task<long> CountByStatusAsync(string status, DateTime from, DateTime to)
    {
        await using var cmd = db.CreateCommand(CountByStatusSql);
        cmd.Parameters.AddWithValue("status", status);
        cmd.Parameters.AddWithValue("from", from);
        cmd.Parameters.AddWithValue("to", to);
        return (long)(await cmd.ExecuteScalarAsync())!;
    }
}
```

```csharp
using System.Diagnostics;

public static class Benchmark
{
    public static async Task RunAsync(string name, int iterations, Func<int, Task<int>> action)
    {
        for (var i = 0; i < 5; i++) await action(i); // warm-up: connection pool, plan cache, buffer cache

        var timings = new double[iterations];
        var rows = 0;
        for (var i = 0; i < iterations; i++)
        {
            var sw = Stopwatch.StartNew();
            rows += await action(i);
            timings[i] = sw.Elapsed.TotalMilliseconds;
        }
        Array.Sort(timings);
        Console.WriteLine($"{name,-22} n={iterations,4}  median={timings[iterations / 2],9:F2} ms  " +
                          $"p95={timings[(int)(iterations * 0.95)],9:F2} ms  avg rows={rows / (double)iterations:F1}");
    }
}
```

```csharp
case "bench":
{
    // Pick real emails so most lookups hit, like a support agent would.
    var emails = new List<string>();
    await using (var cmd = db.CreateCommand("SELECT email FROM customers ORDER BY random() LIMIT 100"))
    await using (var r = await cmd.ExecuteReaderAsync())
        while (await r.ReadAsync()) emails.Add(r.GetString(0));

    await Benchmark.RunAsync("orders by email", 100,
        async i => (await repo.FindByEmailAsync(emails[i % emails.Count])).Count);

    var quarterStart = DateTime.UtcNow.Date.AddDays(-400);
    await Benchmark.RunAsync("cancelled per quarter", 100,
        async i => (int)await repo.CountByStatusAsync("cancelled", quarterStart.AddDays(i), quarterStart.AddDays(i + 90)));
    break;
}
```

---

### Task 6 — Ask the database why

**Expected outcome:** two plans and correct explanations.

**Review checklist** — the trainee should be able to say:
- Orders by email: `Parallel Seq Scan on customers` with `Rows Removed by
  Filter` ≈ 66 666 per worker (× 3 loops ≈ 200k), and `Parallel Seq Scan on
  orders` reading all ~2M rows into a `Hash Join`. Time is dominated by the
  scan of `orders`.
- Count: `Parallel Seq Scan on orders`, ~20k rows kept, ~2M removed.
- Buffers ≈ 18 500 for the count query ≈ the whole `orders` table
  (`SELECT pg_relation_size('orders') / 8192` → ~18 450). For the join,
  ~20 800 = orders + customers.
- The work is proportional to the **table**, not to the result (7–10 rows /
  one number).

**Common mistakes**
- Reading only `Execution Time` and ignoring the plan.
- Missing that numbers in parallel nodes are **per worker** — multiply by
  `loops`.
- Confusing `cost=` (planner estimate, arbitrary units) with `actual time=`.
- Running `EXPLAIN` without `ANALYZE` → only estimates, no actual rows.
- Running `EXPLAIN ANALYZE` on a parameter placeholder — they need a real
  literal value in psql (or `\set`/`\gset`).

**Guiding questions**
- "How many rows did the scan on `orders` read to give you 7 results?"
- "If the shop grows to 20M orders, what happens to this query? To the
  result?"
- "What does `loops=3` mean for the numbers on that line?"

**Hint ladder**
1. "Read the plan bottom-up and inside-out: the most indented nodes run first."
2. "Find the nodes whose name contains `Scan` — those are the ones reading
   tables."
3. Walk through one plan together, line by line, and let them do the second.

**Reference**

```sql
SELECT email FROM customers WHERE id = 12345 \gset
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, o.customer_id, o.status, o.total, o.created_at
FROM orders o JOIN customers c ON c.id = o.customer_id
WHERE c.email = :'email'
ORDER BY o.created_at DESC LIMIT 50;

EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM orders
WHERE status = 'cancelled'
  AND created_at >= now() - interval '400 days'
  AND created_at <  now() - interval '310 days';
```

---

### Task 7 — Fix the support search

**Expected outcome:** `Sql/002_indexes.sql` with a unique index on
`customers(email)` and an index on `orders(customer_id)`. Median ~0.5 ms.

**The intended discovery:** most trainees index `email` first. Let them.
The bench barely moves (~60 → ~55 ms) and the plan shows `Index Scan using
customers_email_key` feeding a hash join that still does `Parallel Seq Scan on
orders`. Then they find the second index and get a ~100× speed-up. **Do not
warn them up front** — the surprise is the lesson.

**Review checklist**
- Unique index (or `UNIQUE` constraint) on `customers.email` — satisfies the
  "no duplicate emails" requirement.
- Index on `orders.customer_id`.
- `IF NOT EXISTS`, so `migrate` stays re-runnable.
- Bench output after each index (two snapshots at least).
- Final plan: `Nested Loop` → `Index Scan` on customers →
  `Bitmap Heap Scan`/`Index Scan` on orders. A small `Sort` on top is fine
  (~10 rows).
- Their prediction written down before the change, and an honest comparison.

**Common mistakes**
- Indexes created by hand in psql and not in a migration → lost on the next
  fresh environment. **Must fix.**
- Composite `(customer_id, created_at)` — not wrong (it removes the sort),
  praise it, but ask whether it's worth it with ~10 orders per customer.
- Index on `orders.id` / `customers.id` — already exist (PK). Ask them to
  check `\d`.
- Expecting `REFERENCES` to have created an index.

**Guiding questions**
- (after the email index) "The customer is found instantly now. What does the
  database do *next* to find their orders? Look at the plan."
- "Is there anything in the plan that is still proportional to the table
  size?"
- "Did `REFERENCES customers (id)` create an index? How can you check?"

**Hint ladder**
1. "The query walks two tables. Look at how each one is accessed now."
2. "Finding the orders of customer 12345 — which column is that lookup on?"
3. "PostgreSQL indexes the referenced side of a foreign key (it's a PK), but
   not the referencing column."

**Reference**

```sql
CREATE UNIQUE INDEX IF NOT EXISTS customers_email_key ON customers (email);
CREATE INDEX IF NOT EXISTS orders_customer_id_idx ON orders (customer_id);
```

---

### Task 8 — Fix the finance dashboard

**Expected outcome:** a comparison of ≥ 3 designs; winner
`(status, created_at)` in a migration. Median ~1.5–2 ms, ~100 buffers.

**Review checklist**
- Comparison table has size, median, and buffers per design.
- Both column orders of the composite index tried.
- Losing indexes dropped (check `\d orders`).
- Winner added to a migration (`003_…` or appended to `002` with
  `IF NOT EXISTS`).
- Explanation mentions: equality on `status` narrows the index to one
  contiguous slice ordered by `created_at`; the range is then a single
  sub-slice. With `(created_at, status)`, the range comes first and the index
  must walk every order in the quarter (~250k entries) checking `status`.
- Bonus: notices `Index Only Scan` + `Heap Fetches: 0`.

**Common mistakes**
- Leaving all experimental indexes in place — they'd skew Task 9 and the
  planner picks whichever it likes. Ask them to check `\d orders`.
- Comparing only latency and not buffers. With everything cached the
  latency gap looks smaller than the work gap.
- Running `EXPLAIN` right after `CREATE INDEX` and getting
  `Heap Fetches > 0` because the visibility map is stale — fine after
  `VACUUM`.
- Concluding "`status` index is useless, indexes on low-cardinality columns
  are always useless" — steer towards selectivity and towards it being useful
  *as the leading column of a composite index*.

**Guiding questions**
- "Imagine the index as a phone book sorted by (status, date). Where are all
  cancelled orders from Q2? Now sort it by (date, status) instead."
- "The `status` index found 160k cancelled orders. How many pages of the
  table did it then have to read? Why so many?"
- "Which plan read the fewest buffers? Is that the same one with the lowest
  latency?"

**Hint ladder**
1. "Try an index on each column alone first, and look at buffers."
2. "One index can cover both conditions. Which column should be first?"
3. "Equality columns first, range columns last."

**Reference**

```sql
CREATE INDEX IF NOT EXISTS orders_status_created_at_idx ON orders (status, created_at);
```

Measured on the reference run (M-series laptop, 90-day window, ~19 650
matches):

| Index                     | Size  | psql time | Buffers | Plan                              |
|---------------------------|-------|-----------|---------|-----------------------------------|
| none                      | —     | ~58 ms    | 18 450  | Parallel Seq Scan                 |
| `(status)`                | 13 MB | ~53 ms    | 18 580  | Bitmap Heap Scan, 47k removed/worker |
| `(created_at)`            | 43 MB | ~14 ms    | 3 580   | Index Scan, 75k removed/worker    |
| `(created_at, status)`    | 75 MB | ~10 ms    | 1 190   | Index Only Scan                   |
| `(status, created_at)`    | 75 MB | ~1.4 ms   | 100     | Index Only Scan                   |

The `(created_at)` index is decent here only because rows are physically
stored in time order (correlation ≈ 1). Good discussion point.

---

### Task 9 — What do indexes cost?

**Expected outcome:** sizes, two seed times, index build time, and a
recommendation: *load first, then create indexes* (or drop → load → create).

**Review checklist**
- Sizes via `pg_relation_size` / `pg_indexes_size` / `\di+`.
- Total index size on `orders` compared to table size (roughly equal!).
- Seed time with indexes noticeably higher (~1.5–2×).
- Index build after load timed with `\timing` — a few seconds total.
- Recommendation is backed by their numbers, and mentions that for a live
  table you can't just drop indexes (link to *Going further*:
  `CONCURRENTLY`).

**Common mistakes**
- Measuring single-row `INSERT` latency and concluding "indexes cost
  nothing". Per-row latency is dominated by the network round trip and
  commit (~0.4 ms either way in the reference run). The cost shows up in
  **throughput** and bulk loads. If they did this, it's a great discussion,
  not a failure.
- Using `pg_total_relation_size('orders')` as "table size" — it already
  includes indexes and TOAST.
- Forgetting that the PK is an index too (43 MB here).

**Guiding questions**
- "Each `COPY`'d row now has to be inserted into how many B-trees?"
- "Is it cheaper to insert 2M keys into a B-tree one by one, or to sort 2M
  keys once and build the tree?"

**Hint ladder**
1. "`\di+` in psql shows index sizes."
2. "`TRUNCATE` keeps the indexes — just run `seed` again."
3. "Drop the indexes, `\timing on`, create them again."

**Reference**

```sql
SELECT indexrelid::regclass AS index, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_index WHERE indrelid = 'orders'::regclass;

SELECT pg_size_pretty(pg_table_size('orders'))   AS table,
       pg_size_pretty(pg_indexes_size('orders')) AS indexes;
```

## Expected results

Reference run: PostgreSQL 17.11 in Docker on Apple Silicon (10 cores, 24 GB),
.NET 9, Npgsql 9.0.3, Release build. Expect 1–5× slower on older laptops or
Docker on Windows; the **ratios** should hold.

| Measurement                              | Baseline        | After Task 7   | After Task 8    |
|------------------------------------------|-----------------|----------------|-----------------|
| Orders by email — median                 | ~60 ms (40–400) | ~0.5 ms (< 2)  | ~0.5 ms         |
| Orders by email — buffers                | ~20 800         | ~15–20         | ~15–20          |
| Cancelled per quarter — median           | ~50 ms (40–400) | ~50 ms         | ~1.6 ms (< 5)   |
| Cancelled per quarter — buffers          | ~18 500         | ~18 500        | ~100            |
| Total size of indexes on `orders`        | 43 MB (PK)      | 61 MB          | 137 MB          |

| Cost                                     | Reference value |
|------------------------------------------|-----------------|
| `orders` table size                      | ~145 MB         |
| Seed time — PK only                      | ~13 s (10–40)   |
| Seed time — with 3 lab indexes           | ~21 s           |
| Seed time — with 3 more "just in case" indexes | ~35 s     |
| Create the 3 indexes after load          | ~2.4 s total    |
| Single-row `INSERT` latency, PK only vs. 6 indexes | ~0.43 vs ~0.45 ms |

Red flags:
- Baseline under 5 ms → data not loaded, or indexes created too early.
- After Task 7 still > 5 ms → `orders.customer_id` index missing, or
  stats not refreshed after load.
- Huge p95 vs median → laptop on battery saver, other heavy processes, or
  Debug + debugger attached.

## Understanding check

1. **Email index alone.** Finding the customer becomes instant, but the
   database still must find that customer's orders, and with no index on
   `orders.customer_id` that means scanning all 2M orders. PostgreSQL only
   requires an index on the *referenced* columns (a PK/unique); the
   referencing FK column is never indexed automatically — you add it yourself
   (also needed for fast `DELETE`/`UPDATE` of customers).
2. **`(status, created_at)` usage.**
   - `WHERE status = 'paid'` — yes, leading column.
   - `WHERE created_at > …` alone — not efficiently; `created_at` is sorted
     only *within* each status. (PG 18 skip scan can help with few distinct
     statuses; on 17, expect a seq scan or full index scan.)
   - `WHERE status = 'paid' ORDER BY created_at DESC LIMIT 10` — yes, best
     case: jump to `paid`, read the last 10 entries backwards, no sort.
3. **`status` alone.** 8 % of 2M = 160k rows, scattered over the whole table
   — almost every 8 kB page holds a cancelled order, so the bitmap heap scan
   still reads ~every page, plus the index. Indexes pay off when the condition
   is *selective* (few rows / few pages). A `status` index would be useful for
   a rare value (e.g. if `pending` were 0.1 %) — even better as a partial
   index — or as the leading column of a composite index, as in Task 8.
4. **Index everything.** Using their numbers: indexes already take about as
   much space as the table; each extra index slowed bulk loading
   (13 → 21 → 35 s); unused indexes still cost on every write, `VACUUM`, and
   in memory (cache). Index the queries you actually run, verified by
   `EXPLAIN`, and drop indexes that `pg_stat_user_indexes.idx_scan` shows are
   unused.
5. **Benchmarking.** The first runs pay one-off costs (opening connections,
   JIT, cold caches). Single runs are noisy; the median is robust to
   outliers; p95 shows what the slowest users experience. Varying inputs
   prevents measuring a single cached page.

## Pitfalls & edge cases

- **Port clash on 5432** with a local PostgreSQL → authentication errors
  against the wrong server. Use another host port.
- **Apple Silicon / ARM:** the official `postgres` image is multi-arch; no
  action needed.
- **`VACUUM cannot be executed within a pipeline` (25001):** multiple
  statements in one `NpgsqlCommand` are sent as a batch; `VACUUM` can't run
  in it. One statement per command.
- **`COPY` writes nothing:** `CompleteAsync()` not called.
- **Timestamps:** Npgsql 6+ maps `DateTime` with `Kind=Utc` to
  `timestamptz`. `DateTime.Now` (Local kind) throws when written to a
  `timestamptz` column — use `DateTime.UtcNow`.
- **Plans differ from the reference** (e.g. `Index Scan` vs `Bitmap Heap
  Scan`, or no parallel workers) — fine; buffers and row counts are what to
  compare.
- **Flaky timings:** close heavy apps, plug in the laptop, run bench 3×,
  compare medians. Docker Desktop on Windows/macOS adds overhead on the
  first runs.
- **`Heap Fetches` > 0 in Task 8:** visibility map not up to date — run
  `VACUUM orders`.
- **Disk:** the lab needs ~500 MB in the Docker volume. Clean up with
  `docker compose down -v` at the end if needed.
- **Today-dependent data:** `created_at` is relative to "now"; re-seeding on
  another day shifts dates but not distributions.

## Out of scope

Answer briefly and point to future labs (not yet written):

- `LIKE '%foo%'` / full-text search → GIN, `pg_trgm`.
- Hash, GiST, BRIN indexes.
- Partitioning large tables.
- EF Core and `HasIndex` — this track uses plain Npgsql until a later lab.
- Connection pool tuning, prepared statements, query plan caching.
- `CREATE INDEX CONCURRENTLY` and zero-downtime migrations — mentioned in
  *Going further*, covered in depth later.
- Transactions and isolation levels.
