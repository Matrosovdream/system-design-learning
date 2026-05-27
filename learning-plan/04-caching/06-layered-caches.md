# Example 06 — Layered caches: the cake between user and DB

In a real production system, a single request can be served by any of **five** caches before it reaches the database. Knowing the layers and what each is good at is essential.

## The five layers

```
[user browser]                          ← L0: HTTP cache
      ↓
[CDN]                                   ← L1: edge cache
      ↓
[reverse proxy / API gateway]           ← L2: gateway cache
      ↓
[app server, in-process LRU]            ← L3: in-process cache
      ↓
[Redis / Memcached cluster]             ← L4: distributed cache
      ↓
[DB buffer pool]                        ← L5: DB engine cache (built into Postgres/MySQL)
      ↓
[disk]
```

A request that hits L0 never leaves the user's machine. A request that hits L1 never leaves their continent. A request that reaches the DB has bypassed (or missed) every layer above.

## Each layer in detail

### L0 — Browser HTTP cache

Controlled by `Cache-Control`, `ETag`, `Last-Modified` headers. Lives in the user's browser. The cheapest cache: zero network.

- **Best for**: static assets, public API responses, anything safely versioned.
- **Pitfall**: invalidation is impossible — you control TTL via headers and live with whatever's on user devices for up to that TTL.

### L1 — CDN

Covered in step 02 example 04. Geographically distributed. Caches whatever the origin says is cacheable.

- **Best for**: static media, public API responses with TTL, images, video.
- **Pitfall**: cache key explosion if you vary on too many headers (e.g., `Cookie` → unique per user → uncacheable).

### L2 — Reverse proxy / API gateway

Nginx, Envoy, AWS ALB+CloudFront stack, or a dedicated gateway like Kong. Caches inside your perimeter.

- **Best for**: responses you want cached but not at the CDN (internal or auth'd responses).
- **Pitfall**: invalidation across many proxy nodes; usually solved by purge APIs or short TTLs.

### L3 — In-process LRU on each app server

A small (MBs to GBs) LRU inside the Go or PHP process. No network round-trip.

- **Best for**: small, very-hot reference data (feature flags, config, hot product details).
- **Pitfall**: each app server has its own copy → brief inconsistency. Need cache-busting on writes (pub/sub on config change, or short TTL).

### L4 — Distributed cache (Redis, Memcached)

The "the cache" most engineers mean when they say "cache". Shared across all app servers.

- **Best for**: user sessions, per-user data, denormalized objects, anything app servers must agree on.
- **Pitfall**: network round-trip (~0.5-1 ms in same DC). Cluster failures degrade performance.

### L5 — DB buffer pool

Postgres's shared buffers, MySQL's InnoDB buffer pool, MongoDB's WiredTiger cache. The DB caches recently-touched pages in RAM so subsequent reads avoid disk.

- **Best for**: hot rows you're reading at the DB level anyway.
- **Pitfall**: a single huge analytical query can flush it (called "buffer pool pollution"). Tune `effective_cache_size`, monitor `cache hit ratio`.

## Hit rates compound

If each layer hits 70% of requests:

| Layer hit | Cumulative hits | Requests reaching the next layer |
|-----------|-----------------|----------------------------------|
| L0 70%    | 70%             | 30%                              |
| L1 70%    | 91%             | 9%                               |
| L2 70%    | 97.3%           | 2.7%                             |
| L3 70%    | 99.2%           | 0.8%                             |
| L4 70%    | 99.76%          | 0.24%                            |

**0.24% of original traffic reaches the DB.** That's the engineering magic of layered caching: very modest hit rates per layer compound into vanishingly small backend load.

## What goes where — design heuristics

| Data                          | Layer that owns it                         |
|-------------------------------|--------------------------------------------|
| App JS bundle, CSS, images    | L0 (browser) + L1 (CDN). Versioned URLs.   |
| Public API response (TTL OK)  | L1 (CDN) + L2 (gateway).                   |
| User-specific data            | L2 (per-user keyed) or L4 (Redis).         |
| Session                       | L4 (Redis with TTL).                       |
| Feature flags / app config    | L3 (in-process, refresh every 30s).        |
| Hot product page              | L3 (top-N) + L4 + L1 if public.            |
| User feed (per-user)          | L4 only (CDN can't help — user-specific).  |
| Computed leaderboard          | L4 (Redis sorted set) + L1 if public.      |
| Generic DB read               | L4 (cache-aside) + L5 buffer pool.         |

## Invalidation: the chain reaction

When the canonical data changes, **every layer holding a stale copy must learn**. Strategies:

1. **Push** — write path actively invalidates: `cache.delete(key)`, `cdn.purge(url)`, message-bus broadcast to in-process caches.
2. **Pull** — short TTLs; everything refreshes itself periodically.
3. **Versioned keys / URLs** — the key changes on write, old keys naturally expire.

For multi-layer invalidation, **versioning** is the simplest correct strategy:
- New asset → new URL (`app.5f3a91.js` → `app.0bd982.js`).
- Updated user → bump `user:123:v` from `7` to `8`; downstream keys become `user:123:v8:...`.

This sidesteps every coordination problem at the cost of slightly more storage.

## Common pitfalls in layered caching

- **Forgetting to invalidate L3** (in-process caches). Devs remember Redis, forget the LRU they added.
- **CDN caching auth'd responses** because `Vary: Cookie` was set incorrectly — leaking data between users.
- **Stampeding the lowest layer** when an upper layer fails. Always have rate limits at the DB.
- **No observability per layer** — you can't tune what you can't see. Track hit rate per layer.

## Architect's takeaway

- **Caches are a layered system, not a single component.** Each layer has different invalidation properties, different latencies, different costs.
- **Push as much as possible to the highest-numbered (closest-to-user) layer.** CDN > Redis > DB.
- **Versioned keys / URLs** are the architect's friend for invalidation correctness.
- **Measure hit rate per layer** in production. Without it, you're tuning blind.
- The 99%+ cumulative hit rates you see at scale come from compounding modest per-layer rates, not from one magic cache being perfect.
