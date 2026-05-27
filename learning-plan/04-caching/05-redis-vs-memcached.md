# Example 05 — Redis vs Memcached: a real comparison

Both are in-memory key-value stores. They look interchangeable on a first pass. They are not.

## At a glance

| Feature              | Redis                                 | Memcached                          |
|----------------------|---------------------------------------|------------------------------------|
| Data structures      | strings, lists, sets, sorted sets, hashes, streams, hyperloglog, bitmaps, geo | strings only |
| Persistence          | Yes (RDB snapshots + AOF log)         | No — purely in-memory              |
| Replication          | Yes (primary/replica, async)          | No                                 |
| Clustering           | Yes (Redis Cluster, hash slots)       | Yes (client-side sharding only)    |
| Transactions         | Yes (MULTI/EXEC, optimistic)          | No                                 |
| Pub/Sub              | Yes                                   | No                                 |
| Scripting            | Yes (Lua, Redis Functions)            | No                                 |
| Threading model      | Mostly single-threaded (per process)  | Multi-threaded by default          |
| Memory model         | One process per Redis instance        | One process, many threads          |
| Eviction policies    | 8 (LRU/LFU/TTL variants)              | Slab allocator + LRU per slab      |
| Maximum value size   | 512 MB                                | 1 MB (configurable)                |

## When Redis wins

- **You need data structures.** Sorted sets for leaderboards, hashes for partial-object updates, streams for log/queue patterns, geo for location apps. This alone is usually the reason Redis is picked.
- **You need persistence** — survive a restart with at least the recent state.
- **You need pub/sub or queueing** — Redis Streams, BLPOP, etc., are real messaging primitives.
- **You need clustering with sharding handled by the server.**
- **You want atomic multi-key operations** (Lua scripts).
- **You need replication** for HA or read-scale.

## When Memcached wins

- **You only need plain string KV.** Memcached is simpler and slightly faster for this case.
- **You want true multi-threading per instance.** Memcached uses all cores in one process; Redis you scale via multiple processes (Redis Cluster does this but adds setup).
- **You want the smallest, dumbest, most predictable cache.** Memcached restarts in milliseconds; there's almost nothing to misconfigure.
- **Massive RAM per node, simple workloads.** Memcached's slab allocator is excellent for uniform-sized values.

## Common mistakes

### Using Redis when Memcached would do

A team has 50 GB of "just cached strings" with no need for sorted sets or persistence. They run a Redis Cluster, manage primary/replica failover, tune AOF, deal with `MOVED` redirects. Memcached on a few boxes would be 10× simpler.

### Using Memcached when Redis would do

A team needs a leaderboard. They store it as a serialized list in Memcached, GET/parse/sort/SET each update. Memcached can't do `ZADD`. This pattern races, loses updates, performs badly. Redis sorted sets are the right tool — one command per update, O(log N).

### Treating Redis as a database

Redis durability is **not** the same as Postgres durability. AOF in `appendfsync everysec` mode (the recommended setting) means up to 1 second of writes can be lost on power loss. `appendfsync always` is durable but slow. **Redis is a cache and message broker — not a primary DB.**

For "persistent KV", look at LMDB, RocksDB, FoundationDB, or a managed service.

### Storing huge values

Both have a value-size limit (Memcached: 1 MB default; Redis: 512 MB but you really shouldn't). Caching a 20 MB JSON blob bloats memory and crushes network throughput. Cache the *hot subset* or break into pieces.

## A decision flow

```
Need data structures (lists, sets, sorted sets, hashes)?
   yes → Redis
   no  → continue

Need persistence?
   yes → Redis (or a persistent KV store)
   no  → continue

Need replication / clustering?
   yes → Redis (or Memcached with client-side sharding)
   no  → continue

Pure transient KV at high throughput?
   → Memcached (or Redis, your call)
```

In 2026, **Redis is the default** most teams pick — even when they don't need most of its features. That's fine; it's mature and ubiquitous. Memcached remains an excellent choice for "cache only, nothing else".

## A note on alternatives

- **Dragonfly** — Redis-compatible, multi-threaded, claims much higher throughput. Worth testing.
- **KeyDB** — multi-threaded fork of Redis.
- **Hazelcast / Apache Ignite** — in-memory data grids, much richer feature sets, Java-centric.
- **Aerospike, ScyllaDB** — hybrid memory+SSD for *very large* hot stores.

## Architect's takeaway

- **Redis is the default for general caching.** Pay for its features by paying for its complexity.
- **Memcached is the right choice for "simple KV at scale".** Don't overcomplicate.
- **Don't use Redis as a primary DB.** Persistence is best-effort, not the same as RDBMS durability.
- **Match data structure to use case.** If you find yourself serializing/parsing inside cache calls, you're probably using the wrong data structure or the wrong tool.
