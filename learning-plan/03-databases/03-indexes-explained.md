# Example 03 — Indexes: what they cost, how to choose them

An index is **a copy of some columns of your table, sorted and stored separately**, used to skip past rows the query doesn't need. Indexes make reads faster — and writes slower, and storage larger. Knowing the trade is the whole point.

## What an index buys you

Without an index, a query like:

```sql
SELECT * FROM users WHERE email = 'alice@example.com';
```

forces a **sequential scan** of all rows until a match is found. On 100M rows that's 100M comparisons.

With an index on `email`, the DB walks a B-tree in O(log N) — ~27 comparisons for 100M rows. **~3.7 million times faster.**

## What an index costs

1. **Storage** — every index is more bytes on disk. A composite index on 4 columns can be larger than the table.
2. **Write latency** — every `INSERT`/`UPDATE`/`DELETE` must update all indexes on affected columns. A table with 10 indexes is ~10× slower to write than one with no indexes.
3. **Memory pressure** — the buffer pool now caches index pages too. Hot index pages compete with hot data pages.
4. **Maintenance** — `VACUUM` (Postgres) / `OPTIMIZE TABLE` (MySQL) periodically rebuild indexes.

Rule of thumb: **add an index when a query is slow because of a scan, not because "this column might be queried someday".**

## Types of indexes

### 1. Primary key index

Automatically created on `PRIMARY KEY`. In InnoDB / Postgres, the table is **clustered** by the primary key — rows are physically laid out in PK order. Lookups by PK are extra-fast.

### 2. Secondary index

A separate B-tree on a non-PK column. The leaves point to the PK (InnoDB) or the row's physical location (Postgres heap).

### 3. Composite index

Index on multiple columns in order: `(country, city, name)`.

Critical property: a composite index supports queries on **any prefix**:
- `WHERE country = ?` ✅ (uses index)
- `WHERE country = ? AND city = ?` ✅
- `WHERE country = ? AND city = ? AND name = ?` ✅
- `WHERE city = ?` ❌ (cannot use this index; no prefix)
- `WHERE name = ?` ❌

**Order matters.** Always list columns from most-selective to least-selective, AND according to the queries you actually run.

### 4. Covering index

Includes **all** columns the query needs (in the SELECT and WHERE). The DB never has to fetch the row itself — the index alone answers the query.

```sql
-- Query
SELECT first_name, last_name FROM users WHERE email = ?;

-- Covering index
CREATE INDEX idx_users_email_cover ON users(email) INCLUDE (first_name, last_name);
```

Massive speedup for hot reads, at the cost of more index storage.

### 5. Partial index

Indexed only over rows that match a predicate:

```sql
CREATE INDEX idx_active_orders ON orders(user_id) WHERE status = 'active';
```

If 95% of your `orders` are `completed` and you query for `active` ones, this index is 20× smaller than a full one — and faster to scan.

### 6. Functional / expression index

Index on the result of a function:

```sql
CREATE INDEX idx_users_lower_email ON users(LOWER(email));
SELECT * FROM users WHERE LOWER(email) = LOWER(?);
```

Without this, the `LOWER(email)` in the WHERE forces a sequential scan.

### 7. Hash index

A hash table instead of a B-tree. O(1) lookup, **but cannot do range queries**. Niche; B-tree is the safer default.

### 8. Full-text / search index

Inverted index — `word → list of doc IDs`. Used by Postgres `tsvector`, MySQL `FULLTEXT`, Elasticsearch.

## How to choose indexes — the workflow

1. **Get a list of your top 20 queries by frequency or latency.** (From `pg_stat_statements`, MySQL slow log, APM.)
2. **For each slow query, run `EXPLAIN`.** Look for `Seq Scan` on large tables.
3. **Design a composite index covering the WHERE clause column order.** Put equality predicates first, range predicates last.
4. **Make it a covering index** if the SELECT list is small.
5. **Re-test with `EXPLAIN ANALYZE`.** Cost should drop dramatically.
6. **Check write performance** — did you make it worse than before?
7. **Drop unused indexes.** Use `pg_stat_user_indexes` / MySQL `sys.schema_unused_indexes`.

## Reading EXPLAIN — the cliff notes

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE country = 'US' AND city = 'SF';
```

Look for:
- `Seq Scan` on a big table → bad. Probably needs an index.
- `Index Scan using idx_xxx` → good. The query uses the index.
- `Index Only Scan` → great. Covering index, no table fetch.
- `Bitmap Heap Scan` → intermediate; using index but still fetching many rows.
- `rows=` estimate vs actual — if very different, statistics are stale (`ANALYZE`).

## Common mistakes

- **Index everything**: write throughput dies; storage explodes.
- **Wrong column order in composite index**: index becomes useless for the actual query.
- **Function in WHERE without expression index**: `WHERE LOWER(email) = ?` won't use a plain `email` index.
- **Index on a low-cardinality column** (e.g., boolean): scanning still hits 50% of rows. Useless.
- **Forgetting indexes on foreign keys**: joins become full scans.

## Architect's takeaway

- **An index is a copy.** Treat it as data, not magic.
- **Add indexes from the query side, not the schema side.** Look at what's slow.
- **Composite > many single-column indexes** when queries combine columns.
- **Covering indexes** are the highest-leverage move for read-heavy hot paths.
- **Periodically audit unused indexes** — they cost writes and storage every day for nothing.
