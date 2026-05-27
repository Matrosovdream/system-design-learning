# Case Study 06 — Design a distributed key-value store (Dynamo-style)

This is the question that tests your distributed-systems knowledge most directly. Touches consistent hashing, replication, quorum reads/writes, leaderless coordination, and conflict resolution.

## Problem statement

Build a distributed KV store with:
- Get / Put / Delete by key.
- Horizontal scalability — add nodes, more capacity.
- High availability — survive node failures.
- Tunable consistency.

This is the design of Amazon Dynamo, Cassandra, ScyllaDB, Riak.

## Clarifying questions

1. **Scale**: keys, ops/sec, value size.
2. **Consistency requirements**: strong, eventual, tunable?
3. **Availability target**: 99.9%? 99.99%?
4. **Geography**: single DC, multi-DC, multi-region?
5. **Value types**: opaque blobs, structured (collections, counters)?
6. **Range scans**: by key range, or only point lookups?
7. **TTLs**: do keys expire?

**Assumed answers:**

- 100B keys, 1M ops/sec peak, value size 1KB average.
- Tunable consistency: app can choose strong or eventual per call.
- High availability: 99.99%.
- Multi-DC, single region. Multi-region future work.
- Opaque values.
- Point lookups only (no range scans for now).
- Optional TTLs.

## Functional requirements

- `get(key)` → value (or null).
- `put(key, value)` → ok.
- `delete(key)` → ok.
- Configurable replication factor.
- Configurable consistency level per operation.

## Non-functional requirements

- p99 latency < 10 ms for both reads and writes.
- 99.99% availability.
- Linear scaling: add a node → proportionally more capacity.
- No data loss on single-node failure.

## Capacity estimation

```
100B keys × 1 KB = 100 TB raw data.
With replication factor 3: 300 TB total disk.

Per node: 4 TB usable SSD → need ~80 storage nodes.
With redundancy and headroom: ~100 nodes in the cluster.

Ops/sec:
  1M ops/sec / 100 nodes = 10k ops/sec per node. Easy for modern hardware.
```

## High-level architecture

```
[client]
   │
   │ chooses a coordinator node (any of the cluster nodes)
   ▼
[coordinator node]
   │
   │ on PUT(key, value):
   │   1. Hash the key to find ring position.
   │   2. Find N replicas responsible (consistent hashing).
   │   3. Send write to all N; wait for W acks. (W ≤ N)
   │
   │ on GET(key):
   │   1. Find N replicas.
   │   2. Send read to N; wait for R responses.
   │   3. If conflicting versions: reconcile.
   │
   ▼
[N replica nodes] ←─ gossip protocol for cluster membership/health

Each node:
   - storage engine (LSM tree)
   - read repair logic
   - anti-entropy (Merkle trees) for background sync
```

## Deep dive: consistent hashing

Cluster nodes positioned on a 0..2^64 hash ring. Each gets many **virtual nodes** (~256 each for even distribution).

Key K:
```
hash_value = murmur3(K)
walk clockwise on the ring → first virtual node owns this key
the next N-1 distinct physical nodes are replicas
```

On node addition/removal: only the keys in the affected arc move. ~1/N of total keys.

Why this matters: rebalancing on cluster topology change is small and online (no downtime). Detailed in step 05 example 04.

## Deep dive: replication and quorum

Replication factor `N`. Common: N=3.

Per operation, two values are chosen:
- `W` = number of nodes that must ack the write.
- `R` = number of nodes that must respond to the read.

Guarantees:
- `W + R > N` → consistent reads (read at least one of the most-recent writers).
- `W + R ≤ N` → may read stale.
- `W = N, R = 1` → fast reads, slow writes.
- `W = 1, R = N` → fast writes, slow reads.
- `W = R = (N+1)/2 = 2 for N=3` → balanced quorum.

Each client request specifies its desired (W, R). The coordinator counts acks and decides.

## Deep dive: write path

```
PUT(key, value):

1. Client picks any node (load balancer or random). Call it the "coordinator".
2. Coordinator hashes the key, finds the N replicas.
3. Coordinator sends the write to all N concurrently.
4. Each replica writes locally (LSM tree).
5. Coordinator waits for W acks.
6. After W acks → success to client.
7. Slow replicas continue async. Eventually all N have the write.

For failure (slow / down replica):
- If can't reach W out of N → write fails (unavailable).
- Tunable: relaxed durability with hinted handoff (next).
```

## Deep dive: hinted handoff

What if a replica is down? Don't lose the write.

The coordinator (or a peer) **holds a "hint"** — a deferred write — for the down replica. When the down replica comes back, the hint is delivered.

```
PUT to replicas {A, B, C}. C is down.
Coordinator stores a hint: "for C, when alive, apply write X".
C comes back → coordinator (or any peer) sends the hint.
C applies the write; consistency restored.
```

This way, transient failures don't reduce write durability.

## Deep dive: read path and reconciliation

```
GET(key):

1. Client picks coordinator.
2. Coordinator queries the N replicas.
3. Each replica returns its value + version.
4. Coordinator waits for R responses.
5. If all R agree → return the value.
6. If they disagree → reconcile:
     a. If versions show clear order (one happens-before all others) → return the latest.
     b. If concurrent (vector clock conflict) → return all (or apply LWW).
7. Coordinator may send a read-repair: write the merged result to any stale replica.
```

This catches replicas that fell behind and "self-heals" on read.

## Deep dive: anti-entropy (background sync)

Read-repair only fixes keys that are actively read. Cold keys can stay inconsistent forever.

Anti-entropy: nodes periodically compare what they hold and reconcile.

### Merkle trees

Each node builds a Merkle tree of the keys it owns. When two nodes compare:
- Walk the trees from the top.
- If a subtree hash matches, that whole range is in sync; stop.
- If it differs, recurse.
- At the leaves: send/receive the differing keys.

Efficient: only mismatched ranges are exchanged.

Cassandra, Riak, Dynamo all use Merkle-tree anti-entropy.

## Deep dive: cluster membership (gossip)

In a 100-node cluster, every node needs to know:
- Which nodes are alive.
- Their ring positions.
- Their load.

Solution: **gossip protocol** (Dynamo-style).

```
Every second, each node picks a random peer and exchanges:
- Membership list (with versions).
- Health status.
- Recent change events.

Within O(log N) rounds, the whole cluster converges on a consistent view.
```

No central coordinator. No SPOF. Scales to thousands of nodes.

## Deep dive: storage engine

LSM tree (per step 03 example 02). Writes go to:
1. Write-ahead log (WAL) for durability.
2. In-memory memtable.
3. Flushed to SSTable on disk when memtable fills.
4. Compaction merges SSTables.

Reads:
1. Check memtable.
2. Check SSTables (use Bloom filters to skip irrelevant ones).
3. Return newest version.

This is what Cassandra, RocksDB, LevelDB do.

## Deep dive: conflict resolution

Concurrent writes from different clients can land on different replicas. Vector clocks detect (step 07 example 05).

Strategies:
- **Last-write-wins (LWW)** by wall-clock timestamp. Simple, lossy.
- **Application-resolved**: store both versions; return on read; let app merge.
- **CRDTs** for specific data types (counters, sets).

Dynamo originally used app-resolved; Cassandra uses LWW; Riak supports both.

## Failure modes

### Single node failure

- Gossip detects within seconds.
- The N-1 other replicas still serve reads.
- Hinted handoffs accumulate for the down node.
- On recovery, hints are delivered; anti-entropy catches up.

### Network partition

- Sides of the partition can still serve operations they have quorum for.
- Some keys may be unavailable to one side.
- After healing: anti-entropy resolves conflicts.

This is **AP** behavior (CAP). Reads/writes remain available; consistency is repaired async.

### Cluster topology change (add/remove nodes)

- New node joins ring at chosen positions.
- Some virtual nodes' data is streamed to the new node.
- Old owners stop accepting writes for those ranges once the transfer is acknowledged.
- ~1/N of keys move; rest stay put.

Cassandra has this online via the `nodetool` tool.

## Trade-offs discussion

### Leaderless vs leader-based

This design is **leaderless** (Dynamo-style). Every node can serve any operation. Pros: no failover. Cons: every concurrent write needs reconciliation.

A leader-based alternative (per partition): one leader handles writes, replicates to followers. Simpler consistency, but failover required.

CockroachDB, TiDB are leader-based-per-partition (Raft per range). Both designs work.

### Tunable consistency

Letting apps choose `(W, R)` per call is powerful — they pick the right trade for each query. But it shifts responsibility to the app; misuse is possible.

### Range scans

This design supports only point lookups efficiently. Range scans (`SELECT * WHERE key BETWEEN X AND Y`) require scanning all nodes if keys are hash-distributed. Use **range partitioning** instead (HBase, Bigtable, CockroachDB) if range scans are critical.

## Common follow-up questions

1. **"How do you handle a hot key?"**
   That key lives on one set of N replicas. They get all the traffic; other nodes idle.
   - Replicate hot keys to more nodes (over the standard N).
   - Use client-side caching (L1).
   - Application-level: split the key (counter shards).

2. **"What if a write succeeds on 2 of 3 replicas, then those 2 die before replicating to the 3rd?"**
   With W=2, the write was acknowledged. After two losses, only the unwritten replica remains; the data is gone. This is why W=2 with N=3 has a small failure window — buy N=5 if you can't tolerate it.

3. **"How does this differ from Cassandra?"**
   Conceptually nearly identical. Cassandra adds:
   - CQL (SQL-like query language).
   - Secondary indexes (limited).
   - Lightweight transactions (Paxos for `INSERT ... IF NOT EXISTS`).

4. **"Multi-region?"**
   Each region has its own cluster. Async replication between regions (one-way or bidirectional). Conflict resolution as before. Higher latency for cross-region writes.

5. **"How would you add range scans?"**
   Switch to range partitioning. Or: maintain a separate index (e.g., a B-tree of keys per node). Or use a different store.

6. **"How big can a cluster get?"**
   Cassandra has run at 1000+ nodes. Gossip overhead grows ~O(N²) message volume per cycle but each message is tiny. Practical limit: a few thousand nodes per cluster.

7. **"What about TTLs?"**
   Each value has an optional TTL. Compaction discards expired values. Simple and free.

## Key takeaways

- **Consistent hashing + virtual nodes** spread data evenly and let topology change online.
- **Quorum (W + R > N) gives consistent reads** without a leader.
- **Hinted handoff + read repair + Merkle-tree anti-entropy** keep replicas in sync.
- **Gossip** decentralizes cluster membership.
- **LSM tree storage** absorbs write-heavy loads.
- **Tunable consistency** lets apps pick per-call.
- This design is exactly what Dynamo, Cassandra, Riak, ScyllaDB are. Understanding it = understanding a large family of real-world distributed databases.
