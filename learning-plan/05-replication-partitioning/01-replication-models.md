# Example 01 — Replication models: leader-follower, multi-leader, leaderless

Three fundamentally different architectures for keeping multiple copies of data in sync.

## 1. Leader-follower (a.k.a. primary-replica, master-slave)

One node is the **leader**. All writes go through it. The leader streams a change log to **followers**, which apply the same writes in order.

```
              ┌─► follower A (read)
[client] ─► [leader] (writes + reads)
              └─► follower B (read)
              └─► follower C (read)
```

- **Reads** can hit leader or followers (followers may be stale).
- **Writes** only go to leader.
- **Failover**: a follower is promoted to leader if the current leader dies.

### Synchronous vs asynchronous

- **Synchronous**: leader waits for at least one (or all) follower(s) to ack before responding to the client. **No data loss on leader crash** (the data exists on a follower) but writes block if a follower is slow.
- **Asynchronous**: leader replies immediately; followers catch up later. **Fast writes**, but up to N seconds of writes can be lost on leader crash.
- **Semi-sync** (common in practice): wait for at least one follower's ack. Balances both.

### Used by

- Postgres (streaming replication)
- MySQL (binlog replication)
- MongoDB (replica sets)
- Redis (default replication)
- AWS RDS, Cloud SQL (with managed failover)

### When this wins

- You have a clear "primary" pattern of writes.
- Read scaling > write scaling is what you need.
- HA via failover is acceptable (seconds of downtime during promotion).

### Limitations

- **One writer** = single point of write bottleneck.
- **Replication lag** = stale reads from followers.
- **Failover is risky** (split brain, lost writes).

## 2. Multi-leader (a.k.a. multi-master, active-active)

**Two or more leaders** accept writes. Each leader replicates to the others.

```
[client US] ─► [leader US] ⇄ [leader EU] ◄─ [client EU]
                    ↓             ↓
              [follower]   [follower]
```

- Used across **datacenters / regions**, where you don't want every write to cross the ocean.
- Or for **offline-first apps** where each device has its own local leader.

### The killer problem: conflicts

If two leaders update the same key at roughly the same time, you have a conflict.

```
t=0: leader US gets WRITE(user, name="Alice")
t=0: leader EU gets WRITE(user, name="A-lice")
t=0+50ms: each replicates to the other.
At t=100ms, both leaders see two writes. Which is the "correct" name?
```

Resolution strategies:
- **Last-write-wins (LWW)** — use timestamps; pick the larger. **Loses data**. Clocks can drift.
- **Application-level resolution** — give the app both versions and let it decide.
- **CRDTs** (Conflict-free Replicated Data Types) — special data structures that mathematically converge regardless of order (e.g., counter, set, last-writer-wins register). Used in distributed databases like Riak, CockroachDB internals, Yjs/Automerge for collaborative editing.

### Used by

- Some Postgres extensions (BDR, pglogical).
- CouchDB.
- Many "edge databases" (Cloudflare D1 R/O replicas + writes routed to leader; Turso/libSQL has multi-primary plans).
- Database-internal: Cassandra is effectively multi-leader at the partition level.

### When this wins

- **Multi-region writes** with locality (writers in each region don't pay cross-ocean latency).
- **Eventual consistency** is acceptable on conflicting writes.
- You have a strategy to handle conflicts.

## 3. Leaderless

**No leader exists.** Any node can accept any read or write. Coordination happens by **quorum**.

```
                ┌─► node 1
[client] ──────►├─► node 2   (each write goes to multiple nodes)
                └─► node 3
```

### How quorum works

You pick:
- **N** = total replicas for the key.
- **W** = how many nodes must ack a write.
- **R** = how many nodes must respond for a read.

The rule: **W + R > N** guarantees you'll read the latest write (the read set overlaps with the write set by at least one node).

Common tunings:
- `N=3, W=2, R=2` — survives 1 node down on each operation.
- `N=3, W=3, R=1` — fast reads, slow writes.
- `N=3, W=1, R=1` — fast everything, weakest consistency.

### How conflicts are resolved

Same as multi-leader, mostly LWW or app-level resolution. Vector clocks (Dynamo paper) detect causally-concurrent writes.

### Used by

- Cassandra (Dynamo-inspired, leaderless).
- Amazon DynamoDB (effectively leaderless internally).
- Riak (the original Dynamo open source).
- ScyllaDB.

### When this wins

- **High availability** even under network partitions.
- **Tunable consistency** per-operation (some reads can be cheap-and-stale; others quorum'd-and-fresh).
- **No failover** to think about — there's no leader to fail.

## Comparison summary

| Property                 | Leader-Follower             | Multi-Leader                  | Leaderless                  |
|--------------------------|------------------------------|--------------------------------|------------------------------|
| Write scaling            | One leader (bottleneck)      | N leaders                      | N nodes                      |
| Read scaling             | Many followers               | Many leaders + followers       | Many nodes                   |
| Consistency by default   | Strong (on leader)           | Eventual (with conflict res.)  | Tunable (quorum)             |
| Conflicts                | None (one writer)            | Frequent (cross-leader)        | Possible (concurrent writes) |
| Operational simplicity   | Highest                      | Lowest                         | Medium                       |
| Failover                 | Required                     | None per leader                | None                         |

## Architect's takeaway

- **Default to leader-follower** unless you have a clear reason for the others.
- **Multi-leader** is right when geographic write latency hurts and you can handle conflicts.
- **Leaderless (Dynamo-style)** is right when you need always-on availability and can tune consistency per call.
- **Sync vs async replication is a durability lever.** Cross-DC sync = expensive but no data loss; async = fast but accept some loss.
- **Most real systems** use leader-follower per shard, and partition data across many shards (next example).
