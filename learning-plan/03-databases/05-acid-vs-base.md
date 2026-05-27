# Example 05 — ACID vs BASE: the trade-off in plain language

ACID and BASE are two ends of a spectrum about **how strong your consistency guarantees are**, and what you pay for them.

## ACID (the traditional SQL guarantee)

- **A**tomicity — all-or-nothing. Either the whole transaction commits, or none of it.
- **C**onsistency — the DB never violates declared constraints (foreign keys, checks).
- **I**solation — concurrent transactions appear to run one at a time (per the isolation level).
- **D**urability — once committed, the data survives crashes.

Postgres, MySQL, Oracle, SQL Server — all ACID by default.

## BASE (the NoSQL alternative coined ~2008)

- **B**asically **A**vailable — the system always returns a response, even if it's "stale" or "ok, ack'd, will be applied later".
- **S**oft state — state can change over time without input, as the system propagates updates.
- **E**ventually consistent — given enough time without new writes, all replicas converge to the same value.

Cassandra, DynamoDB (by default), early MongoDB, S3 (originally) — BASE.

## What's actually traded?

The trade is between:
- **strong consistency** (every read sees the latest write, everywhere) and
- **availability + partition tolerance** (the system keeps responding even when replicas can't talk to each other).

This is **CAP theorem** territory (covered in step 05). Briefly: when the network splits, you must pick consistency *or* availability. ACID systems usually pick consistency; BASE systems usually pick availability.

## A concrete example: a shopping cart on Black Friday

### ACID approach

User adds item → Postgres transaction → COMMIT. Every read by the user, on any device, immediately reflects the new cart.

Black Friday: the primary DB hits write capacity. The DB starts rejecting writes / latency explodes. Users see errors. **Consistency preserved, availability lost.**

### BASE approach

User adds item → DynamoDB write to one replica → ack'd immediately. Other replicas catch up async (~100s of ms).

Black Friday: traffic spike absorbed across replicas. User sometimes sees a slightly stale cart for a few seconds on another tab. **Availability preserved, consistency relaxed.**

Which is "right"? Depends on the business. Stripe wants ACID for payments. Amazon (famously) accepted eventual consistency on the shopping cart in the early 2000s because availability was worth more than strict correctness for that page.

## "Eventual consistency" — what it actually means

It's not "consistency, eventually" handwaved. It comes with specific levels:

- **Read-your-writes**: a user sees their own writes immediately (but not necessarily other users').
- **Monotonic reads**: once you've seen a value, you won't see an older one later.
- **Monotonic writes**: your writes from one client are applied in order.
- **Bounded staleness**: replicas catch up within X ms / Y writes.
- **Consistent prefix**: you never see an "impossible" combination (e.g., a child message without its parent).

Different systems offer different combinations. DynamoDB lets you ask for "strongly consistent read" at higher cost — buying ACID-like behavior at read time.

## ACID at scale — the cost

Pure ACID across many nodes is **slow** and **fragile**:
- 2-phase commit (2PC) is the canonical "ACID across machines" protocol — blocking and prone to coordinator failure.
- Cross-region ACID requires consensus (Paxos/Raft) → bound by the speed of light over WAN links.

That's why most large-scale systems split:
- **ACID for the core domain** (the order, the payment, the auth record).
- **BASE for everything else** (analytics, feeds, recommendations, denormalized views).

## NewSQL: trying to have both

Spanner, CockroachDB, YugabyteDB, TiDB — all aim to give **ACID guarantees at horizontal scale**. They do it with:
- Synchronized clocks (Spanner's TrueTime) or
- Consensus per shard (Raft) + cross-shard 2PC.

The cost is paid in **write latency**: a cross-region write in CockroachDB is bound by inter-region RTT (~100 ms+). For most apps that's fine. For a high-frequency trading order book, it's not.

## Architect's takeaway

- **ACID and BASE are not competing religions.** They're two engineering choices on a spectrum.
- **Pick per data type** — payments ACID, feed cache BASE, all in the same product.
- **Eventual consistency is a feature, not a bug** — it lets you stay up when the network fights you.
- **Beware "eventual" without specifying which level** — "eventually consistent" can mean "10 ms" or "10 minutes". Demand specifics.
- **Spanner-class NewSQL is real** — if you genuinely need global ACID, it's available, just expensive in latency and dollars.
