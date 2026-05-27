# Step 03 — Databases

The database is usually the **first thing to break** and the **last thing to fix** in a scaling story. You need to know how each engine stores data, what an index actually costs, how transactions behave under concurrency, and when "SQL or NoSQL?" has a real answer.

## Goals

- Pick SQL vs NoSQL with a real reason, not a fashion-driven one.
- Explain how a B-tree index and an LSM-tree index store data, and when each wins.
- Name the four ANSI isolation levels and a concrete anomaly each prevents.
- Define ACID and BASE; show where each applies.
- Know what an index *costs* (writes, storage) and design indexes by query, not by column.
- Read an `EXPLAIN` plan and spot a sequential scan that should be an index seek.

## Key concepts

1. **Storage engines** — how rows actually live on disk: row-store vs column-store, B-tree vs LSM-tree.
2. **Indexes** — secondary structures for fast lookup. Primary, secondary, composite, covering, partial.
3. **Transactions** — atomicity, isolation, consistency, durability.
4. **Isolation levels** — read uncommitted → read committed → repeatable read → serializable. Each blocks more anomalies, costs more.
5. **Anomalies** — dirty read, non-repeatable read, phantom read, write skew, lost update.
6. **ACID vs BASE** — strong vs eventual. Not "good vs bad" — context-dependent.
7. **OLTP vs OLAP** — many small transactions vs few huge analytical queries. Different engines.
8. **SQL vs NoSQL family tree** — key-value, document, wide-column, graph, time-series, search.

## Reading

- **DDIA**: Chapters 2 (*Data Models and Query Languages*), 3 (*Storage and Retrieval*), 7 (*Transactions*). These three chapters are the most valuable in the book.
- **Primer**: SQL vs NoSQL, ACID transactions, BASE.

## Examples in this folder

- `01-sql-vs-nosql.md` — when each is the right answer.
- `02-b-tree-vs-lsm.md` — the two dominant on-disk index structures.
- `03-indexes-explained.md` — what an index *costs*, composite/covering/partial.
- `04-isolation-levels.md` — the four levels with anomalies you can actually picture.
- `05-acid-vs-base.md` — the trade-off explained without buzzwords.
- `06-olap-vs-oltp.md` — why one DB rarely does both.

## Self-check

1. Your app needs to record user clicks at 50k/sec. SQL or NoSQL? Why?
2. Why does a B-tree handle range scans cheaply but LSM handle writes cheaply?
3. You add a 4-column composite index `(a, b, c, d)`. Which queries does it speed up — `WHERE a=?`, `WHERE b=?`, `WHERE a=? AND c=?`, `WHERE a=? AND b=?`?
4. Postgres default isolation is "Read Committed". Name one anomaly it does *not* prevent.
5. Why don't analytical queries run well on the same DB as the order-processing system?
