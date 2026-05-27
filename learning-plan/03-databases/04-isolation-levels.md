# Example 04 — Isolation levels: the four levels and the anomalies they prevent

Concurrency is where transactions get exciting. The ANSI SQL standard defines four isolation levels, each preventing more concurrency anomalies than the last, but each at higher cost.

## The four anomalies (the problems)

### 1. Dirty read

Transaction T1 writes a value. T2 reads it before T1 commits. T1 rolls back. T2 now has bogus data.

```
T1: UPDATE accounts SET balance = 0 WHERE id = 1;
T2: SELECT balance FROM accounts WHERE id = 1;  --> reads 0
T1: ROLLBACK;
T2: ...used 0 to make a decision. Wrong.
```

### 2. Non-repeatable read

T1 reads a row twice and gets two different values (because T2 committed an update between them).

```
T1: SELECT price FROM products WHERE id = 5;  --> 100
T2: UPDATE products SET price = 200 WHERE id = 5;  COMMIT;
T1: SELECT price FROM products WHERE id = 5;  --> 200  -- different!
```

### 3. Phantom read

T1 runs the same range query twice and gets a different set of rows.

```
T1: SELECT COUNT(*) FROM orders WHERE total > 100;  --> 50
T2: INSERT INTO orders (total) VALUES (500);  COMMIT;
T1: SELECT COUNT(*) FROM orders WHERE total > 100;  --> 51  -- a phantom appeared!
```

### 4. Write skew (not in the ANSI standard but very real)

Two transactions read overlapping data, make decisions based on what they read, and write disjoint rows. Both commit. The combined result violates an invariant that each transaction individually maintained.

```
Invariant: at least 1 doctor must be on-call.
On-call: { Alice, Bob }.

T1 (Alice): I see both on-call. I'll go off-call. UPDATE Alice off.
T2 (Bob):   I see both on-call. I'll go off-call. UPDATE Bob off.

Both commit. Zero doctors on call. Invariant broken.
```

## The four levels (the solutions)

| Level                 | Prevents dirty read | Prevents non-repeatable read | Prevents phantom read | Prevents write skew |
|-----------------------|---------------------|-------------------------------|------------------------|----------------------|
| Read Uncommitted      | ❌                  | ❌                            | ❌                     | ❌                   |
| Read Committed        | ✅                  | ❌                            | ❌                     | ❌                   |
| Repeatable Read       | ✅                  | ✅                            | ❌ (ANSI) / ✅ (some DBs) | ❌                |
| Serializable          | ✅                  | ✅                            | ✅                     | ✅                   |

## What each DB actually does

The standard is mostly aspirational — engines pick what they implement:

| Database          | Default level     | Notes                                                |
|-------------------|--------------------|------------------------------------------------------|
| PostgreSQL        | Read Committed     | Repeatable Read prevents phantoms via MVCC snapshot |
| MySQL InnoDB      | Repeatable Read    | Prevents phantoms via next-key locking              |
| Oracle            | Read Committed     | Snapshot isolation                                  |
| SQL Server        | Read Committed     | Supports snapshot mode                              |
| MongoDB (4.0+)    | Snapshot (transactions) | Multi-document transactions are snapshot       |
| CockroachDB       | Serializable       | Strict — uses optimistic concurrency control       |
| Spanner           | Serializable       | Strict — uses TrueTime + 2PL                        |

> Postgres at Read Committed gives you a "fresh snapshot per statement". MySQL at Repeatable Read gives you a "fresh snapshot per transaction". Both prevent dirty reads and non-repeatable reads, but Postgres allows phantoms within a transaction, MySQL prevents them.

## Snapshot Isolation: the practical sweet spot

Most modern DBs use **MVCC** (multi-version concurrency control). When you read, you see a consistent snapshot of the DB at the moment the transaction started.

Snapshot Isolation prevents dirty/non-repeatable/phantom reads, **but it does not prevent write skew.** This is what Postgres calls "Repeatable Read" — strictly weaker than Serializable.

## Solving write skew

Options, in order of cost:

### 1. SELECT FOR UPDATE

Explicitly lock the rows you're reading-and-might-write-against. The second transaction blocks.

```sql
BEGIN;
SELECT * FROM doctors WHERE on_call = true FOR UPDATE;  -- locks these rows
-- decide
UPDATE doctors SET on_call = false WHERE id = 1;
COMMIT;
```

Works at any isolation level. Adds blocking.

### 2. Materialize the invariant

Add a row that represents the invariant and lock it:

```sql
SELECT FROM oncall_count WHERE count >= 1 FOR UPDATE;  -- block others
-- then change a doctor's status
```

### 3. Use Serializable isolation

At Serializable, Postgres uses **SSI** (Serializable Snapshot Isolation). Both transactions execute optimistically; if a serialization conflict is detected, one is **aborted with a serialization failure** and the app retries.

The app must be prepared to retry — but the DB guarantees correctness.

## Performance cost

Going from Read Committed to Serializable is **not free**:

- More aborted transactions → retries in app code.
- More locking → throughput drops, deadlocks more likely.
- p99 latency rises.

For most OLTP workloads, **Read Committed is the right default**, with `SELECT FOR UPDATE` used surgically where invariants matter (money, inventory, scheduling). Serializable is the right default when **correctness must trump throughput** (financial ledgers, regulated systems).

## Architect's takeaway

- **Pick the weakest isolation level that still preserves your invariants.** That's the right answer.
- **Read your DB's docs**, not just the ANSI standard — implementations differ.
- **Write skew is the most often-forgotten anomaly.** It causes scary bugs in scheduling, inventory, and resource-allocation code.
- For **money**, use Serializable or surgical row locks. The performance hit is the price of correctness.
- Test concurrency explicitly — most bugs only show up under load.
