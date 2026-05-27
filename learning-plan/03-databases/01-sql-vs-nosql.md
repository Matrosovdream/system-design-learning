# Example 01 — SQL vs NoSQL: a decision framework, not a religion

"NoSQL" is not a thing. It's a label for several wildly different storage models (key-value, document, wide-column, graph). The choice is per-workload, not per-team-fashion.

## What SQL gives you (and what it makes hard)

### SQL (Postgres, MySQL, SQL Server, Oracle, SQLite)

**Strengths**
- **Strong schema** — fields are typed, constraints enforced (`NOT NULL`, `CHECK`, `FOREIGN KEY`).
- **Joins** — combine data from many tables in one query.
- **ACID transactions** by default, across multiple rows and tables.
- **Mature tooling** — 50 years of optimizers, replication, backups, ORMs.
- **Rich query language** — SQL is incredibly expressive.

**Trade-offs**
- **Vertical-scaling first**, horizontal sharding is bolted on (or hard).
- **Schema migrations** on huge tables are painful (`ALTER TABLE` locks).
- **Joins across shards** don't work cheaply.

**Use when:** transactions, relational data, financial correctness, complex queries, you don't yet know exactly how the data will be queried.

## What "NoSQL" gives you (it depends which one)

### Key-Value (Redis, DynamoDB, Memcached)

`SET key value`, `GET key`. Nothing else.

- **Strengths**: simplest, fastest, easiest to shard (hash the key).
- **Trade-offs**: no querying by value, no joins, no rich types.
- **Use when**: caching, session storage, rate-limit counters, leaderboards (Redis sorted sets).

### Document (MongoDB, DynamoDB, Couchbase, Elasticsearch)

Store JSON-like documents. Each doc can have a different shape.

- **Strengths**: flexible schema, natural fit for object-shaped data, embedded documents reduce joins.
- **Trade-offs**: no enforced schema → data quality drifts; cross-doc transactions weak or absent (improving); querying across many fields requires many indexes.
- **Use when**: content (articles, products), catalogs, user profiles, anywhere the "shape" of an entity is naturally hierarchical.

### Wide-column (Cassandra, ScyllaDB, HBase, BigTable)

Tables with rows, but rows can have thousands of columns and not every row has the same columns. Designed to be sharded across many nodes.

- **Strengths**: massive write throughput, linear horizontal scale, fault-tolerant by design (replication built in).
- **Trade-offs**: you design the schema around queries (denormalized, **read paths drive the model**); no joins; secondary indexes are limited.
- **Use when**: time-series data, IoT telemetry, message history, audit logs, event tracking at huge scale.

### Graph (Neo4j, Amazon Neptune, ArangoDB)

Data as nodes + edges. Queries traverse relationships.

- **Strengths**: queries like "friends-of-friends-of-friends" or "shortest path" are cheap.
- **Trade-offs**: harder to scale horizontally; specialized query language (Cypher, Gremlin).
- **Use when**: social graphs, fraud-ring detection, recommendation traversal, knowledge graphs.

### Time-series (InfluxDB, TimescaleDB, Prometheus, VictoriaMetrics)

Optimized for time-stamped values: append-heavy writes, time-range reads, downsampling.

- **Use when**: metrics, monitoring, IoT, financial ticks.

### Search (Elasticsearch, OpenSearch, MeiliSearch, Typesense)

Full-text search with relevance scoring. Inverted index instead of B-tree.

- **Use when**: free-text search, autocomplete, log search, faceted product search.

## The decision framework

Ask in order:

1. **Do I need ACID transactions across multiple records?**
   - Yes → **SQL** (or NewSQL: Spanner, CockroachDB, Yugabyte).
   - No → continue.

2. **Is the data fundamentally relational (lots of cross-entity joins)?**
   - Yes → **SQL**.
   - No → continue.

3. **What's the dominant access pattern?**
   - Lookup by key → **Key-value**.
   - Full-text search → **Search**.
   - Time-range scan → **Time-series**.
   - Graph traversal → **Graph**.
   - Hierarchical object → **Document**.
   - Massive append + range scan → **Wide-column**.

4. **What's the scale?**
   - < a few TB or < ~10k QPS → SQL is almost always fine, simpler is better.
   - 10s+ TB or 100k+ QPS → consider purpose-built NoSQL.

## Common patterns (the real world uses both)

- **SQL for source-of-truth, NoSQL for derived views**: orders in Postgres, search index in Elasticsearch, sessions in Redis, analytics in ClickHouse.
- **DynamoDB + Postgres**: DynamoDB for high-throughput entity lookups (user profile, cart), Postgres for relational reporting.
- **Postgres alone**: most teams underestimate how far this goes. Postgres at 1 TB and 10k QPS is *easy* on a beefy box.

## Common mistakes

- **"NoSQL because we want scale"** before you have any scale problem. Postgres beats most NoSQL stores up to mid-scale on operational simplicity.
- **"We need MongoDB for the flexibility"** — Postgres has JSONB; you usually want some structure anyway.
- **"Joins are slow"** — joins on indexed columns are fast. Denormalized NoSQL just moves the join cost to write time.
- **"Schema is bad"** — schema is a feature. The data drifts without it.

## Architect's takeaway

- **Start with SQL** unless your dominant access pattern clearly points elsewhere.
- The right answer is often **polyglot persistence**: SQL for the core domain, specialized stores for specialized access patterns.
- **Match the engine to the access pattern**, not the buzzword. A wrong DB choice locks in years of pain.
