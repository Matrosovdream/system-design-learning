# Example 06 — OLAP vs OLTP: why one database rarely does both well

Two workload families that look superficially similar but stress the database engine in opposite ways.

## OLTP — Online Transaction Processing

The day-to-day app database. The thing your web server talks to.

**Characteristics**
- **Many concurrent users** (thousands to millions).
- **Tiny transactions** (single row inserts, by-id lookups, a few-row updates).
- **Latency-sensitive** — user is waiting (under 100 ms typically).
- **Hot working set** fits in RAM (mostly).
- **Mostly reads** but writes must be durable and fast.
- **Indexed point lookups** and small range scans dominate.

**Examples**: Postgres, MySQL, SQL Server, Oracle, MongoDB. The classic "the database".

## OLAP — Online Analytical Processing

The data warehouse. The thing your BI tools, dashboards, and ML pipelines talk to.

**Characteristics**
- **Few concurrent users** (handful of analysts, batch jobs).
- **Huge queries** (scan billions of rows, group by, aggregate).
- **Latency-tolerant** — query takes seconds to minutes is fine.
- **Working set is the whole table** or large slices.
- **Mostly reads**; writes are bulk loads (`COPY`, `INSERT INTO ... SELECT FROM ...`).
- **Aggregations** dominate: `SUM`, `AVG`, `GROUP BY`, joins on huge dimensions.

**Examples**: Snowflake, BigQuery, Redshift, ClickHouse, DuckDB, Apache Druid.

## Why they need different storage engines

### Row store (OLTP)

Stores all columns of a row together on disk.

```
[id=1, name=Alice, country=US, age=30, signup=2022-01-01]
[id=2, name=Bob,   country=UK, age=25, signup=2022-02-15]
...
```

Good for: `SELECT * FROM users WHERE id = 1` — one disk read returns the entire row.

Bad for: `SELECT AVG(age) FROM users` — reads every column of every row but only needs one column. Massive I/O waste.

### Column store (OLAP)

Stores each column separately on disk.

```
ids:       [1, 2, 3, ..., 100M]
names:     [Alice, Bob, ..., Zelda]
countries: [US, UK, ..., AU]
ages:      [30, 25, ..., 42]
signups:   [...]
```

Good for: `SELECT AVG(age) FROM users` — read only the `ages` column. 100× less I/O.

Plus columnar data **compresses brilliantly** — a column of `country` values has very low entropy and shrinks 5-50× on disk. This means a column-store can hold orders of magnitude more data per dollar.

Bad for: `SELECT * FROM users WHERE id = 1` — must fetch one value from each of N columns. Slower than row-store for point lookups.

## The performance gap

| Operation                         | Row store (Postgres) | Column store (ClickHouse) |
|-----------------------------------|----------------------|----------------------------|
| `SELECT * WHERE id = 1`           | ~1 ms                | ~10-50 ms                  |
| `SELECT AVG(amount) FROM 1B rows` | minutes              | seconds                    |
| Single-row INSERT                 | ~1 ms                | not designed for it (batch loads only) |
| Bulk load 1B rows                 | hours                | minutes                    |

## Why you don't run them on the same engine

- **Different on-disk format.** Can't be both row and column at once efficiently (some engines try — hybrid stores).
- **Different concurrency model.** OLTP wants thousands of short concurrent txns. OLAP wants one query to use all the cores.
- **Different optimizer.** OLTP optimizer picks indexes; OLAP optimizer picks scan plans + parallel execution.
- **Different durability/HA.** OLTP needs synchronous WAL fsync; OLAP can tolerate a more relaxed write path because writes are batch.

Running OLAP queries on your OLTP database **kills production**. Big aggregation queries hold long locks, blow out the buffer pool, cause replication lag.

## The standard architecture: ELT pipeline

```
[OLTP DB: Postgres]
     │
     │  CDC (Debezium, Fivetran, etc.)
     ↓
[data lake / warehouse: Snowflake / BigQuery / ClickHouse]
     ↓
[dashboards, BI, ML, reports]
```

1. Application writes to Postgres (OLTP).
2. Change data capture (CDC) streams changes to the warehouse.
3. Warehouse is the source of truth for analytics.

The OLTP DB is **never** queried directly by analysts. This pattern keeps prod fast and analytics flexible.

## HTAP — having your cake and eating it

Hybrid Transactional/Analytical Processing tries to do both in one engine: TiDB+TiFlash, SingleStore, Snowflake Unistore.

Mostly: a row store for the OLTP path + a column store for the OLAP path, with internal replication between them. Works for many workloads, but the operational story is more complex than separate systems.

## A quick example with real numbers

E-commerce. 50M orders. Two queries:

```sql
-- OLTP (the app)
SELECT * FROM orders WHERE id = 12345;

-- OLAP (the analyst)
SELECT country, AVG(total), COUNT(*)
FROM orders
WHERE created_at BETWEEN '2025-01-01' AND '2025-12-31'
GROUP BY country;
```

In Postgres: query 1 is 1 ms; query 2 is ~30 seconds and saturates a core.
In ClickHouse: query 1 is ~20 ms; query 2 is ~1 second on a multi-core box.

If you run query 2 against Postgres prod every 5 minutes for a dashboard, you'll degrade query 1 latency for everyone.

## Architect's takeaway

- **OLTP and OLAP are different jobs.** Don't make one engine do both.
- **Row store = transactions; column store = analytics.** This is the cleanest mental model.
- **CDC + warehouse** is the standard separation pattern. Set it up early — it's cheap when small.
- Avoid letting analysts query prod. Give them a read-replica at minimum, a warehouse for real.
- Column stores like ClickHouse + DuckDB are stunningly cheap and fast — analytics is no longer the "expensive specialist tool" it once was.
