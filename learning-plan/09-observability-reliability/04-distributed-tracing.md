# Example 04 — Distributed tracing: following one request across 10 services

In a microservices system, one user request might fan out to 8 service calls. When something is slow, you need to know **which one**. Distributed tracing answers that.

## The model

Every request is assigned a **trace ID** when it enters the system. Every internal call within that request is a **span** under the same trace. Spans form a tree.

```
trace_id: abc-123

  [span: gateway "POST /api/order" 250ms]
    │
    ├─ [span: order-service "createOrder" 230ms]
    │    │
    │    ├─ [span: user-service "getUser" 12ms]
    │    │     └─ [span: postgres "SELECT users" 8ms]
    │    │
    │    ├─ [span: inventory-service "reserveStock" 87ms]
    │    │     └─ [span: postgres "UPDATE stock" 79ms]   ← slow!
    │    │
    │    └─ [span: payment-service "charge" 120ms]
    │          └─ [span: stripe "POST /charges" 115ms]
    │
    └─ [span: kafka "publish order.placed" 5ms]
```

You can see: total request took 250ms. 79ms was spent in one Postgres query inside inventory-service. Now you know where to optimize.

## The components

### Trace context propagation

The trace ID and parent span ID must be passed across every service boundary.

For HTTP, the **W3C Trace Context** standard uses two headers:

```http
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
tracestate: vendor1=opaqueValue1,vendor2=opaqueValue2
```

- `traceparent` carries the trace ID and parent span ID.
- `tracestate` carries vendor-specific extensions.

For gRPC, the same headers are in metadata. For message queues, you put trace context in message headers.

Every service must:
1. **Read** the incoming headers.
2. **Continue** the trace under the same trace_id, with the parent span set to the caller's span.
3. **Propagate** the headers to every outbound call.

If any service drops the headers, the trace is broken there.

### Span lifecycle

```go
// at the start of an operation
ctx, span := tracer.Start(ctx, "createOrder")
defer span.End()

// add attributes (similar to log fields)
span.SetAttributes(
    attribute.String("user_id", userID),
    attribute.Int("item_count", len(items)),
)

// add an event in the timeline
span.AddEvent("validating payment method")

// record an error
if err := chargePayment(ctx, ...); err != nil {
    span.RecordError(err)
    span.SetStatus(codes.Error, "payment failed")
    return err
}
```

Spans have:
- **Name** — what operation.
- **Start/end timestamps** — duration.
- **Attributes** — searchable key/value tags.
- **Events** — points in time within the span.
- **Status** — ok / error.
- **Links** — to other related traces.

### Exporters and backends

A **tracer SDK** records spans in memory. An **exporter** sends them (often batched, async) to a backend. Backends store and visualize traces.

Backends: Jaeger, Tempo (Grafana), Zipkin, Datadog APM, Honeycomb, Lightstep, AWS X-Ray, Google Cloud Trace.

### Sampling

A high-traffic service might serve 100k req/sec. Recording every span at every service = TBs of data. Most teams sample.

**Sampling strategies**:

1. **Head-based sampling**: at trace creation, decide "sample or not". If yes, the whole trace is recorded.
   - Pros: simple, no buffering.
   - Cons: you might miss the rare slow trace.

2. **Tail-based sampling**: record all spans for a trace, decide at the end whether to keep.
   - Pros: you can keep all error traces, all slow traces, plus a baseline.
   - Cons: requires buffering, complexity.

Production tip: head-sample at, say, 1%, but **always sample errors and slow requests**. Most APM products do this automatically.

## What tracing actually fixes

### "Why is this endpoint slow?"

Look at a trace for that endpoint. The span tree shows exactly which service/DB/external call took the time.

### "Service A is timing out — what's it waiting on?"

A's trace shows the outgoing calls and their durations. You'll see which is hanging.

### "We had an incident at 14:23 — what was happening?"

Filter traces around that time. Spot the patterns: a specific endpoint slow, a specific downstream errored, a specific user triggered the issue.

### "We deployed and now something is slow — what changed?"

Compare traces from before and after the deploy. New spans appear? Existing spans take longer?

## OpenTelemetry: the standard

OpenTelemetry (OTel) is the open standard for instrumentation — covering traces, metrics, and logs in one SDK and one wire format.

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/trace"
)
```

You instrument once with OTel; the exporter decides where the data goes (Jaeger, Datadog, etc.). Switching vendors doesn't require re-instrumenting.

For PHP: `open-telemetry/opentelemetry-php`. For Go: `go.opentelemetry.io/otel`.

Most languages have **auto-instrumentation** that wraps HTTP clients, DB drivers, gRPC stubs — you get useful traces with very little code change.

## Implementation cost

The reality:

- **Auto-instrumentation** (HTTP, DB, queue libraries) gives you 60-70% of the value with zero app code changes.
- **Manual instrumentation** (your business logic) gives you the rest. Add `span.SetAttributes(...)` and `tracer.Start(...)` around interesting blocks.
- **The biggest pitfall**: not propagating context across async boundaries (goroutines, queue consumers). The trace ends prematurely.

Cost is real but it pays back the first time you debug a multi-service incident.

## What tracing is bad at

- **Aggregate trends.** Use metrics for that.
- **Detailed code-level debugging.** Use logs for that.
- **Sub-millisecond profiling.** Use a profiler.

Tracing's sweet spot is **"why was this specific request slow / wrong"** in a multi-service system.

## A debugging story

User reports: "Sometimes my checkout takes 30 seconds."

Without tracing: ask each team "is your service slow?" Each says "looks fine". Many hours.

With tracing:
1. Filter traces for `endpoint=/api/checkout AND duration > 10s`.
2. Open one. See 28 seconds in the `notification-service.sendConfirmation` span.
3. Click into it. See its child span: `sendgrid.api.POST` took 28 seconds.
4. Conclusion: SendGrid intermittently slow. Move email sending async (don't block checkout on it).

Total time to diagnose: 5 minutes. The trace pointed directly to the issue.

## Architect's takeaway

- **Tracing is essential when you have 3+ services per request.**
- **W3C Trace Context** is the standard for HTTP header propagation. Use it.
- **OpenTelemetry** is the SDK to standardize on. Vendor-neutral.
- **Auto-instrumentation** gives you most of the value with little code change. Start there.
- **Sample wisely** — head-sample baseline traffic, always keep errors and slow traces.
- **Propagate context everywhere**, including across queues and goroutines. Otherwise traces break.
