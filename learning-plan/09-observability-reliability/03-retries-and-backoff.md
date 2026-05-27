# Example 03 — Retries and backoff: done right, done badly

Retries fix transient failures. Done badly, they **cause** failures. Every reliable distributed system thinks carefully about its retry policy.

## When to retry

- **Network timeout** — packet might have been lost.
- **5xx response** from downstream — often transient.
- **429 Too Many Requests** — back off and retry.
- **Specific known-retryable errors** — e.g., `ECONNRESET`, "deadlock detected" in DB.

## When NOT to retry

- **4xx (except 429)** — the request was bad; retrying doesn't help.
- **Application-level "no" errors** — "card declined", "user not found". Retrying makes no difference.
- **Non-idempotent operations** when you don't know if the original succeeded.

## The naive retry, and why it kills you

```python
for attempt in range(5):
    try:
        return call_downstream()
    except:
        continue
raise
```

What's wrong:
1. **No delay between attempts.** Hammers the downstream.
2. **No backoff.** If the downstream is slow because it's overloaded, 5 immediate retries make it worse.
3. **No jitter.** If 100 callers retry at the same time, they all retry at the same time.
4. **No bound on total time.** A request can wait 30 seconds for retries.
5. **Indiscriminate retries.** A 401 Unauthorized gets retried 5 times — pointless.

## The right shape

```python
def call_with_retries(fn, *, max_attempts=4, base_ms=100, cap_ms=5000, max_total_ms=10000):
    deadline = now_ms() + max_total_ms
    attempt = 0
    while True:
        try:
            return fn()
        except RetryableError as e:
            attempt += 1
            if attempt >= max_attempts:
                raise
            # exponential backoff with full jitter
            delay = random.uniform(0, min(cap_ms, base_ms * (2 ** attempt)))
            if now_ms() + delay > deadline:
                raise
            sleep_ms(delay)
        except NonRetryableError:
            raise
```

Key elements:

### 1. Exponential backoff

Each retry waits **longer** than the last. Common: double each time.

```
attempt 1: wait 100ms
attempt 2: wait 200ms
attempt 3: wait 400ms
attempt 4: wait 800ms
attempt 5: wait 1600ms
```

This gives the downstream time to recover. Aggressive immediate retries make a struggling system struggle more.

### 2. Jitter

If 1000 clients all hit a 503 at the same moment and all retry "in 200ms", they retry **at the same moment 200ms later**. The thundering-herd repeats.

**Full jitter**: pick a random delay between 0 and the max for this attempt. Spreads the retries out across time.

```
attempt 2: wait random(0..200ms)
attempt 3: wait random(0..400ms)
```

AWS recommends "Full Jitter" specifically — and so do many other guides — because it's the most effective spread.

### 3. Cap

Don't let the delay grow without limit. Cap at, e.g., 5 seconds. Otherwise you wait 30 seconds for attempt 7, which is probably worse than just giving up.

### 4. Total time budget

Set a **deadline** for the whole operation. If you've already spent 9 seconds and the next backoff would exceed your 10-second budget, **give up now** rather than waste 1 more second.

### 5. Discriminate retryable errors

Retry on `503`, `504`, timeouts, `ECONNRESET`. **Don't retry** on `400`, `401`, `403`, `404`. These are not transient.

## Idempotency is non-negotiable

When you retry a write, you might cause duplicates. The original might have succeeded (and you didn't hear the ack); the retry creates a second.

**Solution**: use idempotency keys. (Step 06, example 04.)

For non-idempotent operations without keys, you should **not retry blindly**.

## Retry budget: bounding retry traffic globally

Even with backoff, if every client retries 5 times on every failure, a 5% error rate becomes 25% downstream load. Retries can amplify load **5×** during a failure.

The fix: **retry budget**.

A retry budget is a global limit: "we allow at most 10% extra traffic from retries". If retries already total 10% of normal traffic, the budget is exhausted and additional retries are **dropped** (return error immediately).

Implemented as a token bucket alongside the main traffic counter.

Used by Envoy (`retry_budget`), Google's gRPC libraries, many Netflix systems.

## Retry storms: how retries cause outages

```
1. Service B is slightly slow (say 5% errors).
2. Service A retries: error rate at A is 5% × 0.05 = 0.25%, but A is now generating 1.05× the traffic to B.
3. B is slightly more loaded. Errors go up to 7%.
4. A retries even more. B sees 1.07× → 1.10× → 1.15× traffic.
5. B saturates. Real error rate → 50%.
6. A retries 5x on every request. B sees 5× traffic on top of existing.
7. B is dead. A is dead. Outage.
```

This is a **retry storm** or **metastable failure**. The system was operating fine at 95% capacity; a small dip caused retries that pushed it over 100% capacity; the system never recovers.

**Defense**:
- Retry budgets (cap retry amplification).
- Circuit breakers (stop retrying when downstream is in trouble).
- Adaptive retries — back off harder when error rates rise.

## Server-side cooperation: Retry-After header

A well-behaved server tells the client when to retry:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
```

The client honors this instead of guessing. Especially important for **rate-limited APIs** (Stripe, GitHub, Twitter).

## Real-world recipes

### Calling Stripe

```
- Retry on 5xx, 429, network errors.
- Don't retry on 4xx (except 429).
- Use Stripe's Idempotency-Key header so retries don't double-charge.
- Honor Retry-After header on 429.
- Total budget: ~30 seconds for the operation. Beyond that, give up and queue for later.
```

### Calling your own internal gRPC service

```
- Retry on UNAVAILABLE, DEADLINE_EXCEEDED, RESOURCE_EXHAUSTED.
- Don't retry on INVALID_ARGUMENT, NOT_FOUND, PERMISSION_DENIED.
- 3 attempts max; exponential backoff 50ms → 200ms → 800ms with full jitter.
- Circuit breaker per (target service, target instance).
```

### Reading from a queue (consumer)

```
- Retry on transient failures (downstream timeouts).
- After N retries, send to DLQ.
- Backoff between retries (1s → 5s → 25s → ...).
- Track the retry attempt count per message.
```

## Architect's takeaway

- **Retry only retryable errors.** 5xx, timeouts, 429, specific known retryable codes.
- **Exponential backoff with full jitter** is the right shape. Always.
- **Cap delays and total time.** Don't retry forever.
- **Retry storms** are the failure mode to fear. Use retry budgets + circuit breakers.
- **Idempotency keys** are mandatory for retried writes.
- **Service meshes can implement retries centrally** — turn off app-level retries to avoid double-retrying.
