# Example 02 — B-tree vs LSM-tree: the two on-disk index structures that matter

Almost every persistent database uses one of these two structures to store and index data. The choice shapes write performance, read performance, space usage, and compaction behavior.

## B-tree (Postgres, MySQL InnoDB, Oracle, SQL Server)

A balanced tree of pages. Each page holds keys + pointers. The leaves hold (key → row) entries.

```
                [50 | 100]
               /    |    \
        [10|30] [60|80] [110|150]
        / | \   / | \   / | \
      ...rows...rows...rows...
```

- **Read**: walk from root to leaf, O(log N) page reads.
- **Write**: find the right leaf, update it in place. If the leaf is full, split it.
- **Pages are typically 4-16 KB**, kept in a buffer pool in RAM.

### B-tree characteristics

| Trait                       | B-tree                                       |
|-----------------------------|----------------------------------------------|
| Read latency                | Excellent (predictable, O(log N))            |
| Range scan                  | Excellent (sequential leaf traversal)        |
| Write amplification         | Low to moderate (in-place updates)           |
| Random write throughput     | Moderate (page splits, fsync per commit)     |
| Sequential write throughput | Moderate                                     |
| Disk fragmentation          | Yes — pages get half-full, periodic vacuum   |
| Space overhead              | Moderate                                     |
| Crash recovery              | WAL + checkpoint (Postgres) or undo logs     |

## LSM-tree (Cassandra, RocksDB, LevelDB, ScyllaDB, HBase)

Log-Structured Merge tree. Writes go to an in-memory **memtable**. When it fills up, it's flushed to disk as an immutable sorted file (**SSTable**). Background compaction merges SSTables.

```
write  →  [memtable in RAM] ──flush──► [SSTable Level 0]
                                        │
                                   compaction
                                        ↓
                                  [SSTable Level 1]
                                        │
                                        ↓
                                  [SSTable Level 2]
                                       ...
```

- **Write**: append to memtable + WAL. No disk seek. Insanely fast.
- **Read**: check memtable → check each SSTable, newest first → return first match. Costs O(levels × log SSTable size). Mitigated by **bloom filters** that say "this SSTable definitely doesn't have your key".
- **Compaction**: merges SSTables to reduce read amplification.

### LSM-tree characteristics

| Trait                       | LSM-tree                                          |
|-----------------------------|---------------------------------------------------|
| Read latency                | Good when caches warm; worse on cold tail        |
| Range scan                  | Good (each SSTable is sorted)                    |
| Write amplification         | High (data rewritten during compactions)         |
| Random write throughput     | Excellent (memory + sequential disk writes)      |
| Sequential write throughput | Excellent                                        |
| Disk fragmentation          | Less (compaction defragments)                    |
| Space overhead              | Lower steady-state; higher during compaction     |
| Crash recovery              | Replay WAL → rebuild memtable                    |

## Head-to-head

| Workload                       | Winner          | Why                                          |
|--------------------------------|-----------------|-----------------------------------------------|
| OLTP transactions              | B-tree (Postgres) | Predictable latency, mature transaction support |
| Time-series ingestion          | LSM             | Append-heavy, sequential                      |
| Cache layer with persistence   | LSM (RocksDB)   | Massive write throughput                      |
| Analytical scan over warm data | B-tree slightly | More uniform layout (but columnar wins both) |
| Mostly reads, occasional writes| B-tree          | Lower read amplification                      |
| Write-heavy logs / events      | LSM             | Writes are essentially free                   |
| Strict latency p99             | B-tree          | LSM compaction can cause latency spikes      |

## A concrete example

You're choosing storage for a "user events" stream: 50k writes/sec, occasional time-range reads.

- **B-tree backed DB (Postgres):** writes hit the WAL fast, but the table's primary B-tree gets a lot of random insert pressure if `event_id` is not roughly time-ordered. Vacuum and bloat become operational concerns. Probably caps at ~5-15k writes/sec on a single box.

- **LSM backed DB (Cassandra/Scylla):** writes go to memtable + commit log, no seeks. 50k writes/sec is comfortable on a small cluster. Compaction runs in the background. Time-range reads are good if the partition key is `(user_id, event_time)`.

LSM is the obvious choice for this workload.

## Hybrid systems and where the line blurs

- **Postgres** is B-tree, but its WAL is essentially log-structured.
- **MySQL InnoDB** is B-tree, with a redo-log change buffer to absorb writes.
- **RocksDB** (LSM) is embedded inside many systems (Kafka Streams, Flink, TiKV, MyRocks for MySQL).
- **TiDB / TiKV** — distributed SQL on top of LSM.

The strict B-tree-vs-LSM dichotomy is becoming hybrid in practice.

## Architect's takeaway

- **B-tree = OLTP default.** Predictable, well-understood, transaction-friendly.
- **LSM = write-heavy default.** Time-series, telemetry, logs, write-amplified workloads.
- **Bloom filters + cache** turn LSM read performance from "scary" to "fine" most of the time.
- **Compaction is the LSM operator's life.** Tune it, monitor it, or it will surprise you with disk usage and latency.
- The choice is in the engine, not visible at the SQL layer — but it determines what kind of load the engine will swallow.
