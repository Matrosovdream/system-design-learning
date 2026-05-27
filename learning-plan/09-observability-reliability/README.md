# Step 09 — Observability & Reliability

A system you can't observe is a system you can't trust. A system without resilience patterns is one that turns small failures into outages. This step covers both.

## Goals

- Distinguish **logs, metrics, traces** and know what each is good for.
- Set up basic **SLI / SLO / error-budget** thinking (you saw it in step 01; here it gets practical).
- Implement **circuit breakers**, **retries with backoff**, **bulkheads** correctly.
- Implement **distributed tracing** (W3C trace context, OpenTelemetry).
- Recognize **gray failures** and design defenses.
- Plan for incidents: runbooks, on-call, postmortems.

## Key concepts

1. **Three pillars of observability**: logs, metrics, traces.
2. **Black-box vs white-box monitoring** — outside-in synthetic checks vs internal instrumentation.
3. **SLI / SLO / error budgets** — turning reliability into a measurable, tradeable resource.
4. **Circuit breakers** — stop calling broken downstreams.
5. **Retries with exponential backoff + jitter** — and when retries make things worse.
6. **Bulkheads** — isolating failure to one part of the system.
7. **Distributed tracing** — following a request across many services.
8. **Gray failures** — when a service isn't down but isn't right.
9. **Runbooks, on-call, postmortems** — the human/operational side.

## Reading

- **Google SRE book** — chapters 6 (Monitoring), 22 (Addressing Cascading Failures), 14 (Managing Incidents). Free at https://sre.google/sre-book/
- **OpenTelemetry docs** — the standard for instrumentation.
- **Release It!** by Michael Nygard — the bible of stability patterns.

## Examples in this folder

- `01-three-pillars.md` — logs, metrics, traces — when each shines.
- `02-circuit-breakers.md` — the pattern that prevents cascades.
- `03-retries-and-backoff.md` — retries done right (and badly).
- `04-distributed-tracing.md` — following one request across 10 services.
- `05-error-budgets.md` — the SRE management technique.
- `06-gray-failures.md` — the hardest kind of failure to detect.

## Self-check

1. Your DB latency is high. What signal in logs, metrics, traces do you check first?
2. Service A retries 5 times when service B is slow. Service B gets even slower. Why and how do you fix it?
3. A circuit breaker is open. What does that mean and what's the correct behavior?
4. Your SLO is 99.9% and you've spent 80% of your monthly error budget by day 12. What's the org's call?
5. Service B is "up" by all monitors but somehow some users get errors. What kind of failure is this?
