# Example 03 — Cache stampede ("thundering herd")

The bug nearly every cache-using team hits once. A hot cache entry expires; thousands of concurrent requests all miss, all hit the DB, all try to re-populate the cache. The DB falls over.

Also called **dog-pile**, **cache miss storm**, **thundering herd**.

## Anatomy of the failure

```
Time 0: key "homepage" has 10k req/sec, all cache hits. DB sees 0 QPS.
Time T: TTL expires.
Time T+1ms: 10k requests in flight, all miss simultaneously.
            10k SQL queries hit the DB.
Time T+50ms: DB CPU saturates. Latency goes from 5ms to 5s.
Time T+200ms: requests time out. Retries fire. Stampede gets worse.
Time T+30s: ops paged. Manual mitigation.
```

The cruel irony: the **better** your cache normally is, the **worse** the stampede when it fails. Months of "100% cache hits" condition the DB to be sized for 1 QPS. Then 10k arrive at once.

## Why simple cache-aside has this bug

```
value = cache.get(key)
if value is nil:               # 10k goroutines reach this point
    value = db.read(key)       # 10k queries fire
    cache.set(key, value)      # 10k writes overwrite each other
```

There's no coordination across requests. Every miss thinks it's alone.

## Mitigations, from cheapest to most robust

### 1. Randomized TTL ("TTL jitter")

Instead of `TTL = 60` for all keys, use `TTL = 60 ± random(10)`. Keys expire spread out across time. Reduces *correlated* expiry — multiple keys won't expire at the same instant.

```python
ttl = base_ttl + random.uniform(-jitter, jitter)
cache.set(key, value, ttl)
```

Cheap, no coordination, no new infrastructure. **Always do this.**

Helps but doesn't fix the case where **one** key has 10k QPS — its single expiry still causes a stampede.

### 2. Probabilistic early expiration (XFetch)

Each reader, when reading a still-valid key, computes a probability of refreshing it early. The probability grows as the key nears expiry.

```python
def get(key):
    value, ttl_remaining, fetch_time = cache.get_with_meta(key)
    delta = fetch_time * BETA * log(random.random())  # negative
    if -delta > ttl_remaining:
        value = db.read(key)
        cache.set(key, value, ttl=fresh_ttl)
    return value
```

This spreads renewals over time so most refreshes happen *before* any user misses.

Used by Memcached "lease" semantics, popularized by an early Facebook paper.

### 3. Single-flight ("only one fetcher")

Per process or globally: when a miss happens, allow only **one** request to fetch from the DB. Others wait for the result.

Go's standard `golang.org/x/sync/singleflight`:

```go
var group singleflight.Group

func GetUser(id string) (User, error) {
    val, err, _ := group.Do(id, func() (interface{}, error) {
        return db.Read(id)   // executes once even if 1000 goroutines call simultaneously
    })
    return val.(User), err
}
```

Powerful — kills in-process stampede entirely. **Distributed** stampede (across 100 servers) needs a distributed lock.

### 4. Distributed lock on the miss path

When a cache miss happens, acquire a short-lived lock on the key in Redis. Only the lock holder fetches; others poll the cache or wait.

```python
def get(key):
    value = cache.get(key)
    if value is not None:
        return value
    lock = redis.set(f"lock:{key}", "1", nx=True, ex=5)
    if lock:
        value = db.read(key)
        cache.set(key, value, ttl)
        redis.delete(f"lock:{key}")
        return value
    else:
        # someone else is fetching; wait and re-read
        time.sleep(0.05)
        return cache.get(key) or db.read(key)
```

Robust across many app servers. Adds one Redis round-trip per miss.

### 5. Stale-while-revalidate

Serve **stale** data immediately, refresh in the background.

```python
def get(key):
    value, age = cache.get_with_age(key)
    if age > ttl:
        if age < ttl + grace:
            # stale but allowed; serve and refresh async
            async_refresh(key)
            return value
        else:
            # too stale; full refresh sync
            return refresh(key)
    return value
```

The user never blocks on a miss for a popular key — only the async refresher does. HTTP `Cache-Control: stale-while-revalidate=60` formalizes this for browsers and CDNs.

### 6. Pre-warm / write-through for known-hot keys

If you know "homepage data" is hot, write the cache **from the writer** (CMS publish, scheduled job) instead of relying on lazy population. Combine with refresh-ahead.

## A real-world story

Twitter's early "fail whale" had cache stampedes as a contributing pattern. Facebook published the *Memcached at Facebook* paper (2013) describing leases and other coordination primitives specifically built to suppress stampedes at their scale.

The pattern: **the first time you hit it, it'll look like a DB problem.** Look at cache hit rates over time, correlate with the DB spike. The hit rate drops in correlated bursts → stampede signature.

## Defense in depth

In production caches you usually want **all of**:
- TTL jitter (cheap, free win).
- Single-flight in process (Go: `singleflight`, PHP: lock around the cache miss).
- Stale-while-revalidate for genuinely hot keys.
- Distributed lock or probabilistic refresh for very hot keys.

## Architect's takeaway

- **The single-server fix is single-flight; the distributed fix is a lock or probabilistic refresh.**
- **Always TTL-jitter** — costs nothing.
- **For genuinely hot keys**, stale-while-revalidate gives the best user experience.
- **Test cache outages on purpose** — simulate full eviction in staging. The bug only shows under load.
- The first incident teaches every team this lesson. Better to learn it from a doc.
