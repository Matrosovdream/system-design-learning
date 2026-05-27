# Example 05 — Rate limiting: token bucket, leaky bucket, sliding window

Rate limiting protects your services from overload (intentional or accidental). Done badly, it punishes legitimate users; done well, it's invisible to most and brutal to abusers.

## What you're rate-limiting

- **Per-user** — prevent abuse from one account.
- **Per-IP** — defend against bots and DDoS (but mobile carriers share IPs).
- **Per-API key** — for partner integrations with tiered plans.
- **Global** — protect a downstream from any source overwhelming it.
- **Per-endpoint** — `/login` is more sensitive than `/products`.

A real API often applies several at once.

## The four common algorithms

### 1. Fixed window

Counter resets every N seconds.

```
window: 1 minute
limit: 100 requests
counter: increment on each request
on reset (every minute): counter = 0
if counter > limit: reject
```

**Pros**: simplest, trivial in Redis (`INCR` + `EXPIRE`).

**Cons**: **edge effect.** If limit is 100/minute and 100 requests arrive at 12:00:59, then another 100 at 12:01:00, that's 200 requests in 2 seconds — well under "rate limit" but a real burst.

### 2. Sliding window (log)

Store timestamps of every request. On a new request, drop entries older than the window, count the remainder, allow if under limit.

```python
def allow(user_id):
    now = time.now()
    window_start = now - 60
    redis.zremrangebyscore(f"rate:{user_id}", 0, window_start)
    count = redis.zcard(f"rate:{user_id}")
    if count >= limit:
        return False
    redis.zadd(f"rate:{user_id}", {now: now})
    redis.expire(f"rate:{user_id}", 60)
    return True
```

**Pros**: precise — no edge effects.

**Cons**: stores one record per request. Expensive for high-rate users.

### 3. Sliding window (counter)

Approximation of the log: weight current and previous fixed windows.

```
last_minute_count = redis.get("rate:user:12:00")  # might be 80
current_minute_count = redis.get("rate:user:12:01")  # 30, but we're 30 seconds into the minute

estimated = last_minute_count * (1 - 30/60) + current_minute_count
          = 80 * 0.5 + 30 = 70
```

**Pros**: O(1) state. Smooth — no edge effects.

**Cons**: approximate (not 100% accurate when rate is bursty).

Used by Cloudflare's rate limiter and many production systems.

### 4. Token bucket

Imagine a bucket that fills with tokens at a fixed rate. Each request consumes one token. If the bucket is empty, request is rejected.

```
bucket: capacity 100, refill rate 10/sec

at any time:
  tokens = min(capacity, last_tokens + (now - last_time) * refill_rate)

on request:
  refill (above)
  if tokens >= 1:
      tokens -= 1
      allow
  else:
      reject
```

**Pros**:
- **Allows bursts** up to bucket capacity.
- **Smooth steady-state** rate via refill.
- O(1) state.

**Cons**: slightly more state to track per user (tokens + timestamp).

Used by AWS, GCP, Stripe, GitHub. The most common production choice.

### 5. Leaky bucket

Similar to token bucket but inverted: requests enter a queue, drain at fixed rate. If the queue is full, new requests are rejected.

**Pros**: smooths spikes into a steady output rate.

**Cons**: enforces strict steady-rate (no bursts).

Use when you need to **protect a downstream** that has a hard rate limit (database, third-party API), not just when you want to limit a caller.

## Where to apply rate limits — layered

### Layer 1: CDN / WAF

Global, per-IP, very simple. Defends against DDoS, basic abuse. Cloudflare, AWS WAF, Fastly all support this.

### Layer 2: API gateway

Per-IP, per-API-key, per-endpoint. Configured by ops, no app code change.

```yaml
# Kong rate-limit plugin config
limits:
  /api/login: 5/minute per ip
  /api/*:    1000/hour per api-key
```

### Layer 3: Application

Per-user, per-action, business-aware. E.g., "max 10 password resets per email per day".

These can't live at the gateway because they require knowing the user / business state.

### Layer 4: Downstream rate-limit awareness

When calling Stripe (which rate-limits you), implement client-side rate limiting so you don't get throttled. Use a leaky bucket internally.

## Implementation in Redis

Simple token bucket with Lua atomicity:

```lua
-- KEYS[1] = bucket key
-- ARGV[1] = capacity
-- ARGV[2] = refill_rate (per second)
-- ARGV[3] = current timestamp (seconds)
-- ARGV[4] = tokens requested (usually 1)

local data = redis.call('HMGET', KEYS[1], 'tokens', 'last_refill')
local tokens = tonumber(data[1]) or tonumber(ARGV[1])
local last = tonumber(data[2]) or tonumber(ARGV[3])

local elapsed = tonumber(ARGV[3]) - last
tokens = math.min(tonumber(ARGV[1]), tokens + elapsed * tonumber(ARGV[2]))

local requested = tonumber(ARGV[4])
if tokens >= requested then
    tokens = tokens - requested
    redis.call('HMSET', KEYS[1], 'tokens', tokens, 'last_refill', ARGV[3])
    redis.call('EXPIRE', KEYS[1], 3600)
    return 1
else
    redis.call('HMSET', KEYS[1], 'tokens', tokens, 'last_refill', ARGV[3])
    return 0
end
```

Atomic, fast, scales to millions of users.

## Telling the client clearly

Good rate-limited APIs include headers:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 873
X-RateLimit-Reset: 1716123456    ← unix timestamp
```

On rejection:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60                  ← seconds to wait
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1716123456
Content-Type: application/json

{ "error": "rate_limit_exceeded", "retry_after_seconds": 60 }
```

This lets clients back off intelligently (vs guessing or fast-retrying).

## Special endpoints to rate-limit hardest

- **/login, /signup** — prevent credential stuffing, account creation abuse.
- **/password-reset** — abuse → email bombing.
- **/api/expensive-search** — DB-killing endpoints.
- **/api/upload** — file upload bandwidth abuse.
- **Authenticated APIs in general** — per-user, more generous; per-IP, stricter.

## Common mistakes

### One global rate limit only

You limit "1000 RPS to API". One bad user hits 999; everyone else can only do 1 RPS. Always rate limit **per identity** (user, key, IP), not just globally.

### Identifying users only by IP

Mobile carriers NAT thousands of users behind one IP. A strict per-IP limit blocks the whole carrier. Use per-user limits when you have user identity; per-IP only as a fallback or for unauthenticated traffic.

### Not telling the client

You silently drop or 500 over-limit requests. Clients don't know it's a rate limit; they retry harder; you make it worse. Return **429** with **Retry-After**.

### Too low limits

You discover the legitimate user pattern peaks at 50 RPS. You set the limit to 100 to be safe. Marketing campaign drives 5× normal traffic; legitimate users hit the limit. Set limits with measurement-driven generosity.

### No exemption for system traffic

Your own health-check or monitoring traffic counts against the rate limit. Disabling probes won't help, and you can't tell when a service is broken because all you see is 429.

## Architect's takeaway

- **Layer rate limits**: CDN / gateway / app / downstream. Each catches a different abuse pattern.
- **Token bucket is the default algorithm** — allows bursts, smooth steady state.
- **Identify the right key**: user, API key, IP (in fallback). Avoid IP-only.
- **Tell the client**: 429 + Retry-After + headers. Don't be silent.
- **Sensitive endpoints get stricter limits** — login, password reset, expensive queries.
- **Measure traffic before setting limits**, not the other way around.
