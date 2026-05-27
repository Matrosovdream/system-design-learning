# Example 03 — Two-phase commit (2PC): what it is and why most systems avoid it

The classic "distributed transaction" protocol. Useful when you must atomically update multiple databases. Avoided because it's slow and fragile. Worth understanding so you know what to choose **instead**.

## The problem 2PC solves

You're transferring $100 from account A (in DB1) to account B (in DB2).

Naive:
```
DB1: UPDATE accounts SET balance -= 100 WHERE id = A;
DB2: UPDATE accounts SET balance += 100 WHERE id = B;
```

What if DB1 succeeds and DB2 crashes? You've lost $100 in real money. Need atomicity across two systems.

## The 2PC protocol

A **coordinator** orchestrates two phases across **participants** (the databases).

### Phase 1: Prepare

```
coordinator → DB1: PREPARE (debit $100 from A)
coordinator → DB2: PREPARE (credit $100 to B)
```

Each participant:
- Tentatively applies the change.
- Locks the affected rows.
- Writes to disk the fact that "I'm prepared to commit transaction T".
- Replies VOTE-COMMIT (yes, ready) or VOTE-ABORT (something went wrong).

### Phase 2: Commit (or abort)

If **all** participants voted yes:

```
coordinator → DB1: COMMIT
coordinator → DB2: COMMIT
```

Each participant releases locks, makes the change permanent, replies ACK.

If **any** voted no:

```
coordinator → DB1: ABORT
coordinator → DB2: ABORT
```

Each participant rolls back, releases locks.

## Why it's atomic

After phase 1, every participant has **durably promised** to commit if asked. Even if a participant crashes between phases, on restart it reads its "prepared" log entry, asks the coordinator "what happened to T?", and commits or aborts accordingly.

The coordinator's logged decision is the source of truth. Once the coordinator decides, the decision is final.

## Why 2PC is hated

### 1. It's blocking

Between PREPARE and COMMIT, participants hold locks on the rows involved. If the coordinator crashes after PREPARE but before COMMIT, **participants are stuck**:
- They can't commit (don't know the decision).
- They can't abort (might violate the agreement with the coordinator).
- They wait.

The locks block every other transaction touching those rows. Throughput collapses.

Recovery requires manual intervention or a coordinator restart that replays its log.

### 2. The coordinator is a SPOF

If the coordinator dies forever (unlikely but possible), prepared transactions are **stuck forever** in a half-committed state. DBAs end up running emergency SQL to break the protocol.

Mitigation: replicate the coordinator (now you're running two consensus systems — one for replication, one for 2PC).

### 3. It's slow

A single 2PC commit is two round-trips to **every** participant. With 2 participants in the same DC: ~2 ms. Across regions: 200+ ms.

Multiply by every committed transaction. Throughput drops 10-100× compared to single-DB transactions.

### 4. It doesn't compose well with replicated databases

A "single" participant might itself be a Raft group. Now the commit involves Raft within each shard *and* 2PC across them. The latency stacks.

### 5. It can't handle every failure

If a participant has voted COMMIT and then crashes, on restart it must replay and commit. But if it crashes for a long time, and the coordinator times out, weird recoveries are possible.

XA (the standard for 2PC) has a long history of corner-case bugs in real implementations.

## When 2PC is still used

- **Inside one database** that internally uses 2PC for cross-shard transactions (CockroachDB, Spanner) — they take on the operational burden so you don't.
- **Java EE / XA transactions** in legacy enterprise systems.
- **PostgreSQL's `PREPARE TRANSACTION`** — manual 2PC across multiple Postgres instances. Rarely used.
- **JMS / WS-AtomicTransaction** in enterprise messaging.

## What modern systems use instead

### Saga (next example)

Don't try to be atomic across services. Instead, do each step **eventually** and provide **compensating actions** if a later step fails. Trade strict atomicity for availability and decoupling.

### Outbox + idempotency

```
1. In one local DB transaction:
   - Apply the local change.
   - Insert into an "outbox" table the event to send.
2. Commit.
3. Background relay reads outbox and publishes to other services.
4. Other services consume and idempotently apply.
```

The local DB transaction is atomic. Cross-service consistency is **eventual** but reliable. No 2PC.

### Single-DB-with-many-tables

If business correctness requires atomicity, sometimes the right answer is **don't split the data**. Put related entities in one DB. Use local transactions.

This is the Stripe / banking approach: keep money in one DB cluster with rigorous ACID, accept that scaling means vertical hardware and careful schema design, not microservices.

### Spanner-class NewSQL

Hide 2PC inside a NewSQL system that pays the latency cost for you (via Paxos/Raft + 2PC + TrueTime). You write transactions that look local; the DB handles distribution. Cost: latency and dollars.

## The architect's mental model

Think of cross-service atomicity as a **spectrum**:

```
strong, slow                                        weak, fast
   ↑                                                    ↑
[Spanner/CockroachDB] - [2PC] - [Saga] - [Outbox] - [eventual w/ no coordination]
```

Most production systems sit between Saga and Outbox. Strict 2PC is rare in modern designs.

## Architect's takeaway

- **2PC works.** It's just expensive.
- **The coordinator-blocking and locking** problems are why it doesn't scale.
- **Most modern systems avoid 2PC** in favor of sagas, the outbox pattern, or NewSQL.
- **If you genuinely need cross-service ACID atomicity**, consider whether the data should be split at all. Sometimes the right answer is "one DB, multiple tables".
- **Inside one NewSQL DB**, you get the appearance of distributed ACID without writing 2PC yourself. That's usually the right trade.
