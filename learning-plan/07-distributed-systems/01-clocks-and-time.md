# Example 01 — Clocks and time: why you can't trust them

In distributed systems, "what time is it?" is one of the hardest questions. Get this wrong and you ship subtle bugs like "events processed out of order" or "duplicate writes both look newest".

## Three kinds of clocks

### 1. Wall clock (`time.Now()`)

What your phone says. Synchronized by NTP. Used by `System.currentTimeMillis()`, `Date.now()`, `time.Now()`, etc.

**Properties:**
- Aligned to UTC (sort of).
- **Can jump backward** when NTP corrects it.
- **Drifts** between corrections — maybe 100 ms/day.
- Different on every server.

**Don't use for:** measuring elapsed time, ordering events between machines, generating unique IDs.

### 2. Monotonic clock

A counter that only increases. Reset on reboot. Has no relation to wall-clock time.

**Properties:**
- Never goes backward.
- Tells elapsed time correctly even if NTP jumps.
- **Local only** — meaningless across machines.

**Use for:** measuring durations on one machine (timeouts, retries, profiling).

In Go: `time.Now()` returns a value with both wall and monotonic readings; `time.Since(t)` uses monotonic. In Java: `System.nanoTime()`. In C: `clock_gettime(CLOCK_MONOTONIC, ...)`.

### 3. Logical clock

A counter that orders events causally, with no reference to physical time. Lamport clocks, vector clocks, HLCs.

**Properties:**
- Captures **happens-before** relationships.
- Cheap to update (just increment + send with each message).
- Doesn't tell you wall time — only relative order.

**Use for:** ordering events across machines.

## Why wall clocks fail at distributed ordering

```
Server A wall clock: 12:00:00.500
Server B wall clock: 11:59:59.900 (NTP drift, 600 ms slow)

t=0 (real): A writes user.name = "Alice"
t=0 (real): B writes user.name = "Bob" (slightly later)

LWW based on wall clocks: A's write timestamp = 12:00:00.500
                          B's write timestamp = 12:00:00.300 (B thinks)
A wins. But A's write happened FIRST in real time.
```

You silently lose Bob's write because B's clock is slow. This is **last-write-wins lying**.

### Real failures

- **Cassandra LWW**: famously vulnerable to clock skew. A node with a fast clock can overwrite legitimate later writes from a node with a slow clock. The Cassandra community has long-standing best practices about NTP discipline.
- **Kafka log timestamps**: come from the producer's wall clock. Reordered if producers have different clocks.
- **Many real systems**: subtly wrong on the edge case "two events at almost the same time on two machines".

## NTP gives you about ±10 ms typically

Network Time Protocol synchronizes wall clocks against authoritative servers. In data centers with PTP (Precision Time Protocol) you get ~microseconds. Over the public internet, ~10s of milliseconds is typical.

**That's not good enough** for ordering events that arrive at ~ms intervals. If A and B both write within 10 ms, you cannot reliably tell which came first using only their wall clocks.

## Logical clocks: ordering without time

### Lamport timestamps

Each node maintains a counter `t`. On every event, `t++`. When sending a message, include `t`. On receive, set `t = max(local_t, received_t) + 1`.

Result: if event X **causally precedes** Y, then `t_X < t_Y`. The reverse isn't guaranteed — two events with `t_X < t_Y` might be unrelated.

Useful for "total order" of operations across a system. Used internally by many distributed systems.

### Vector clocks

Each node maintains a **vector** of counters, one per other node. On every event, increment your own. On message receive, take element-wise max + 1 of your own.

Vector clocks let you detect **concurrent** events (neither happens-before the other) — useful in leaderless replication (Cassandra-style) for conflict detection.

More memory; more bookkeeping. Used by Riak, DynamoDB internals.

### Hybrid Logical Clocks (HLC)

Combine wall-clock + logical counter. The wall part lets you correlate with real time (for human debugging); the logical part guarantees monotonicity even when the wall clock drifts.

Used by CockroachDB, MongoDB (for causal consistency), YugabyteDB.

## Spanner's TrueTime: the gold standard

Google's Spanner uses **TrueTime**: hardware (GPS + atomic clocks) gives every datacenter a clock with **known uncertainty bounds** (~7 ms typically). Reads return `(earliest, latest)` rather than a single value.

For consistency, Spanner **waits out the uncertainty**: after a write at time T, the system waits T+ε before declaring the write "visible globally", where ε is the clock-uncertainty bound. This makes writes slightly slower but allows true **externally consistent** ordering.

Most other systems can't do this — they don't have atomic clocks. They use HLC + consensus instead.

## Practical advice

### For ordering on one machine
Monotonic clock. Done.

### For ordering events across machines
- **If you have consensus** (Raft / single leader): use the consensus log's order. Done.
- **If you don't** (leaderless / multi-master): use HLC or vector clocks. Don't use wall clocks.

### For ID generation
- **Sortable IDs** that include time: Twitter Snowflake, ULID, UUIDv7. They embed wall time + a counter + a worker ID. Within one worker, monotonic. Across workers, almost monotonic.
- **UUIDv4**: random, no order guarantee. Good for uniqueness only.

### For TTLs and expiry
Wall clock. The brief drift is acceptable; the alternative (logical TTL) doesn't make sense.

### For deduplication windows
Wall clock with a generous buffer. If you dedup over 24 hours, 10 ms clock skew doesn't matter.

## A real story: leap second

In 2012, a leap second was inserted. The Linux kernel had a bug. Cassandra, Java, Hadoop, NTP, Reddit, Mozilla, LinkedIn — all had outages. Wall clocks went backward by one second and code that did `time.Now() > previous_time` looped infinitely or got confused.

The fix in the years since: smear the leap second over many hours so wall clocks never visibly go backward. But the lesson stuck: **don't depend on wall-clock monotonicity**.

## Architect's takeaway

- **Wall clocks: don't use them for ordering.** They lie.
- **Monotonic clocks: use them for elapsed time, locally.**
- **Logical clocks (HLC, vector): use them for cross-machine ordering.**
- **Consensus orders by log position**, not by clock. If you have consensus, use it.
- **NTP is essential but not enough.** Time precision matters; pay for PTP if you need sub-millisecond.
- **Watch out for leap seconds, NTP step jumps, VM hibernation** — all can break wall-clock assumptions.
