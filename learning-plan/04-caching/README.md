# Step 04 — Caching

Caching is the **single highest-leverage performance technique** in distributed systems. It also has the **most subtle bug surface**: invalidation, stampedes, staleness, consistency. The famous quip — *"there are only two hard things in computer science: cache invalidation and naming things"* — is half about caching for a reason.

## Goals

- Pick the right cache strategy (aside / through / behind) for a workload.
- Explain why "just add a cache" can make things slower or worse.
- Know the main eviction policies and which one fits which access pattern.
- Diagnose and prevent: cache stampede, hot key, cache penetration.
- Choose between Redis and Memcached for a real use case.
- Layer caches correctly: browser → CDN → app cache → DB cache.

## Key concepts

1. **Cache strategies** — cache-aside, read-through, write-through, write-back, refresh-ahead.
2. **Eviction policies** — LRU, LFU, FIFO, ARC, random, TTL-only.
3. **Invalidation** — how the cache learns the underlying data changed.
4. **Cache stampede** ("thundering herd") — when many requests miss the same key at once.
5. **Hot keys** — one key receiving a disproportionate share of traffic.
6. **Cache penetration** — requests for keys that don't exist, hitting the DB every time.
7. **Negative caching** — caching "not found" to defeat penetration.
8. **Redis vs Memcached** — when to pick which.
9. **Layered caching** — browser, CDN, app server, distributed cache, DB buffer pool.

## Reading

- **Primer**: Cache, Cache-aside, Write-through, Write-behind, Refresh-ahead.
- **DDIA**: chapter 11 (Stream Processing) discusses materialized caches at scale.
- **Redis docs**: eviction policies page, cluster spec.

## Examples in this folder

- `01-cache-strategies.md` — aside, through, behind, ahead.
- `02-eviction-policies.md` — LRU, LFU, ARC, why "random" sometimes wins.
- `03-cache-stampede.md` — the bug everyone hits once, and how to prevent it.
- `04-hot-keys-and-penetration.md` — two adjacent pathologies and their fixes.
- `05-redis-vs-memcached.md` — pick-correctly cheat sheet.
- `06-layered-caches.md` — the cake of caches between user and DB.

## Self-check

1. Why is cache-aside the most common pattern in practice?
2. You have a "popular product" page where 10k req/sec hit one key. The cache TTL expires. What happens next, and how do you stop it?
3. When does write-through actually help (vs. write-around)?
4. Why doesn't LRU work well for an access pattern with a periodic full scan?
5. Why might Redis be a wrong choice for a 10 TB read-only cache?
