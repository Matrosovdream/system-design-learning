# Case Study 10 — Design a stock exchange / order matching engine

The most extreme version of "every microsecond matters". Probes low-latency design, fairness, ordering, and audit. Very different from CRUD-shaped systems — single-process matching is the rule, not distribution.

## Problem statement

Build a system that:
- Accepts orders from traders (buy/sell, market/limit, quantities).
- Matches buyers and sellers fairly.
- Maintains the order book.
- Publishes market data (price changes, trades) to subscribers.
- Records everything immutably for audit.

## Clarifying questions

1. **Scale**: orders/sec, traders, instruments?
2. **Latency target**: microseconds (HFT) or milliseconds (retail)?
3. **Order types**: market, limit, stop, iceberg, etc.?
4. **Fairness**: strict price-time priority?
5. **Multi-venue**: one matching engine per instrument or all in one?
6. **Audit / regulation**: full trail of every event?

**Assumed answers:**

- 1M orders/sec peak; 10k traders; ~5000 instruments.
- Median latency target: sub-100 microseconds for matching.
- Market and limit orders; stop orders later.
- Strict price-time priority.
- One matching engine per instrument (no cross-instrument matching).
- Full audit trail for SEC-style compliance.

## Functional requirements

- Submit / amend / cancel orders.
- Match orders following price-time priority.
- Publish trades and order book updates in real time.
- Persistent audit log.

## Non-functional requirements

- **Matching latency: 50-100 μs (microseconds).** Far below normal app design.
- Determinism: same orders in same order → same result.
- Audit: every event recorded immutably.
- High availability: failover within seconds.
- Fairness: no trader can be unfairly disadvantaged.

## Capacity estimation

```
1M orders/sec across all instruments.
~5000 instruments → average 200 orders/sec per instrument.
Hot instruments: maybe 10k orders/sec.

Storage:
  1M ops/sec × ~200 bytes/event = 200 MB/sec of events.
  ~17 TB/day if every event recorded.
  Realistic: persist trades and snapshots; reconstruct events from log.
```

## High-level architecture

The architecture is **unlike** most other case studies. Distributed systems make latency worse, not better. So exchanges concentrate matching in a single process per instrument and pay extreme attention to in-process performance.

```
[trader gateway: order entry, FIX protocol]
   │
   ▼
[Order router]   ← decides which matching engine to send to
   │
   ▼  (sharded by instrument symbol)
[Matching engine per instrument or per shard]
   │   - single-threaded, in-memory order book
   │   - sequential processing of orders for determinism
   │   - writes to:
   │       ├─► [Event log: append-only, replicated, e.g., Aeron or custom]
   │       └─► [Multicast UDP to market-data subscribers]
   │
[Persistent storage]
   │   replays event log → snapshots periodically
   │
[Market data fan-out]
   │
   ▼
[traders & data subscribers]
```

## Deep dive: the matching engine

Single-threaded, per instrument. Why?

- **Determinism**: orders processed in a fixed order.
- **No locking**: single thread → no contention.
- **Cache-friendly**: all data in CPU caches → microsecond latencies achievable.

Distribution / parallelism would add coordination overhead and non-determinism. For matching, **fewer cores doing serial work > many cores doing parallel work with locks**.

### The order book

Two sides:
- **Bids** (buy orders): sorted by price descending, then by time ascending.
- **Asks** (sell orders): sorted by price ascending, then by time ascending.

```
Asks (sell)
$101 - 100 shares (T1)
$102 - 50 shares (T2)
$102 - 200 shares (T3)
$105 - 1000 shares (T4)
─────────────
$100 - 50 shares (T5)
$99  - 100 shares (T6)
$99  - 200 shares (T7)
$95  - 500 shares (T8)
Bids (buy)
```

Best bid = highest. Best ask = lowest. Spread = ask - bid.

### Limit order: typical case

```
Order: BUY 100 shares at $102 (limit).

Engine:
1. Look at best ask: $101.
2. Best ask <= our limit ($102)? Yes → match.
3. Trade: 100 shares at $101. (Crossed at the resting order's price.)
4. Reduce ask side by 100 shares.
5. Publish trade event.

If the order was BUY 300 at $102:
- Match 100 at $101 (T1's order). Order remains: BUY 200 at $102.
- Match 50 at $102 (T2's order). Order remains: BUY 150 at $102.
- Match 150 of T3's 200 at $102. T3 has 50 remaining.
- Our order is fully filled.
- T3 still has a resting order for 50 at $102.
```

This is **strict price-time priority**: best price first; same price, earliest order first.

### Market order

Like a limit order with no price constraint — match against the order book at whatever the best price is, until filled or order book exhausted.

### Order data structure

For O(1)+O(log N) performance:

```
struct PriceLevel {
    price: int64,
    orders: deque<Order>,    // FIFO, time-ordered
    total_quantity: int64,
}

asks: sorted map<price, PriceLevel>     // ascending
bids: sorted map<price, PriceLevel>     // descending
order_lookup: hashmap<order_id, *Order> // for cancellation
```

Inserting a new order: O(log levels). Matching: pop from best price level. Canceling: O(1) lookup, then remove from price level's deque.

In C++ / Rust: with cache-friendly memory layout, all of this runs in tens of nanoseconds per order.

## Deep dive: durability + replication

Single process is fast but fragile. If the matching engine crashes, you've lost the order book.

### Approach 1: synchronous append-to-log before matching

```
on order received:
  1. Write to append-only log (synchronously).
  2. Match.
```

Cost: adds ~10-100 μs of log-write latency. Trade-off worth it for durability.

### Approach 2: replication via deterministic state machine

Two (or more) matching engines run in parallel. Both receive the same input stream (orders) in the same order. Both run the same deterministic logic. Both produce the same output. One is "primary" — its output is used; the other is "hot standby" — its output is dropped but ready to take over.

Aeron's "cluster" and LMAX's "Disruptor" patterns use this. Sub-microsecond failover possible.

```
[order stream] → [primary engine] → [output]
              ↘ [secondary engine] → [drop]
              ↘ [tertiary engine]  → [drop]

If primary fails: secondary becomes primary instantly.
```

This is **state-machine replication**. The key insight: deterministic inputs + deterministic logic = deterministic output → replicas converge perfectly.

## Deep dive: market data dissemination

Every order book change → publish to subscribers (traders, ticker apps, regulators).

At HFT scale, subscribers expect microseconds. UDP multicast is the standard transport:

```
matching engine → multicast UDP → subscribers
```

UDP because:
- TCP would add per-subscriber connection overhead and latency.
- Multicast lets one packet reach all subscribers atomically.
- Drops are acceptable (subscribers handle gaps).

For gap recovery: a separate "gap fill" service handles "I missed packet N — send it" requests.

For retail / less time-sensitive feeds: WebSocket, REST polling, FIX protocol over TCP — slower, simpler.

## Deep dive: sequence numbers and ordering

Every event has a strictly monotonic sequence number. Used for:
- Replay (start from seq=12345).
- Gap detection (subscribers see seq=4, then seq=6 → request seq=5).
- Reconciliation (compare your event log to ours at seq=X).

The sequence number is the **canonical order** of events. Time stamps are nice-to-have but the sequence is authoritative.

## Deep dive: fairness

In an exchange, fairness is a regulatory requirement.

### Strict FIFO at the matching engine

Orders are processed in the order they arrive. Two orders at the same price → the earlier one wins.

### Random network jitter is the enemy

Trader A and B both send orders at "the same time". A's order arrives at the gateway 1 μs before B's. A's order matches first. B might feel that's unfair.

**Mitigations**:
- **Co-location**: all traders' machines in the same data center, close to the matching engine. Equalizes network latency.
- **Speed bumps**: deliberately delay all orders by a small constant (e.g., 350 μs) to homogenize arrival jitter. Used by IEX.
- **Auction-based matching**: instead of continuous matching, accept orders for an interval, then match all together. Avoids latency arms race entirely. Used in some opening/closing auctions.

### Order modification rules

Modifying an order's price = losing your time priority. Modifying quantity-down = keep priority. These rules prevent "queue jumping" tricks.

## Deep dive: persistent storage

The event log is the source of truth. It's persisted to disk synchronously (often to fast NVMe + replicated).

Periodically, snapshot the order book to disk. On restart: load most recent snapshot + replay events since.

Long-term archive: events stored in append-only files, compressed and shipped to cold storage daily. Compliance retention: 5-7+ years.

## Trade-offs discussion

### Why not distributed?

Distribution adds:
- Network hops (microseconds).
- Coordination overhead (consensus rounds, locking).
- Non-determinism (which message arrives first?).

For matching, you want **determinism** and **microsecond latency**. A single fast process beats a distributed system for this. The throughput limit is in the millions of orders/sec on modern hardware, which is plenty for the world's biggest exchanges per instrument.

### Sharding across instruments

Different instruments don't interact in matching. So shard the matching layer: each engine handles its set of instruments. Order routers send orders to the right engine.

Sharding by instrument is parallelism without coordination — clean.

### State-machine replication

The right way to get HA without losing determinism. Replicas process the same stream; one's output is used; failover is instant.

### UDP multicast for market data

Loss-tolerant for performance. Gap recovery via separate path. Standard in the industry.

## Common follow-up questions

1. **"How do you handle a circuit breaker (trading halt)?"**
   The engine has a special state: when triggered (e.g., 10% price move in 1 min), reject all incoming orders for the halt period. Resume with auction at end.

2. **"How do you support stop orders?"**
   Stops are held outside the order book until triggered (price condition met). The engine watches each new trade against held stops; if triggered, the stop becomes a market or limit order.

3. **"How do you prevent self-trading?"**
   The engine knows which orders belong to the same trader. Match logic skips matches between same trader (or generates a "self-trade" event for compliance).

4. **"What if the matching engine is 10× slower than required?"**
   - Profile: hot path mostly in CPU caches?
   - Reduce allocation: use object pools.
   - Simpler data structures: skip-list or custom B-tree of price levels.
   - Hardware: faster CPU + NIC.
   - Co-locate gateways.

5. **"How do regulators reconstruct events?"**
   Append-only event log + snapshots + replay. Full deterministic reconstruction. Every modification, cancellation, trade — all there.

6. **"How is this different from a database?"**
   A DB optimizes for general queries. A matching engine optimizes for one thing: match orders fast. It's specialized; not generic. The order book is in memory; persistence is via append-only log, not B-tree updates.

7. **"How do you scale market data to millions of subscribers?"**
   Multicast scales trivially (one packet → N subscribers). For non-multicast (consumer subscribers on the internet), fan out via WebSocket gateways, possibly tiered.

## Key takeaways

- **Stock exchanges are about microseconds, not milliseconds.** Distribution typically hurts.
- **Single-threaded matching per instrument** is the standard. Deterministic, fast.
- **State-machine replication** = HA without sacrificing determinism.
- **Strict FIFO + sequence numbers** = fairness and auditability.
- **UDP multicast** = low-latency market data dissemination.
- **The architecture is unusual** for "system design": minimal distribution, maximal in-process optimization. It's a reminder that **good architecture is context-dependent**.
- Studying this teaches the limits where conventional patterns don't apply.
