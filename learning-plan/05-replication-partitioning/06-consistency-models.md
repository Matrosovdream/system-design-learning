# Example 06 — Consistency models: from strong to eventual

"Consistency" is not one thing. There's a spectrum of guarantees from "exactly like a single machine" down to "anything goes". Knowing the levels lets you pick the right one and not over-pay.

## The spectrum (strongest → weakest)

```
Linearizable  →  Sequential  →  Causal  →  Read-your-writes  →  Monotonic reads  →  Eventual
```

## 1. Linearizable (a.k.a. strong / atomic / strict)

There is **one consistent global order** of all operations, and that order matches **real time**. If you finish writing X at 12:00:00.500, every subsequent reader anywhere sees X.

The illusion of a single machine. The strongest, most expensive guarantee.

### Cost

- Every write requires a global consensus round-trip.
- Across regions, bound by light-speed RTT (~100-200ms inter-continent).
- Available only when a majority of nodes can communicate.

### Examples

- Spanner, FoundationDB, etcd, ZooKeeper, single-node databases.
- Single-shard transactions in CockroachDB, TiDB.

### Use when

- Distributed locking (only one node can be elected leader).
- Money transfers across accounts.
- Inventory decrements (avoid overselling).

## 2. Sequential consistency

All operations appear in **some** consistent global order, but that order **need not match real time**.

In practice: your write at 12:00 might appear after someone else's write at 12:01 in the global order, as long as both observers agree on *some* order.

Rarely the explicit choice; you usually pick linearizable or weaker.

## 3. Causal consistency

If operation B causally **happened after** A (i.e., B saw A's effect), then **everyone sees A before B**. But unrelated operations can be seen in any order.

### A concrete picture

```
Alice posts message M1.
Bob reads M1 and replies M2.
Carol writes M3 (unrelated).

Anyone watching:
- must see M1 before M2 (causal)
- may see M3 anywhere
```

### Cost

- Track causal links (vector clocks, hybrid logical clocks).
- Lighter than linearizable; available without global consensus.

### Examples

- COPS, Eiger (research systems).
- MongoDB causal consistency (since 3.6).
- Some collaborative editing systems.

### Use when

- Threaded conversations (replies must appear after the original).
- Comment threads.
- Multi-step workflows where order matters within one user's actions.

## 4. Read-your-writes (a.k.a. read-after-write)

After you write X, *you* will read X. **Other users might still see the old value**.

### Cost

Cheap: pin the user's reads to the leader briefly, or carry a version cursor.

### Examples

- Most modern apps implement this even on eventually-consistent backends.
- DynamoDB session consistency mode.

### Use when

This is the **minimum bar** for most user-facing apps. Without it, "I just saved that — where did it go?" bug reports flood in.

## 5. Monotonic reads

Once you've seen a value V at time T, all your subsequent reads see **V or newer**. You never see time go backward.

### Cost

Pin reads to one node, or sticky-session the user.

### Use when

User browsing data — likes count, follower count, etc. Going from 100 → 95 → 102 looks broken.

## 6. Eventual consistency

If writes stop, all replicas **eventually** converge. No bounds on how long "eventually" takes.

### Cost

Almost none. Cheapest, most available.

### Examples

- DNS.
- DynamoDB default reads.
- Async replicas of Postgres for non-critical reads.

### Use when

- Aggregate counters that don't need to be exact.
- Analytics, dashboards, view counts.
- Anywhere staleness of seconds is fine.

## A practical menu by feature

| Feature                            | Minimum consistency needed             |
|------------------------------------|------------------------------------------|
| Choose a leader / acquire a lock   | Linearizable                             |
| Bank transfer                      | Linearizable                             |
| Inventory decrement                | Linearizable (or app-level lock + retry) |
| Threaded comment reply             | Causal                                   |
| User's own profile after edit      | Read-your-writes                         |
| Like counts                        | Monotonic reads (don't go backward)      |
| Trending hashtags                  | Eventual                                 |
| Social feed                        | Eventual + RYW for own posts             |
| Analytics dashboard                | Eventual                                 |
| Real-time multiplayer state        | Causal + low latency (often game-specific) |

## How DBs let you choose

Per-operation knobs are common:

- **DynamoDB**: `ConsistentRead: true/false`. Strong (linearizable) vs eventual.
- **Cassandra**: tunable `CONSISTENCY` (`ONE`, `QUORUM`, `ALL`, etc.) per query.
- **MongoDB**: `readConcern` and `writeConcern`, plus causal consistency sessions.
- **CockroachDB**: serializable always; you don't usually relax.

You can **mix per query**. Strong-read the user's own profile, eventual-read the trending feed.

## A multi-region example

Global product, US-east primary DB, EU and APAC replicas.

| Feature              | Strategy                                                  |
|----------------------|-----------------------------------------------------------|
| Login (auth)         | Strong read from regional cache + occasional linearizable check |
| Order placement      | Linearizable to US-east primary                           |
| Order status         | Read-your-writes via session cookie carrying LSN          |
| Catalog browse       | Eventual read from regional replica                       |
| User feed            | Eventual                                                  |
| Notifications        | Causal (preserve order)                                   |

You pay the expensive consistency (linearizable + cross-region) **only** where the business requires it. Everything else uses the cheapest consistency that works.

## Architect's takeaway

- **There's no "default consistency level".** Pick per data type.
- **Read-your-writes is the minimum bar** for user-facing apps. Free or near-free to implement.
- **Linearizable is expensive** (latency, availability). Reserve it for true invariants: money, leadership, uniqueness.
- **Eventual consistency is a feature** for high-availability, low-latency systems.
- **Modern DBs let you tune per-operation.** Use that knob — don't pay for strong consistency on read paths that don't need it.
