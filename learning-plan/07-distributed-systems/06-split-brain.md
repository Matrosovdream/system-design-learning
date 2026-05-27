# Example 06 — Split-brain: two leaders, one cluster, broken invariants

Split-brain is the failure mode where a cluster gets partitioned and **both sides** elect themselves leader. Both sides start accepting writes. Now you have two divergent histories that can't be merged. It's one of the worst things that can happen to a distributed database.

## The setup

A 3-node Postgres cluster with automatic failover: A (primary), B (standby), C (standby).

```
[A primary] ──── [B standby]
       \           /
        \         /
         [C standby]
```

The network experiences a partition:

```
[A] | [B + C]
     ^ partition
```

- A still thinks it's primary.
- B and C can't reach A → they think A is dead.
- B and C run a failover; B is promoted to primary.

**Now both A and B think they are primary.** Clients on the A side write to A; clients on the B side write to B. Histories diverge.

When the partition heals: irreconcilable conflicting commits. Manual intervention (data merge, picking a "winning" side and discarding the other) is the only recovery — and that means losing real customer writes.

## Why this happens

Any system with **automatic failover** and **no consensus protocol** can split-brain. Specifically:

- **Simple "ping the primary, promote if down" failover** has no way to distinguish "primary is dead" from "primary is unreachable from us".
- **Both sides may have a quorum-perceiving subset of clients** and serve writes independently.

In a multi-AZ network failure or partial network problem, the cluster can split into two pieces, each containing some nodes and some clients.

## How proper consensus algorithms prevent it

Raft, Paxos, ZAB (ZooKeeper's protocol) all require a **majority quorum** to elect a leader or commit writes:

- 3 nodes: majority = 2.
- 5 nodes: majority = 3.

In a partition:
- One side has majority (e.g., 2 of 3, or 3 of 5).
- The other side has minority.

**Only the majority side can elect a leader.** The minority side recognizes it has no quorum and **stops accepting writes**.

```
3 nodes split as [A] | [B, C]
B + C = majority (2). They elect a leader (say B).
A by itself = minority. A cannot serve writes (no quorum to commit).

When partition heals:
A sees B with a higher term, recognizes B as leader, becomes follower.
No conflicting writes. No split-brain.
```

This is **the** reason to use a consensus protocol for leader-based replication, instead of rolling your own "primary monitoring + failover" logic.

## Practical examples of split-brain in the wild

### Old Postgres with manual or scripted failover

Tools like `repmgr`, `pacemaker`, simple watchdog scripts could promote a standby without checking whether the old primary is truly down. Famous postmortems exist.

**Fix**: use a proper consensus-based proxy / coordinator. Patroni + etcd (or Consul) is the canonical setup. Etcd uses Raft to ensure only one node is recognized as leader, and Patroni enforces this through fencing.

### MongoDB without majority writes

MongoDB defaults to `writeConcern: w: 1` — a write is acked when **one** node has it. In a split-brain scenario, writes can happen on both sides briefly until elections settle. Using `w: majority` ensures every committed write is on a majority and survives any partition.

### Old Elasticsearch (pre 7.x) "minimum_master_nodes"

Misconfigured Elasticsearch clusters famously split-brained when `minimum_master_nodes` wasn't set to majority. Two halves each elected their own master. ES 7.x simplified this with automatic quorum management.

### Redis Sentinel without proper quorum

Sentinel monitors a master. If you have only 2 Sentinels, both can independently decide "master is down" — each can promote a different replica. Always 3+ Sentinels with quorum-based decisions.

### DRBD / RAID-like replication with auto-failover

Storage-replication systems (DRBD, GlusterFS) used to split-brain easily. Modern versions ship with quorum-based fencing.

## Fencing: making split-brain safe

Even with consensus, you sometimes need **fencing** — a mechanism to forcibly evict the old leader from being able to write.

Approaches:

### STONITH ("Shoot The Other Node In The Head")

When the new leader takes over, it explicitly kills/powers off the old one to make sure it's gone. Common in cluster managers (Pacemaker, etc.).

### Fencing tokens

Every leader gets a token (a monotonically increasing number). All downstream services check the token. An old leader's writes carry an old token — rejected.

Used by Google's Chubby, ZooKeeper (zxid), and many internal systems.

### Lease + clock-bounded grace

The old leader has a lease. When the lease expires (clock-bounded), the leader stops accepting writes voluntarily. The new leader can take over after the lease + safety margin.

Used in Spanner, etcd leases, lock services.

## Detecting it after the fact

If you suspect split-brain happened, look for:

- **Two nodes that both ran writes in the same time window** with no replication trail between them.
- **Conflicting primary-key inserts** in your DB logs.
- **WAL / replication log divergence** at a known point.
- **Customer reports** of "I made a change and it vanished" or "I see two of the same record".

Recovery is **case-by-case painful work**: identify divergent commits, decide a winner, manually reconcile. This is why prevention is everything.

## Worst-case story (composite from real postmortems)

A SaaS company's primary DB cluster auto-failed-over during a network blip. Both old and new primaries ran writes for ~3 minutes. ~12,000 customer-facing changes were written to **both** primaries with conflicting state. Recovery: 4 engineers, 14 hours, a lot of manual SQL, and a public postmortem.

The root cause: failover logic was checking "can I reach the primary?" from one observer. The observer was in a different AZ than the new primary candidates. Network partition meant the observer thought the primary was dead, but the primary was still serving most clients.

After: switched to a Raft-based consensus on cluster state. Failover now requires a true quorum decision, with fencing tokens enforced at the application layer.

## Architect's takeaway

- **Auto-failover without consensus is dangerous.** It's how you get split-brain.
- **Use a consensus protocol** (Raft, Paxos, ZAB) for leader election. Don't write your own watchdog.
- **Fencing is essential** to make sure the old leader cannot continue to write.
- **`majority` write concern** is the right default in replicated systems. Never `w: 1` for important data.
- **Odd-sized clusters only.** Even sizes give no extra safety and risk no-quorum partitions.
- **Patroni + etcd, Vitess + Zookeeper, Spanner-class systems** — proven architectures that prevent split-brain. Lean on them; don't reinvent.
