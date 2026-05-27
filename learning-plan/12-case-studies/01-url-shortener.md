# Case Study 01 — Design a URL shortener (bit.ly, TinyURL)

The classic warm-up question. Simple-sounding, but probes how you think about reads, writes, and uniqueness at scale.

## Problem statement

Build a service that takes a long URL and returns a short URL like `https://sho.rt/Xa8Bp2`. Visiting the short URL redirects to the long one.

## Clarifying questions (always ask first)

1. **Scale**: How many URLs per day? How many redirects?
2. **Read vs write ratio**: redirects vs URL creations.
3. **URL lifetime**: do URLs expire? How long?
4. **Custom aliases**: can users specify a custom short code?
5. **Analytics**: do we track clicks?
6. **Geography**: global users, or one region?
7. **API or UI**: who calls this? (Affects rate limiting, auth.)

**Assumed answers for this walkthrough:**

- 100M new URLs per day, 10B redirects per day (100:1 read/write).
- 5-year retention; older URLs may expire.
- Custom aliases supported.
- Basic click analytics (total + last 24h).
- Global with low latency.
- API for partners; public web UI.

## Functional requirements

- Create short URL from a long URL (with optional custom alias).
- Redirect short → long URL.
- Expire URLs after configured TTL.
- Analytics: click count per URL.

## Non-functional requirements

- Read latency p95 < 100 ms globally.
- Write latency p95 < 500 ms.
- Availability: 99.99%.
- Durability: never lose a URL.
- Strong: no collisions (one short code → one long URL).

## Capacity estimation

```
Writes per second:
  100M / 86400 ≈ 1,160/sec average
  Peak (3x): ~3,500 writes/sec

Reads per second:
  10B / 86400 ≈ 116,000/sec average
  Peak: ~350,000/sec  ← need aggressive caching

Storage per year:
  100M URLs/day × 365 = ~36B URLs/year
  Per URL: ~500 bytes (short code, long URL, owner, timestamp, click count)
  36B × 500 bytes = ~18 TB/year

Bandwidth:
  Writes: 3,500 × 500B = 1.75 MB/sec
  Reads: 350k × 300B (HTTP 301 response) = 100 MB/sec  ← CDN it
```

## API design

```
POST /api/v1/shorten
  Body: { long_url, custom_alias?, ttl_days? }
  Response: 201 { short_url, expires_at }

GET /{short_code}
  Response: 301 Moved Permanently
            Location: <long_url>

GET /api/v1/urls/{short_code}/stats
  Response: 200 { clicks, created_at, expires_at }

DELETE /api/v1/urls/{short_code}    (owner only)
  Response: 204
```

## High-level architecture

```
                        ┌─ [CDN cache for /{short_code}]
                        │     (cache 301 responses for popular URLs)
[user] → [LB] → [API gateway] ─┐
                        │       │
                        │       ▼
                        ▼       [URL service]
                  [Redis: hot cache]
                        ↑
                        │
                    [Postgres or Cassandra: URL store]
                        │
                        └─ [Analytics: Kafka → ClickHouse]
                              (async click counting)
```

## Data model

### Primary store: a sharded KV store (DynamoDB / Cassandra) or sharded Postgres

```
urls
  short_code (PK, varchar 7-10)
  long_url (text)
  owner_id (varchar)
  created_at (timestamp)
  expires_at (timestamp)
  click_count_cached (bigint)
```

Why KV-style? Lookups are by primary key (short_code). No joins. Naturally shardable.

Sharding key: hash(short_code) for uniform distribution.

### Analytics store: separate

Click events streamed to Kafka, aggregated in ClickHouse or similar columnar store. The `urls` table only holds approximate aggregate counts updated every few minutes.

## Deep dive: generating short codes

The crux of the design. Two approaches.

### Approach A: Hash the long URL

```
short_code = base62(murmur3(long_url))[:7]
```

7 characters of base62 = 62^7 ≈ 3.5 trillion combinations. Plenty.

**Pros**: deterministic — same long URL always produces the same short code. Idempotent inserts.

**Cons**: collisions are possible (different URLs hashing to the same code). Need to detect and resolve.

```
on create:
  code = hash(long_url)[:7]
  if exists(code):
      if existing_long_url == long_url:
          return existing_code  # already shortened
      else:
          code = hash(long_url + salt)[:7]  # rehash with salt
          ... retry up to N times
  insert(code, long_url)
```

Collision rate at 36B URLs in a 3.5T space: ~1% by birthday paradox. Manageable, but adds complexity.

### Approach B: Counter-based (sequential IDs)

```
id = global_counter++  (Snowflake-style, or Zookeeper, or one DB sequence)
short_code = base62(id).pad(7)
```

**Pros**: zero collisions by construction. Sequential codes (not random) — slight predictability concern for spam/abuse.

**Cons**: need a distributed counter or a centralized one. Snowflake (~64-bit IDs in 7 chars of base62) gives you ~1 trillion IDs — fine. Counter coordination across regions is harder.

### Approach C: Random with collision check (most common in practice)

```
on create:
  loop:
    code = random_base62(7)
    inserted = try_insert(code, long_url) -- atomic
    if inserted: return code
    -- else: collision, retry
```

Simple. Collisions are rare at this length. The atomic insert (unique constraint) prevents races.

**Recommendation for the interview**: random with atomic insert. Mention the alternatives, justify the choice (simplicity + no coordination needed).

## Deep dive: scaling reads

10B reads/day = 350k QPS peak. Can't serve all from Postgres.

### Layer 1: CDN

Most popular URLs (e.g., the top 1%) get billions of clicks. Cache the 301 response at the CDN edge.

```
GET /{short_code}
  → CDN cache (TTL 5 min, with stale-while-revalidate)
  → cache hit: serve 301 immediately
  → cache miss: forward to origin
```

This absorbs ~80%+ of traffic at the edge.

### Layer 2: Redis cluster

Behind CDN. Cache the lookup `short_code → long_url`.

```
key: "url:Xa8Bp2"
value: "https://example.com/long/url"
TTL: 1 hour
```

Cache-aside pattern. Cache penetration defense: also cache "not found" for shorter TTL.

### Layer 3: DB

Sharded KV store or Postgres. Reads should be rare here (most absorbed above).

### Click counting (decoupled)

Counting clicks on every GET would double-write per redirect. Instead:

- Each app server batches click events in memory.
- Flushes to Kafka every few seconds.
- A consumer aggregates → updates `click_count_cached` in DB every minute.
- Detailed analytics → ClickHouse for queries like "clicks per hour over last 30 days".

This decouples user-facing read latency from analytics overhead.

## Trade-offs discussion

### Why not just use Postgres for everything?

For URLs: fine, could use Postgres with sharding. Slightly more flexible (queries on owner, analytics). The "Cassandra/DynamoDB" choice is for ease-of-scale (auto-sharding, multi-region replication built in).

### Why 7-character codes?

Sweet spot: short enough to be useful (typing/sharing), long enough that random collisions are rare for years of growth.

### Why 301 (not 302)?

301 is permanent. Browsers/CDNs cache 301 strongly. Result: future redirects to that URL don't even hit our service. For analytics, this means we **undercount** repeat clicks from same user — a known trade-off. Some shorteners use 302 for accurate analytics at the cost of reduced caching.

### Read-your-writes

After a user creates a short URL, they should be able to use it immediately. The write path (Postgres) is the source of truth; we serve reads from it briefly until cache populates. Or: write-through Redis on creation.

### Custom aliases

Add an atomic insert with `INSERT ... ON CONFLICT DO NOTHING`. If conflict, return error. Custom aliases share the same namespace as auto-generated.

## Failure modes

- **DB primary down**: failover to standby; some writes might be lost depending on replication. Reads continue from cache + replicas.
- **Redis down**: more load on DB. Capacity planning should account for this — DB sized to handle peak with no cache.
- **CDN node down**: traffic shifts to other CDN nodes; origin sees a brief spike, then stabilizes.
- **Hot URL** (one URL going viral, 100k QPS): CDN absorbs most; Redis with replicas handles the rest. Origin barely sees it.

## Common follow-up questions

1. **"How do you ensure short codes can't be guessed?"**
   Random codes from a 62^7 space — not enumerable. For sensitive URLs, lengthen the code or add user-bound tokens.

2. **"How do you delete expired URLs?"**
   Background job scans for `expires_at < now()`. Deletes from DB; tombstones the cache entry; emits an event so analytics removes data too.

3. **"How would you build the analytics dashboard?"**
   Click events → Kafka → ClickHouse (or Druid / Pinot). Pre-aggregated rolling counts. Frontend reads from ClickHouse.

4. **"What about abuse / phishing URLs?"**
   - Domain blocklist check on creation.
   - Periodic scan of URLs against safe-browsing APIs.
   - Per-user rate limit on URL creation.
   - Captcha for anonymous users.

5. **"How do you operate this across regions?"**
   - DB: regional active-passive with async replication, or use Spanner/Cosmos for global ACID at higher cost.
   - Cache and CDN: globally distributed already.
   - Writes: usually pinned to a primary region; reads served regionally.

6. **"What if 1 URL receives 1 million QPS?"** (Celebrity URL)
   - CDN absorbs at the edge.
   - L1 in-process cache on every app server.
   - Replicate the hot key across Redis shards (`url:Xa8Bp2:r0`, `url:Xa8Bp2:r1`, ...).
   - This is exactly the hot-key problem from step 04 example 04.

## Key takeaways

- **It's mostly a read problem.** Cache everywhere; the DB is the boring bit.
- **Short code generation** is the design crux. Random + atomic insert is simplest.
- **Decouple analytics** from the redirect path with Kafka + warehouse.
- **CDN + Redis + DB** is the standard three-layer architecture.
- **Even simple-looking problems have real depth** — that's why URL shortener is the warm-up: it lets the interviewer see the full breadth of your thinking quickly.
