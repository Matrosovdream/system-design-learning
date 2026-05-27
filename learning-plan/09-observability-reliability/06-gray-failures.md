# Example 06 — Gray failures: when "up" isn't the same as "working"

The most insidious kind of failure. A service is **technically up** — health checks pass, monitors are green — but it's **not actually serving correctly**. Some users are unhappy; the dashboards say everything's fine.

## What gray failure looks like

- The DB is responding to `SELECT 1` but production queries are timing out.
- A service is at 99% CPU and serving requests very slowly, but not refusing them.
- One out of three replicas has a corrupted disk and returns stale data.
- The load balancer health check probes a special endpoint that doesn't exercise the real path; the real path is broken.
- The service has stopped consuming from one of its Kafka partitions but the others are fine.
- The network has 5% packet loss to one AZ — some clients see timeouts, most don't.
- The pod has been OOM-killed and restarted 47 times in the last hour but each restart shows "ready".
- A bug only affects requests with certain payloads; affected users see errors, others don't.

The system is in a **degraded but undetected** state. Monitors are green. Customer service tickets are red.

## Why this is uniquely painful

- **Health checks lie.** They check easy paths, not real workloads.
- **Aggregate metrics hide it.** Average latency looks fine; the slow 5% are invisible.
- **It's been happening for a while** when you finally notice.
- **It might be self-inflicted** — your monitor *says* it's working because the monitor itself was the easy path.

## Causes — the categories

### 1. Health-check mismatch

Health checks should exercise the real path. They don't.

```
/healthz endpoint: returns "ok" — no DB call.
Real endpoint: calls DB, which is timing out.
Health check passes; real users see errors.
```

**Fix**: health checks should hit the dependencies that matter. Or have **shallow** (process alive) and **deep** (functioning) health checks; route on shallow, alert on deep.

### 2. Partial failures

Multi-replica system; one replica broken.

```
3 cache nodes. Node 2 is in a weird state — accepts connections but returns garbage.
LB routes 33% of traffic to node 2 → 33% of users get garbage.
Aggregate: 67% success looks "mostly fine"; specific users see catastrophe.
```

**Fix**: per-instance health, automatic eviction, customer-level tracking.

### 3. Cardinality blind spots

You aggregate across users. One user (or one tenant) is broken; the aggregate looks fine.

```
Tenant A: 100% error rate.
Tenants B-Z: 0.1% error rate.
Average: ~4% — looks fine if your alert is at 10%.
```

**Fix**: monitor by tenant / customer / route. Watch the worst, not the average.

### 4. Gradual degradation

The system slowly gets worse. You compare to "yesterday", which was already worse than the day before.

```
Day 1: p99 latency = 100ms
Day 2: p99 = 110ms
Day 3: p99 = 120ms
...
Day 30: p99 = 1100ms — but no day-over-day jump triggered an alert.
```

**Fix**: long-baseline trends. Compare to a month ago, not just yesterday.

### 5. Self-monitoring failures

The monitor itself is broken.

```
Prometheus scrape failed silently for 6 hours.
Dashboards show "no data" but you don't notice.
Real metrics are bad; you don't see it.
```

**Fix**: alert on absence of data, not just on bad data.

### 6. Asymmetric network failures

Server A can reach client; client can't reach server. The server thinks it's serving fine.

```
Server's logs: "served 1000 requests, all 200 OK".
Client's experience: "I can't connect."
```

**Fix**: client-side monitoring (real user monitoring, synthetic checks from outside your network).

## Defensive patterns

### Black-box monitoring from outside

A monitoring agent **outside** your infrastructure (different AZ, ideally different cloud) calls your service like a real user does. Compares response.

If users can't reach you, this catches it even when internal monitors are green.

Examples: Pingdom, UptimeRobot, Datadog Synthetics, AWS Synthetics, Cloudflare Workers.

### Deep health checks

Distinguish:

- **Liveness probe** (`/healthz`): is the process responding at all? Used by k8s to restart pods.
- **Readiness probe** (`/readyz`): can this instance serve traffic? Check dependencies. Used by LB to add/remove from rotation.

Make readiness deep — checks DB, cache, downstream API. If it fails, the LB removes the instance until it's truly ready.

But: don't make it so deep that one bad downstream takes you out of rotation when you could still serve degraded.

### Per-instance metrics

Don't only track aggregate "service x is good". Track per-instance:
- Error rate per pod / container.
- p99 latency per instance.
- Resource utilization per instance.

A bad apple in a basket of 100 is invisible in the average; visible per-instance.

### Per-customer monitoring

For multi-tenant systems: per-tenant error rates, per-tenant latency. Alert if one customer's metrics regress dramatically vs theirs historically.

### Canary requests

Periodically (e.g., every minute) send a synthetic request that exercises the full happy path. Track its success and latency. Catches "the API is up but no one's actually completing checkout" scenarios.

### Real user monitoring (RUM)

Have the user's browser/app report timing and errors. RUM telemetry tells you **what users actually see**, not what your servers think happened.

Tools: Datadog RUM, Sentry, Honeycomb, New Relic Browser.

### Graceful degradation paths

Even when monitoring catches a gray failure, your system response matters. Build:

- **Feature flags** to disable specific features that are degraded.
- **Fallbacks** that return less-fresh but-still-useful results.
- **Bulkheads** that prevent one broken feature from taking out the whole product.

## Detecting gray failure: what to alert on

Beyond the four golden signals (latency, traffic, errors, saturation), add:

- **Anomaly detection** on critical metrics: catch deviations from a learned baseline.
- **Burn-rate alerts** on SLO budgets (catches slow leaks).
- **Distribution monitoring**: alert when p99 / p50 latency ratio drifts (some users hit a bad path).
- **Absence-of-data alerts**: alert when expected metrics stop flowing.
- **Black-box checks**: external synthetic monitoring.
- **Customer report integration**: when 10 support tickets per hour mention the same word, page someone.

## A real story (composite)

A SaaS company shipped a deploy. Internal metrics looked fine. 12 hours later, a customer support ticket: "I can't log in." Investigation revealed:

- Auth service returned `200 OK` with a payload that the new mobile app version didn't understand.
- Old mobile app version: fine. New version: broken.
- Server logs: "successful auth, 200 OK". Real users: cannot log in.
- Aggregate error rate: 0.0%, because the server thought it was succeeding.

The fix was simple. The detection took 12 hours **because the server-side monitors had no way to know the response was useless.**

Lessons:
- Add **client-side error reporting** (Sentry, custom event).
- Add **integration smoke tests** that exercise the real client codepath.
- Track **business-level success** (logins per hour) in addition to HTTP-level success.

## Architect's takeaway

- **Health checks must exercise the real path** to mean anything.
- **Always monitor from outside** your own infrastructure (black-box monitoring).
- **Per-customer, per-instance, per-route metrics** find what aggregates hide.
- **Real user monitoring** is the truth — your server's view is wishful thinking.
- **Business-level signals** (orders/min, signups/min, logins/min) catch what server-level metrics miss.
- **Gray failures are the hard ones.** You can't completely prevent them; you can detect them faster with the right layered monitoring.
