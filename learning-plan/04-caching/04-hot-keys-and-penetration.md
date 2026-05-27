# Example 04 — Hot keys and cache penetration

Two adjacent pathologies that look similar but have different causes and fixes.

## Hot key

One key receives a disproportionate share of all traffic.

Examples:
- `product:iPhone15` during a launch — 100k req/sec while every other product gets 100.
- `user:elon` on Twitter — millions of timeline lookups hit this user.
- `game:championship_final` — score lookups overwhelm one cache shard.

### Why it hurts

In a sharded Redis cluster, that one key lives on **one node**. That node is now red-hot while the other 9 nodes are bored. Result:
- Single-shard CPU saturates.
- Network egress on that node saturates.
- Cluster looks "60% utilized on average" but is actually broken.
- Adding more shards doesn't help — the key still lives on one.

### Fixes

#### 1. Replicate the hot key

Store the same value under multiple keys: `product:iPhone15`, `product:iPhone15:r1`, `product:iPhone15:r2`. App picks a random suffix on each read. Reads spread across all nodes holding a copy.

```python
suffix = random.randint(0, N_REPLICAS - 1)
value = redis.get(f"product:iPhone15:r{suffix}")
```

Writes are slightly more expensive (write all replicas), but reads scale linearly.

#### 2. Local cache (L1) on each app server

Even a 30-second in-process LRU cache absorbs the hot read locally. Now Redis sees one request per app-server per 30s instead of millions.

```go
// in-process L1 in front of distributed L2
value, ok := localCache.Get("product:iPhone15")
if !ok {
    value = redis.Get("product:iPhone15")
    localCache.SetWithTTL("product:iPhone15", value, 30*time.Second)
}
```

Trade-off: short window of staleness across servers. For most hot reads, fine.

#### 3. Detect dynamically (request coalescing at the edge)

Some CDNs and reverse proxies have **request collapsing**: if 1000 requests for the same URL arrive in the same millisecond, only one is forwarded to origin; the response fans out to all 1000 callers. Cloudflare, Varnish, and Nginx (with proxy_cache_lock) all support this.

#### 4. Pre-compute hot results

If `product:iPhone15` is expensive to compute (joins, aggregation), pre-compute on write and store the rendered version. Reads are cheap key lookups.

## Cache penetration

Requests for keys that **never** exist in cache **or** the underlying DB.

Examples:
- An attacker scans random user IDs: `user:abc`, `user:xyz`, `user:123456789`...
- A buggy client requests deleted products in a loop.
- An old URL gets indexed by Google; long after the page is gone, search bots still hit it.

### Why it hurts

```
GET user:abc → cache miss → DB query → no row → DB returns nothing
                                                ↓
The app correctly returns 404. But nothing is cached.
Next request for user:abc → cache miss → DB query → ...
```

Every request hits the DB. The cache contributes nothing. A small bot can DoS your DB.

### Fixes

#### 1. Negative caching

Cache the *absence* of the result, not just the result.

```python
def get_user(uid):
    cached = cache.get(f"user:{uid}")
    if cached == "MISS_SENTINEL":
        return None  # cached negative
    if cached is not None:
        return cached
    user = db.get_user(uid)
    if user is None:
        cache.set(f"user:{uid}", "MISS_SENTINEL", ttl=60)  # short
    else:
        cache.set(f"user:{uid}", user, ttl=600)
    return user
```

Use a **short TTL** for negative entries (60s typical) so a legitimately created user shows up reasonably fast.

#### 2. Bloom filter at the cache front

A Bloom filter says "this key definitely does not exist" in O(1) memory.

```
exists_filter = BloomFilter(items=all_user_ids, fpp=0.01)

def get_user(uid):
    if not exists_filter.contains(uid):
        return None  # zero DB hit
    ...
```

Sub-millisecond lookup. False-positive rate ~1% (acceptable — they fall through to the normal path, which then negative-caches).

Rebuild the Bloom filter periodically (hourly, daily) — additions are cheap; deletions require a counting Bloom filter or rebuild.

#### 3. Rate-limit on miss

If the cache miss rate for a single client exceeds a threshold, rate-limit them. Defends against probing attacks.

#### 4. Authorization at the edge

The simplest fix: require auth before lookup. If `user:xyz` requires a valid token belonging to xyz, no random attacker can probe IDs.

## A combined attack pattern: hot-key penetration

Attacker crafts a hot probe at a non-existent key (e.g., shows up in a viral tweet pointing to `/products/666666`). Now you have a **hot key** that **doesn't exist** — every request misses, every miss hits DB, DB melts.

Mitigations stack:
- Negative cache the 404.
- Bloom filter to skip the DB entirely.
- Rate limit at the edge.
- WAF rules to drop the bad URL pattern.

## Architect's takeaway

- **Hot key** = uneven distribution. Spread it with replicas, L1 cache, or pre-computation.
- **Penetration** = misses without recourse. Negative-cache, Bloom-filter, or rate-limit.
- These two often combine — defend with the layered approach above.
- **Measure both**: track per-shard CPU (hot key signal) and miss-rate per endpoint (penetration signal). They're invisible until you graph them.
