# Step 05 — Replication & Partitioning

To scale data beyond one machine you have **two** independent dimensions:

- **Replication** — same data on multiple machines (for read scale, HA, geographic locality).
- **Partitioning / sharding** — different data on different machines (for write scale, storage scale).

Most production data systems combine both. This step is also where CAP theorem stops being a slogan and becomes practical.

## Goals

- Explain the three replication models (leader-follower, multi-leader, leaderless) and pick correctly.
- Diagnose replication lag and its impact ("read-your-own-writes" problem).
- Pick a partitioning strategy (range, hash, directory) given a use case.
- Explain consistent hashing and why naive `% N` breaks on resize.
- State CAP and PACELC, give a concrete example for each region of the diagram.
- Distinguish strong / eventual / causal / read-your-writes consistency.

## Key concepts

1. **Replication** — copying data to multiple nodes.
2. **Synchronous vs asynchronous replication** — durability vs latency trade.
3. **Leader-follower** (master-slave) — Postgres, MySQL, MongoDB, Redis default.
4. **Multi-leader** — both nodes accept writes, conflicts must be resolved. Used in multi-DC setups.
5. **Leaderless** — quorum reads/writes, anyone can write. Cassandra, DynamoDB.
6. **Replication lag** and the read-your-own-writes problem.
7. **Partitioning strategies** — range, hash, hash with consistent-hash ring.
8. **Rebalancing** — adding/removing nodes without taking the cluster down.
9. **CAP theorem** — pick 2 of {Consistency, Availability, Partition tolerance}.
10. **PACELC** — even without a Partition, you choose between Latency and Consistency.
11. **Consistency models** — strong, sequential, causal, eventual, read-your-writes, monotonic reads.

## Reading

- **DDIA**: Chapter 5 (Replication), Chapter 6 (Partitioning), Chapter 9 (Consistency and Consensus). This is the heart of the book.
- **Primer**: replication, sharding, CAP theorem.
- **Xu V1**: Chapter 5 (Consistent Hashing), Chapter 6 (Key-Value Store) — practical applications.

## Examples in this folder

- `01-replication-models.md` — leader-follower, multi-leader, leaderless.
- `02-replication-lag.md` — the bug that breaks "I just saved it".
- `03-partitioning-strategies.md` — range vs hash vs directory.
- `04-consistent-hashing.md` — the ring, virtual nodes, why naive hash mod fails.
- `05-cap-and-pacelc.md` — the trade-off with real-world examples.
- `06-consistency-models.md` — strong → eventual, with one example each.

## Self-check

1. You add a follower replica to scale reads. A user saves a profile change, then refreshes and the old name is back. What happened and what are the three ways to fix it?
2. Range-partitioning by user signup date — when does it bite you?
3. What does consistent hashing buy you compared to `hash(key) % N`?
4. Cassandra is "AP". Postgres is "CP". What does each statement actually mean in a network-partition scenario?
5. Name a scenario where "eventual consistency" is unacceptable, and one where it's better than strong.
