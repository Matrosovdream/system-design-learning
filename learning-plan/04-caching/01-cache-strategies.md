# Example 01 — Cache strategies: aside, through, behind, ahead

How does data get into the cache, and what happens on a write? Five named patterns cover almost everything.

## 1. Cache-aside (lazy loading) — the default

App talks to **both** the cache and the DB. Cache is treated as an optional fast layer.

```
READ:
  value = cache.get(key)
  if value is nil:
      value = db.read(key)
      cache.set(key, value, ttl)
  return value

WRITE:
  db.write(key, value)
  cache.delete(key)   # invalidate; next read repopulates
```

**Pros**
- Cache failures don't break the app — DB is still the source of truth.
- Only what's actually read gets cached → hot working set, not whole DB.
- Simple to implement.

**Cons**
- First read of a key is slow (cache miss → DB read).
- Brief race: read, miss, db.read, **between this and cache.set, another write commits new value** — old value is now cached. Mitigate with short TTLs or explicit invalidation on write.

**This is the default for 90% of caching use cases.** Redis + Postgres apps almost always do this.

## 2. Read-through

Cache **owns** the read path. The app talks only to the cache; the cache loads from the DB on miss.

```
app:    cache.get(key)
cache:  if not present, call DB loader, store, return
```

**Pros**
- App code is cleaner — only one API.
- Cache library can deduplicate concurrent misses (one DB load per key, not 100 — solves stampede).

**Cons**
- Cache outage = total outage (unless app has a fallback path).
- Tight coupling between cache and DB schema.

**Used in**: Caffeine (Java), Guava cache, some Go LRU libraries with `LoadingCache` semantics. Less common in distributed Redis setups.

## 3. Write-through

Every write goes to the cache **and** the DB synchronously, in the same critical section.

```
WRITE:
  cache.set(key, value)
  db.write(key, value)
  ack to caller
```

**Pros**
- Cache is always consistent with DB.
- Subsequent reads are warm.

**Cons**
- Every write pays for two stores → higher write latency.
- Pointless if the value is unlikely to be re-read soon ("write but never read" wastes cache space).

**Use when**: writes are followed by immediate reads (e.g., user profile updated on a settings page, then re-displayed).

## 4. Write-back (write-behind)

Writes go to the cache **only**. The cache flushes to the DB asynchronously, in batches.

```
WRITE:
  cache.set(key, value)   ← ack here, super fast
  ...later...
  background worker batches dirty keys → db.bulk_write
```

**Pros**
- Very fast writes (cache speed).
- Batches reduce DB load by 10-100×.

**Cons**
- **Risk of data loss** if cache crashes before flushing.
- Complexity: dirty tracking, replay on restart, ordering guarantees.

**Use when**: high write volume, some loss is acceptable (analytics counters, view tracking). **Never for money, orders, identity.**

Often combined with append-only logs (Kafka, AOF) for durability.

## 5. Refresh-ahead

Before a popular entry expires, the cache **pre-fetches** a new value so the user never sees a miss.

```
on access:
  if ttl_remaining < threshold:
      asyncly refresh from DB
  return cached value (still valid)
```

**Pros**
- Eliminates miss latency for hot keys.
- Smooths DB load (no sudden spike at expiry).

**Cons**
- Wasted refreshes for items that won't be re-read.
- Only effective if you can predict popularity.

**Use when**: hot popular data with steady traffic (homepage, top-N lists, leaderboards).

## Which one to use — the cheat sheet

| Workload                              | Strategy                       |
|---------------------------------------|--------------------------------|
| General app data (users, products)    | **Cache-aside** (default)      |
| Per-request data, predictable shape   | **Read-through** if your library supports it |
| Profile updates re-read immediately   | **Write-through**              |
| Counters, telemetry, analytics events | **Write-back** (with WAL/Kafka)|
| Hot homepage / "top N" data           | **Refresh-ahead** + cache-aside|

## A real example: e-commerce product page

```
User loads /product/12345:

App:
  product = cache.get("product:12345")
  if product is nil:
    product = db.query("SELECT ... WHERE id=12345")
    cache.set("product:12345", product, ttl=600)
  return product
```

That's cache-aside.

Now, an admin edits the price:

```
App:
  db.update("UPDATE products SET price=... WHERE id=12345")
  cache.delete("product:12345")
```

Next user read repopulates the cache. Simple, correct, fast.

Now add a *featured product* on the homepage — it gets 10k req/sec. At TTL expiry, all 10k requests miss simultaneously → DB swamped. This is **cache stampede**, covered in example 03.

## Architect's takeaway

- **Cache-aside is the default.** Reach for others only when you have a specific reason.
- **Write-through is rarely worth it** unless you know the write will be read immediately.
- **Write-back without a durable log is dangerous** — it has lost data at every company that has used it naively.
- **Combine strategies per data type**: cache-aside for products, write-back (durable) for counters, refresh-ahead for hot homepage data.
