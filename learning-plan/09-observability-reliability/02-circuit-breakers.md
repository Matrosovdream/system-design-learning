# Example 02 — Circuit breakers: stop calling broken things

The pattern that prevents **cascading failures** in distributed systems. Named after household electrical circuit breakers — they trip when there's a fault and refuse to conduct current until reset.

## The problem

Service A calls service B. Service B becomes slow (DB issue, network blip, downstream outage). Each A→B call now takes 30 seconds before timing out.

```
Service A: thread 1 → calling B → waiting 30s
           thread 2 → calling B → waiting 30s
           ...
           thread N → calling B → waiting 30s
```

A's thread pool fills with calls hung on B. New requests to A pile up in the queue. A's queue overflows. **A is now down**, even though A's code is fine.

This is **cascading failure** — B's problem propagated to A. If A has its own callers, they pile up too. The blast radius grows.

## The fix: a circuit breaker around the call

Wrap the call to B in a state machine that tracks failure rate.

```
state: CLOSED  → calls go through. count failures.
state: OPEN    → calls fail immediately. don't even try B.
state: HALF-OPEN → allow one test call. if it succeeds, close. if it fails, re-open.
```

```
CLOSED (normal):
  call B
  if fails: increment failure count
  if failure rate > threshold: → OPEN

OPEN (broken):
  do NOT call B; return error immediately (or fallback)
  after cooldown: → HALF-OPEN

HALF-OPEN (testing):
  allow one call to B
  if success: → CLOSED (and continue normal)
  if failure: → OPEN (back to cooldown)
```

## What "OPEN" buys you

When B is broken, A **stops trying**. A's threads don't get tied up. A returns errors fast (or a fallback). A stays healthy even though B is sick.

```
Service A with circuit breaker:
  request → CB("call B"):
    breaker OPEN → return error or fallback in 0ms
  
  A's threads stay free.
  A's queue stays empty.
  A is still responsive (even if some requests fail).
```

## The thresholds

Typical configuration:

- **Window**: rolling 10-second window of the last N calls.
- **Failure threshold**: 50% of calls failed, or absolute count > 20 failures.
- **Minimum requests**: don't trip on a single failure when traffic is low.
- **Open cooldown**: 30 seconds before trying again.
- **Half-open trial**: 1 (or a small percentage) call allowed to test recovery.

Tuning depends on your latency tolerance and downstream's recovery profile.

## Fallbacks: what to return when the breaker is OPEN

The breaker doesn't just "return error". You decide:

### 1. Return cached / stale data

If B is the recommendations service, the old recommendations from cache are fine. Show those.

### 2. Return a default

If B is the personalization service, fall back to a default homepage. Less personalized, but the user sees a page.

### 3. Return a partial response

If B's data is part of a larger response, return everything else and a flag/empty field for B's part.

### 4. Return an error gracefully

"Recommendations unavailable right now, refresh in a minute." Better than a hung request.

## Why this works at scale

Without a breaker: A's failure mode is **proportional to B's recovery time**. B is down for 10 min → A is busy timing out for 10 min, processing zero useful work.

With a breaker: A detects B is broken within a few seconds, stops calling B, and **keeps doing whatever doesn't need B**. When B recovers, the breaker probes and re-engages.

**A becomes resilient to B's failures.** That's the whole point.

## Implementation: per-instance state

The breaker state is per-process (or sometimes per-process-per-downstream-instance). Each instance of A independently tracks B's health.

This is good: if one B instance is broken and another is fine, smart load balancing combined with per-instance breakers naturally isolates the bad instance.

```
Service A instances:
  A1 → calls B1 (failing) → breaker open
  A1 → calls B2 (fine)    → breaker closed
  A2 → calls B1 (failing) → breaker open
  A2 → calls B2 (fine)    → breaker closed
```

Each (A_i, B_j) pair has its own breaker state.

## Libraries

| Language | Library                                |
|----------|-----------------------------------------|
| Java     | Hystrix (now deprecated), Resilience4j  |
| Go       | sony/gobreaker, afex/hystrix-go         |
| PHP      | ackintosh/ganesha                       |
| Python   | pybreaker, tenacity                      |
| C#       | Polly                                   |
| Node     | opossum                                 |

Most are simple: wrap a function with a breaker, configure thresholds, get automatic state machine + metrics.

In service-mesh setups (Envoy/Istio), breakers can be configured **without changing app code** — set at the sidecar level.

## Common mistakes

### 1. Treating timeouts as success

If B doesn't respond, the call is a "timeout", not a success. Failure to track timeouts as failures defeats the purpose. Make sure your breaker counts timeouts.

### 2. Threshold too high

You set "trip at 100 failures". Failures pile up; cascading happens; the breaker never trips. Threshold should reflect "we've seen enough to know B is broken" — usually a small absolute count or a percentage.

### 3. No fallback

The breaker is open. Your code: `return error`. The caller of A sees an error. The user gets a 500.

Better: have a fallback so most users get a degraded but functional experience.

### 4. Cooldown too short

You set "30 seconds open" but B takes 5 minutes to recover. The breaker keeps re-opening, half-opening, failing, opening. Better: longer cooldown with health-check probe.

### 5. Per-call breakers instead of per-downstream

If you put a breaker on each individual call to B (5 endpoints), then a slow B affects each independently. Better: one breaker for "B" overall (or maybe per-endpoint if endpoints have very different failure characteristics).

## Variants

### Bulkhead

Closely related: instead of (or in addition to) tripping at failure rate, **limit the resources** spent on calls to B. E.g., "no more than 50 concurrent goroutines may call B at once". Even if B is slow, only 50 of A's threads are tied up. Other work continues.

### Rate limit / quota

If you can't outage B, at least don't make B worse. Limit your call rate.

### Hedged requests

Make a duplicate request to a different instance after a small delay. Take the first response. Helps with tail latency without overloading.

## A real story

Netflix popularized the circuit-breaker pattern through their Hystrix library. The Chaos Monkey would kill instances of a service; Netflix's other services had Hystrix breakers around every call. The dead service's breakers would trip, fallbacks would kick in, **users barely noticed**. Other services kept running because they weren't piling up calls to the dead one.

This is the gold standard for production resilience.

## Architect's takeaway

- **Wrap every cross-service call in a circuit breaker.** Especially calls to external APIs (Stripe, Twilio, S3).
- **Combine with bulkheads** to limit resource consumption.
- **Always have a fallback.** A breaker without a fallback is half the pattern.
- **Service meshes** (Istio/Linkerd) can apply breakers without app code changes.
- **Test your breakers** — staging chaos engineering, fault injection.
- **Without breakers, slow downstreams take down everything.** With them, slow downstreams stay isolated.
