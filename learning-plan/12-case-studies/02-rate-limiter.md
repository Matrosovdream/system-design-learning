# Case Study 02 — Design a distributed rate limiter

A service that decides "should I allow this request?" based on per-user / per-IP / per-key limits. Critical infrastructure for any large API.

## Problem statement

Build a rate limiter that:
- Limits requests per user (or API key, or IP) over a time window.
- Operates across many app servers (distributed).
- Has sub-millisecond decision latency.
- Returns to the client whether their request is allowed.

## Clarifying questions

1. **What gets rate-limited?** Per user, per IP, per API key, per endpoint? Combinations?
2. **What windows?** Per second, per minute, per hour, per day? Multiple at once?
3. **Hard or soft limits?** Reject excess or just delay?
4. **Accuracy tolerance?** Exact or approximate (e.g., off by ±5%)?
5. **What scale?** QPS at peak.
6. **Multi-region?** Does the limit apply globally or per region?
7. **Failure mode**: if rate limiter is down, fail open (allow) or fail closed (deny)?

**Assumed answers:**

- Limit by (user_id, endpoint). Different limits per endpoint.
- Multiple windows: 100/min, 5000/hour, 50000/day per endpoint.
- Hard reject with `429 Too Many Requests`.
- Approximation acceptable (±2%).
- 1M req/s peak.
- Global limits (cross-region).
- Fail open if rate limiter is down (better availability than perfect enforcement).

## Functional requirements

- Decide allow/deny per request based on configurable limits.
- Support per-user, per-IP, per-API-key limits, per-endpoint.
- Multiple windows (minute, hour, day).
- Update limits via config without restart.

## Non-functional requirements

- Decision latency < 5 ms p99.
- High availability (rate limiter outage shouldn't take down API).
- Low overhead: <10% of resources on the request path.
- Scale to 1M decisions/sec.

## Capacity estimation

```
1M decisions/sec × seconds in a day = ~86B decisions/day.
Distinct keys: ~10M users × 10 endpoints = 100M (key, endpoint) pairs.
State per key: ~50 bytes (tokens, last refill timestamp, counters per window).
Total state: 100M × 50 bytes = 5 GB. Fits in Redis comfortably.

Network: each decision ~1 Redis round-trip; 1M req/s = 1M Redis ops/s.
   → need a Redis cluster sized for this (e.g., 10-20 nodes, 100k ops/s each).
```

## API design

There are two API shapes:

### Internal: called by the gateway/app before each request

```
isAllowed(key, endpoint) → { allowed: bool, remaining: int, reset_at: timestamp }
```

This is the most common shape. The gateway/app calls the rate limiter inline.

### External: rare, only if the rate limiter is exposed (uncommon)

Generally, rate limiting is an internal concern.

## High-level architecture

```
[client]
   ▼
[API gateway / app]
   │
   ├─► [Rate limiter service]   (or inline library + Redis)
   │       │
   │       ▼
   │   [Redis cluster: token buckets / counters]
   │
   ▼ (if allowed)
[downstream services]
```

Two common deployment shapes:

### Shape A: As a sidecar / library (lowest latency)

Each app process embeds a rate-limiter library. The library talks directly to Redis. No network hop to a separate rate-limiter service.

Pros: lowest latency.
Cons: every app implements the lib (consistency burden).

### Shape B: Standalone rate-limiter service

A dedicated service (gRPC). Apps call `RateLimiter.Check(...)`.

Pros: centralized logic, easy to update, consistent across apps.
Cons: extra network hop.

In practice, the **gateway-level rate limiter** (Kong, Envoy, AWS API Gateway) does this in shape B; in-app fine-grained limits use shape A.

## Deep dive: the algorithm

Token bucket (covered in step 10 example 05). Per (user_id, endpoint) pair, maintain:

```
{
  tokens: float,
  last_refill: timestamp
}
```

Each request:
1. Compute elapsed time since `last_refill`.
2. Refill tokens at rate `refill_rate` up to `capacity`.
3. If `tokens >= 1`, allow, decrement.
4. Else, reject.

In Redis, this must be atomic (multiple processes incrementing concurrently). Use a Lua script (executed atomically inside Redis):

```lua
-- KEYS[1] = "rl:user:42:endpoint:/api/orders"
-- ARGV[1] = capacity
-- ARGV[2] = refill_per_sec
-- ARGV[3] = now (unix ms)

local data = redis.call('HMGET', KEYS[1], 'tokens', 'last')
local tokens = tonumber(data[1]) or tonumber(ARGV[1])
local last = tonumber(data[2]) or tonumber(ARGV[3])

local elapsed = (tonumber(ARGV[3]) - last) / 1000.0
tokens = math.min(tonumber(ARGV[1]), tokens + elapsed * tonumber(ARGV[2]))

if tokens >= 1 then
    tokens = tokens - 1
    redis.call('HMSET', KEYS[1], 'tokens', tokens, 'last', ARGV[3])
    redis.call('EXPIRE', KEYS[1], 3600)
    return {1, tokens}
else
    redis.call('HMSET', KEYS[1], 'tokens', tokens, 'last', ARGV[3])
    return {0, tokens}
end
```

The Lua script runs atomically in Redis. One round-trip per decision. ~1 ms latency in the same DC.

## Deep dive: multiple windows

Each (user, endpoint) might have several limits: 100/min, 5000/hour, 50000/day.

Each is a separate bucket. The Lua script checks all relevant buckets; if any rejects, the request is denied.

```
limit check:
  bucket_minute = check_bucket("rl:user:42:ep:/api/orders:min", 100, 100/60)
  bucket_hour = check_bucket("rl:user:42:ep:/api/orders:hr", 5000, 5000/3600)
  bucket_day = check_bucket("rl:user:42:ep:/api/orders:day", 50000, 50000/86400)
  if any rejected: deny
  else: consume from all, allow
```

Three Redis ops per request. Acceptable, but for very high-QPS endpoints, you might pre-compute the bottleneck window and check only that one first.

## Deep dive: distributed across regions

If you have 3 regions and a global limit of 100/min per user, each region can't independently allow 100/min — total would be 300.

Options:

### Approach 1: Centralized Redis

One Redis cluster in one region. All regions consult it. Latency-prohibitive for cross-region.

### Approach 2: Per-region buckets with sub-allocation

Each region gets 1/N of the limit. User in region A → check region A's local Redis. Local; fast; but if traffic is uneven across regions, some users hit limits in busy regions while others' quota goes unused.

### Approach 3: Approximate global counter (CRDT-style)

Each region has a local counter. Periodically (every few seconds), regions exchange counts. Total is approximate but globally consistent.

For most APIs: **Approach 2** is sufficient. True global-precise limiting is rare (mostly anti-fraud cases) and expensive.

## Failure mode design

If Redis is unreachable, what does the rate limiter do?

### Fail open (allow everything)

Pros: API stays available.
Cons: limits are not enforced; abuse can briefly happen during incidents.

### Fail closed (deny everything)

Pros: limits stay enforced.
Cons: rate limiter outage = full API outage. Bad.

**Standard recommendation**: fail open for general rate limiting. Fail closed for security-critical limiters (e.g., the login endpoint — better to reject some legitimate users than allow a credential-stuffing attack).

In-app: cache last decision per (user, endpoint) briefly so if Redis is down for 5 seconds, the app uses stale data instead of failing.

## Client communication

When rejected, respond clearly:

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1716123456
Retry-After: 47
Content-Type: application/json

{ "error": "rate_limit_exceeded", "retry_after_seconds": 47 }
```

Clients can:
- Honor `Retry-After`.
- Display "please wait" to users.
- Switch to exponential backoff.

## Trade-offs discussion

### Why Redis?

In-memory + atomic Lua scripting + cluster mode + low latency. The de facto standard for distributed rate limiting.

Alternatives:
- **In-process counters**: fast but not distributed; would allow per-process limits only.
- **DB-backed**: too slow.
- **Specialized systems** (Stripe's Doorman, Envoy's RLS): viable for very specific needs.

### Approximate vs exact

Token bucket as implemented above is approximate due to refill timing and (with multi-bucket) check sequencing. For 99.9% of use cases, ±1-2% is fine. For "exactly 100 requests/min, never 101", you need stricter (sliding-window log; more storage; more CPU).

### Per-user vs per-IP

Per-user is more accurate (one IP can have many users behind NAT; one user can switch IPs). Use per-user when you have user identity; per-IP as a fallback for unauthenticated traffic.

### Synchronous gateway check vs async background

Gateway-level rate limiting is synchronous (the gateway must decide before forwarding). Some advanced systems use async approval (allow first request, asynchronously update state, reject only at very obvious limits) — gains tiny latency at the cost of more lax enforcement.

## Common follow-up questions

1. **"What if 1 user generates 1M req/s?"**
   That's a hot key in Redis. Same problem as step 04 example 04. Replicate the key, or use in-process limiter for very high-volume users.

2. **"Can you build this without Redis?"**
   Per-process counters synchronized by gossip; in-memory with eventual consistency. Possible but more complex than Redis. Used when latency requires sub-ms decisions.

3. **"How do you change limits without restarting?"**
   Limits in config (e.g., etcd, Consul KV, DB table). Rate limiter watches the config; updates in-place. The actual buckets in Redis don't need to change — only the `capacity` and `refill_rate` parameters used.

4. **"How do you give some users higher limits?"**
   Look up the user's tier (free, paid, enterprise) from a fast cache (Redis or local). Use tier-specific limits.

5. **"What about burst tolerance?"**
   Token bucket already supports bursts up to bucket capacity. To allow more burst: increase capacity (but keep refill rate). E.g., capacity 200, refill 100/min → can burst 200 then settles to 100/min.

6. **"How do you test this without affecting production traffic?"**
   Shadow mode: rate limiter computes the decision but doesn't enforce; logs what *would* have been rejected. Compare against current production to tune thresholds.

7. **"What about the rate limiter's own throughput?"**
   Redis cluster scales horizontally. 1M req/s with 10ish nodes is well-trodden. Use Redis Cluster, partition by `(user_id)`.

## Key takeaways

- **Token bucket is the standard algorithm.** Simple, allows bursts, scales.
- **Redis + Lua script** is the standard implementation. Atomic, fast.
- **Layer rate limits**: gateway / app / downstream. Each catches different abuse.
- **Fail open for general limits**, fail closed for security-sensitive ones.
- **Communicate clearly** to clients: 429 + Retry-After + headers.
- **Approximate is usually fine** — chasing exactness costs a lot for diminishing returns.
