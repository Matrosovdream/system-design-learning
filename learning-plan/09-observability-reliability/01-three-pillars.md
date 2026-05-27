# Example 01 — The three pillars of observability: logs, metrics, traces

You can't fix what you can't see. Three telemetry primitives give you visibility into a running system. Each is good at different things; you need all three.

## Logs

Discrete events with text and structured fields.

```json
{
  "ts": "2026-05-27T14:23:01.847Z",
  "level": "error",
  "service": "orders",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "user_id": "12345",
  "order_id": "67890",
  "err": "stripe charge failed: card_declined",
  "duration_ms": 472
}
```

### Strengths

- **High-fidelity per event** — full context for a specific occurrence.
- **Searchable** — "show me every error mentioning order 67890".
- **Familiar** — everyone has worked with logs.

### Weaknesses

- **Expensive at high volume** — terabytes/day at scale.
- **Not aggregated** — you can count errors with log search, but it's slow.
- **Sampling is tricky** — drop the wrong sample and miss the bug.

### Use logs for

- **Debugging a specific incident** — what happened to this request?
- **Audit trails** — what did the system do, when?
- **Errors and exceptions** — full stack traces and context.

### Tools

- **Self-hosted**: ELK / OpenSearch + Logstash + Kibana, Loki + Grafana.
- **Managed**: Datadog Logs, Splunk, New Relic Logs, AWS CloudWatch Logs.

## Metrics

Numeric measurements aggregated over time.

```
http_requests_total{service="orders", status="500"} = 12473
http_requests_total{service="orders", status="200"} = 1843291
db_query_duration_seconds{service="orders", quantile="0.99"} = 0.187
```

### Strengths

- **Cheap** — a metric is bytes per data point; you can have millions of these per second per service for pennies.
- **Aggregated** — perfect for dashboards, alerts, capacity planning.
- **Time-series friendly** — graph trends, detect regressions.

### Weaknesses

- **Pre-aggregated** — you lose individual event context. You see "1473 errors at 14:00" but not which ones.
- **Cardinality explosions** — too many label dimensions blow up storage. Avoid metrics like `http_requests{user_id="..."}` (millions of users → millions of series).
- **Bounded dimensions** — must be low-cardinality.

### The four "golden signals" (Google SRE)

For every service, measure at least:

1. **Latency** — how long requests take. Look at percentiles, not averages (p50, p95, p99).
2. **Traffic** — how many requests/second.
3. **Errors** — error rate (and types).
4. **Saturation** — resource utilization (CPU, memory, disk, queue depth).

These four catch ~80% of issues.

### Use metrics for

- **Dashboards** — see the system health in real time.
- **Alerts** — page when something exceeds threshold.
- **Capacity planning** — "we'll outgrow our cluster in 3 months at this rate".

### Tools

- **Self-hosted**: Prometheus + Grafana (de facto standard), VictoriaMetrics, Mimir.
- **Managed**: Datadog Metrics, New Relic, CloudWatch Metrics, Honeycomb.

## Traces

End-to-end record of a single request as it flows through multiple services.

```
[gateway: 245ms]
   ├─[orders-service: 234ms]
   │    ├─[user-service: 12ms]
   │    ├─[inventory-service: 87ms]
   │    │    └─[postgres: 79ms]   ← here's the slow step
   │    └─[stripe: 120ms]
```

Each "span" represents one operation. Spans are linked into a "trace" by a shared trace ID.

### Strengths

- **Shows the path of one request** across many services.
- **Latency attribution** — see exactly where time goes.
- **Causal analysis** — "which downstream caused this slow response?"

### Weaknesses

- **High data volume** at scale — usually sampled (1-10% of traces).
- **Requires instrumentation** in every service.
- **Cross-team coordination** — every service needs to propagate the trace ID.

### Use traces for

- **Latency investigations** — "why is this endpoint slow?"
- **Service dependency analysis** — what calls what.
- **Cross-service debugging** — when the bug is in the seam between two services.

### Tools

- **Self-hosted**: Jaeger, Zipkin, Tempo, OpenTelemetry collector + storage backend.
- **Managed**: Datadog APM, Honeycomb, Lightstep, AWS X-Ray.

## How the three work together

A typical debugging story:

```
1. Alert fires: "error rate > 1% on /api/checkout"
   Source: METRICS dashboard exceeded threshold.

2. Look at TRACES filtered to /api/checkout errors.
   See: 90% of failures happen in the payment-service span.
   The payment-service span has an error tag.

3. Look at LOGS for payment-service around that time, filtered to the trace_id.
   See: "stripe API returned 503".

4. Conclusion: Stripe was having an outage. Confirmed on their status page.
   Action: nothing to fix on our side (this time). Add a fallback to a secondary payment provider.
```

Each pillar contributed:
- Metrics told you there was a problem.
- Traces narrowed it to one component.
- Logs gave you the specific cause.

## Cardinality: the hidden gotcha

Metrics scale by **number of time series**, not number of data points. Each unique combination of labels is one series.

```
http_requests{service="orders", method="GET", status="200"}                       ← 1 series
http_requests{service="orders", method="GET", status="200", user_id="alice"}      ← 1 series per user
```

If you label by user_id (1M users), you get 1M time series **per metric**. Prometheus will choke. This is the #1 mistake people make with metrics.

**Rule**: only label by dimensions with **bounded cardinality** (status, method, endpoint, region). Never label by user_id, session_id, request_id — those go in **traces** and **logs**.

## OpenTelemetry: one standard for all three

OpenTelemetry (OTel) is the emerging standard for instrumentation. One SDK in your app produces logs, metrics, and traces in a vendor-neutral format. You configure an exporter to send to Datadog, Honeycomb, Prometheus, Jaeger — whatever.

OTel is now the answer to "how do I instrument my service for observability?". Use it.

## A worked example: a Go service with all three

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/metric"
    "go.uber.org/zap"
)

var (
    tracer = otel.Tracer("orders")
    meter  = otel.Meter("orders")
    log    = zap.NewProduction()
    requestCounter, _ = meter.Int64Counter("http_requests_total")
)

func handleOrder(ctx context.Context, w http.ResponseWriter, r *http.Request) {
    ctx, span := tracer.Start(ctx, "handleOrder")
    defer span.End()

    requestCounter.Add(ctx, 1, metric.WithAttributes(
        attribute.String("endpoint", "/order"),
    ))

    log.Info("processing order",
        zap.String("trace_id", span.SpanContext().TraceID().String()),
        zap.String("order_id", orderID),
    )

    if err := process(ctx, order); err != nil {
        span.RecordError(err)
        log.Error("order failed", zap.Error(err))
        http.Error(w, "internal", 500)
        return
    }
}
```

The same `trace_id` ties together the trace, the log line, the metrics — you can pivot between all three.

## Architect's takeaway

- **Use all three pillars.** Each answers different questions.
- **Metrics for "is something wrong?"; traces for "where?"; logs for "exactly what happened?".**
- **Watch metric cardinality.** High-cardinality dimensions belong in logs and traces, not metrics.
- **Standardize on OpenTelemetry.** It's the future-proof choice.
- **The four golden signals (latency, traffic, errors, saturation)** are your minimum metric coverage. Have them for every service.
- **Tag everything with a trace_id.** It's how you tie observability data together.
