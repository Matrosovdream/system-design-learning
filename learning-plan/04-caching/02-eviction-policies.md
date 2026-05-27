# Example 02 — Eviction policies: choosing which key dies

A cache has finite memory. When it's full and a new key comes in, **something must go**. The eviction policy decides which.

## The classics

### LRU — Least Recently Used

Track the order keys were last accessed. Evict the one untouched longest.

- **Implementation**: doubly-linked list + hashmap. O(1) access and eviction.
- **Wins when**: recent access predicts future access (the common case).
- **Loses when**: a one-off full scan walks every key (cold scan evicts your hot set).

**Default policy in Redis (`allkeys-lru`)**, Memcached (`-M`), and most caching libraries.

### LFU — Least Frequently Used

Track access count per key. Evict the least-counted.

- **Implementation**: needs counters + a way to age them (raw counts give pathological behavior — keys hot 6 months ago dominate forever).
- **Wins when**: there's a stable "popular set" — old hot keys stay hot.
- **Loses when**: workload shifts (yesterday's hot key is today's cold, but its count is huge).

Redis offers `allkeys-lfu` with frequency aging.

### FIFO — First In, First Out

Evict in insertion order, ignoring access pattern.

- **Implementation**: trivial — circular buffer.
- **Wins when**: you don't trust access patterns (e.g., adversarial inputs).
- **Loses**: usually, because it ignores reality.

Rare in practice except for highly uniform workloads.

### Random

Evict a random key.

- **Wins when**: workload is uniformly distributed.
- **Surprising fact**: under some real workloads, random is within a few % of LRU and uses **far less metadata** (no linked list to update on every read).

Available in Redis (`allkeys-random`). Used by Memcached's "slab" mechanic internally.

### TTL-only

Don't evict by access at all. Set a TTL on every key; expire by clock.

- **Wins when**: items have a natural freshness deadline (sessions, OTPs, short-lived tokens).
- **Loses**: if memory fills before TTL fires, writes start failing.

## Advanced policies

### ARC — Adaptive Replacement Cache

Used in ZFS, some L2 caches. Maintains two LRU lists (recently used, frequently used) and dynamically rebalances between them. Beats LRU on most workloads.

Not in Redis or Memcached; appears in some specialized stores.

### W-TinyLFU (used by Caffeine, the Java cache library)

Admission policy: a new key gets in **only if** it's likely to be hotter than the candidate eviction. Uses a sketch counter.

State of the art for many workloads. Significantly higher hit rates than plain LRU at the same memory budget.

### Segmented LRU / 2Q

Two LRU regions: a small "probationary" zone where new keys live, and a "protected" zone for keys accessed twice. Cold scans only pollute the probationary zone.

Good defense against scan-heavy workloads.

## Pathologies you should recognize

### Sequential scan pollution

```
hot keys: A, B, C, D, E (95% of traffic)
some job:  SELECT * FROM huge_table  ← walks 10M keys

Under LRU: the 10M keys evict A-E. Your hot set is gone.
Under LFU/2Q: the scanned keys hit once → low priority → kept out.
```

This is why analytics queries on the OLTP DB can ruin cache hit rate, even if the data fits — they pollute the buffer pool.

### Working set bigger than cache

If your hot working set is 100 GB and your cache is 50 GB, **no eviction policy saves you**. You'll be missing constantly. The fix is more memory, not a smarter policy.

Useful rule: aim for **>90% hit rate** in production. Lower than that, your cache is mostly wasted effort.

## Redis eviction policies in detail

| Policy             | Considers       | Description                                                |
|--------------------|-----------------|------------------------------------------------------------|
| `noeviction`       | —               | Writes fail when full (default!)                            |
| `allkeys-lru`      | all keys, recency| LRU across the whole keyspace                              |
| `allkeys-lfu`      | all keys, frequency | LFU across the whole keyspace                          |
| `volatile-lru`     | keys with TTL, recency | LRU but only among keys with a TTL set              |
| `volatile-lfu`     | keys with TTL, frequency | LFU on TTL'd keys                                |
| `allkeys-random`   | all keys, random | Random eviction                                            |
| `volatile-random`  | TTL keys, random | Random among TTL'd keys                                    |
| `volatile-ttl`     | TTL keys, time   | Evict the one with the shortest TTL remaining             |

**The `noeviction` default is a trap.** Production Redis without TTLs will OOM-error writes once full. Always set a maxmemory and an eviction policy.

## How to pick

1. **Mixed real-world workload** → `allkeys-lru` (or LFU if you can tolerate the slightly higher overhead).
2. **Sessions / short-lived tokens** → `volatile-ttl` (let things expire by clock).
3. **Cache that absolutely cannot lose certain keys** → `volatile-lru` (mark must-keep keys with no TTL, only evict TTL'd ones).
4. **Truly uniform access** → `allkeys-random` (cheaper).
5. **Highly scan-heavy traffic** → consider an L2 admission cache (Caffeine-class) instead of pure LRU.

## Architect's takeaway

- **LRU is the safe default**, but it's not always best. Test hit rate.
- **`noeviction` is dangerous** in production — it converts "cache full" into "writes broken".
- **Eviction policy ≠ TTL.** TTL ages out stale data; eviction handles memory pressure. You usually want both.
- **More memory beats a smarter policy** if your working set doesn't fit. Diagnose before tuning.
- **Modern caches (Caffeine, etc.)** beat naive LRU significantly — worth considering for in-process caching.
