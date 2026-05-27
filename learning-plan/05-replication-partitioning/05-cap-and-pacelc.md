# Example 05 — CAP and PACELC: the real trade-off

The most-quoted, least-understood theorem in distributed systems.

## CAP, stated correctly

When a **network partition** occurs (some nodes can't talk to others), a distributed system must choose between:

- **C**onsistency — every read sees the most recent write, or an error.
- **A**vailability — every request gets a non-error response (possibly stale).

You **cannot have both** while a partition is happening. Partition tolerance (**P**) is not optional — partitions happen in real networks.

So the real choice is **CP** or **AP**.

> Common misreading: "CAP says you pick 2 of 3". Wrong. You always have P (network failures are real). You pick C or A *when a partition occurs*. The rest of the time, you can have both.

## Concrete: what CP and AP look like in a partition

### CP — Consistency over Availability

When the network splits, a node that can't confirm consensus with the rest of the cluster **refuses** the request (returns an error or hangs).

The data stays consistent but some clients are denied service until the partition heals.

Examples:
- **Postgres in a leader-follower setup with a lost leader** — followers refuse writes until a new leader is elected. (CP.)
- **HBase, Spanner, etcd, Consul (with strict consistency)** — quorum required for reads/writes.
- **Most ACID RDBMS** when sync replication is enabled across partitioned sides.

### AP — Availability over Consistency

When the network splits, every node continues to accept reads and writes against whatever copy it has. The data may **diverge** between sides of the partition.

When the partition heals, the system **must** reconcile divergent versions (LWW, CRDTs, app-level resolution).

Examples:
- **Cassandra** (tunable, but AP by default).
- **DynamoDB** (in eventual-consistency mode).
- **Riak, CouchDB**.
- **DNS** (a giant AP system — caches everywhere, eventually consistent).

## PACELC: the realer trade-off

CAP only describes partition time, which is rare. The rest of the time, you still trade things.

**PACELC** (Daniel Abadi, 2010):

> **If P**artition, **A**vailability or **C**onsistency. **E**lse, **L**atency or **C**onsistency.

In other words: even when the network is fine, you choose between fast-but-stale and slow-but-strict.

### Why this matters

A system that demands consensus for every write (Spanner across regions) is **slow at all times**, not just during partitions. The cross-region RTT (~150 ms) is in every write path.

A system that doesn't require consensus (Cassandra) is **fast at all times** — but may return stale reads at any time.

### PACELC examples (the textbook table)

| System            | If Partition         | Else                |
|-------------------|----------------------|---------------------|
| Postgres (single) | (n/a)                | (n/a)               |
| Postgres (sync replication, multi-DC) | PC (refuses writes)  | EC (slower for consistency) |
| Spanner           | PC                   | EC (latency for consistency) |
| Cassandra         | PA (keep going)      | EL (low latency, tunable)    |
| DynamoDB (default)| PA                   | EL                  |
| DynamoDB (strong read mode) | PC          | EC                  |
| MongoDB (default replica set) | PC       | EC                  |
| etcd / ZooKeeper  | PC                   | EC                  |
| Riak              | PA                   | EL                  |

## How this plays out in design

### Money / inventory / identity → CP / EC

You'd rather **stop service** than process a wrong payment. Postgres with sync replication, Spanner, CockroachDB. Higher latency, brief unavailability during failover, **but invariants hold**.

### Social feed / browse / view counts → AP / EL

You'd rather show a slightly stale feed than show an error. Cassandra, DynamoDB, eventually-consistent caches. Always-on, sub-10ms latencies, but the like count might be 5 seconds behind reality.

### Hybrid (the common pattern)

Inside one app:
- Orders: Postgres (CP / EC).
- Cart: Redis (AP-ish, with TTL).
- Search index: Elasticsearch (AP / EL).
- Session: Redis (AP).
- Audit log: append-only, eventual ok.

You pick **per data type**, not per company.

## Common misconceptions

### "CAP says NoSQL is better than SQL"

No. CAP describes a trade-off. Both classes of system make a choice. SQL traditionally chose CP; many NoSQL chose AP. Both are valid for their use case.

### "Cassandra is AP and Postgres is CP — that settles it"

Cassandra is **tunable**: `CONSISTENCY ALL` makes it CP-ish (writes fail if any node down). Postgres in async-replica mode is closer to AP (writes succeed even when a follower can't replicate, you just get lag).

### "I'll pick CP everywhere because consistency matters"

You'll regret it the first time a single AZ outage takes your write path down. **AP has real availability value** in user-facing read paths.

## A real story: AWS S3 and DynamoDB

S3 was originally eventually consistent (AP). Years later it became strongly read-after-write consistent (CP-ish), at the cost of some operational complexity, because customer apps kept asking for it.

DynamoDB by default returns "eventually consistent" reads — fast and cheap. You opt in to "strongly consistent" reads (CP-ish behavior) per call, paying 2× the read units and accepting higher latency.

> The same underlying system can lean either way, per-request, if you build it that way.

## Architect's takeaway

- **CAP is about partition behavior.** PACELC adds the always-on latency-vs-consistency trade.
- **Choose per data type**, not per system. A bank has both CP and AP data.
- **"Eventual consistency" is a tool**, not a defect. Used right, it gives you availability that pure CP can't.
- **Avoid pure CP across regions** unless you really need it — the latency cost is high (Spanner is expensive in milliseconds *and* dollars for a reason).
- **Avoid pure AP for money flows** — the eventual reconciliation will fail in ways your customers notice.
