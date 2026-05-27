# Example 02 — Replication lag and read-your-own-writes

The most common operational problem with replicas. The fix is conceptually simple; the implementation is annoying.

## The setup

You have one leader and three followers. Reads are load-balanced across followers for scale. Writes go to the leader, replicated asynchronously to followers.

Normal lag: ~10-50 ms. Under load: 1-30 seconds. During an incident: minutes.

## The bug

```
t=0:    User updates profile name "Alice" → "Bob". Write goes to leader.
t=10ms: Write committed on leader. UI shows "Saved!".
t=12ms: User refreshes the page. Read goes to follower (round-robin).
t=12ms: Follower hasn't replicated yet. Returns "Alice".
t=200ms: Follower catches up.
```

The user sees: "I just changed my name to Bob, but the page still shows Alice." They re-save. Get the same. They file a support ticket.

This is the **read-your-own-writes** problem. Eventual consistency is fine — except across one user's session.

## The three fixes

### 1. After a write, read from the leader for that user/session for a short window

Every user has a small "last write timestamp" stored client-side (cookie) or server-side. While `now - last_write_ts < lag_threshold`, route reads to the leader.

```
on write:
  user.last_write_ts = now()

on read:
  if now() - user.last_write_ts < 5s:
      query the leader
  else:
      query a follower
```

Costs: leader handles a small fraction of reads. Simple. **Most popular fix.**

### 2. Sticky to one follower per session

Pin a session to one follower. That follower will have the write the moment it replicates; other followers might still lag, but *this user* only ever reads from theirs.

Doesn't fully solve the problem: the user's specific follower is also async, so they can still get a stale read. Only helps with the *cross-follower* inconsistency where two followers disagree.

### 3. Write to leader, then propagate a version cursor

The leader, on write, returns the resulting **log sequence number (LSN)** or version. Subsequent reads pass the LSN. Followers check: "do I have at least LSN X applied?" If yes, serve. If no, wait or forward to leader.

```
write → leader returns lsn=42
read with header X-Read-After: lsn=42
   follower: my replay position is 50 → serve
   follower: my replay position is 38 → wait or redirect
```

Most correct, most plumbing. Some libraries (Vitess, CockroachDB, MongoDB causal consistency) support this natively.

## Beyond "read your own writes" — other anomalies

### Monotonic reads

User reads "comments count = 10". Next page load reads "comments count = 8". Their data went backward.

Cause: first read hit a fresher follower; second read hit a more lagging one.

Fix: same as above — pin to one follower per session, **or** use a version cursor.

### Consistent prefix

User reads a chat where a reply appears before the message it's responding to. Each message is correct in isolation, but the order is wrong.

Cause: writes hit different shards with different lag; reader sees them out of order.

Fix: **causal consistency** — preserve the happens-before order. Vector clocks, hybrid logical clocks, or single-shard ordering for related entities.

## How big a problem is replication lag really?

Depends on the workload:
- **Read-heavy app, infrequent writes per user** (e-commerce browsing) → almost no impact, fix with strategy 1 (read leader after own write).
- **Active user content** (Twitter, comments, gaming) → strategy 3 (LSN-based) preferred; you often see custom solutions.
- **Money / inventory** → don't replicate-and-read from followers for these queries. Always read leader.

## Detecting it in production

Track:
- **Replication lag in seconds and bytes** on each follower (Postgres: `pg_stat_replication`).
- **Number of reads served per follower** vs leader.
- **Lag spikes** correlated with write bursts, vacuum, big transactions.

A 10-second replication lag during normal hours is a major incident — alert on it.

## When you can't tolerate lag at all

Some workloads need synchronous replication: at least one follower acks the write before the leader returns.

- Postgres: `synchronous_commit = on` + `synchronous_standby_names`.
- MySQL: semi-synchronous plugin.
- Cost: write latency tied to slowest sync follower; if it's down, writes can stall.

You usually only sync to **one** local follower for HA, and async to remote/read replicas.

## Architect's takeaway

- **Replication lag is the default operating mode** for any system with async replicas. Plan for it.
- **Read-your-own-writes** is the most common bug; the cheapest fix is "read from leader for N seconds after this user's write".
- **Track lag in production.** It will spike during incidents, and you need to know.
- For **money or strict-correctness flows**, just read from the leader. The simplicity is worth the lost scale.
- For everything else, **eventual consistency with smart routing** is usually invisible to users.
