# Example 06 — Event-driven order flow: an e-commerce checkout in events

A worked walkthrough of how a checkout becomes a series of events flowing through multiple services. This is the architecture that powers Amazon-class systems and is now the default for most B2C e-commerce.

## The product

User clicks "Place Order". Behind the scenes:
1. Validate payment.
2. Reserve inventory.
3. Send confirmation email.
4. Notify warehouse to fulfill.
5. Update analytics.
6. Update recommendation engine.

In a monolith: one giant function does it all, serially, in one HTTP request. Slow (5+ seconds), brittle (any failure rolls back the whole thing), tightly coupled (analytics changes require redeploying the order code).

In event-driven architecture: each step becomes an event, each service reacts independently.

## The event flow

```
[Order Service]
   POST /api/orders → validate input → db.insert(order, status='pending')
                  └─► publish "order.placed"

                    ↓
        ┌───────────┴────────────┬──────────────┬──────────────┐
        │                        │              │              │
   [Payment]               [Inventory]    [Analytics]    [Recommender]
   listens to              listens to     listens to     listens to
   "order.placed"          "order.placed" "order.placed" "order.placed"
        │                        │
        ├─► charge customer      ├─► reserve stock
        │   (Stripe API)         │   (db update)
        └─► publish              └─► publish
            "payment.succeeded"      "stock.reserved"
            "payment.failed"         "stock.unavailable"

                    ↓
              [Order Service]
              listens to payment & stock events
              updates order status:
              - both succeed → "confirmed"
              - either fails → "cancelled" + refund / restore

                    ↓
              [Order Service]
              publishes "order.confirmed"

                    ↓
        ┌───────────┴────────────┬─────────────────────┐
        │                        │                     │
   [Email]                 [Fulfillment]         [Loyalty]
   listens to              listens to            listens to
   "order.confirmed"       "order.confirmed"     "order.confirmed"
        ├─► send confirmation  ├─► task for warehouse  ├─► add points
        └─► publish            └─► publish              └─► publish
            "email.sent"           "fulfillment.queued"   "loyalty.granted"
```

## Detailed step-by-step

### 1. User clicks "Place Order"

```
POST /api/orders → Order Service

1. Validate input (auth, product IDs valid, quantities > 0).
2. Insert into DB:
   INSERT INTO orders (id, user_id, items, total, status)
   VALUES (uuid, user, jsonb, $50, 'pending');
3. Publish event:
   topic: "orders"
   event: { type: "order.placed", order_id: uuid, items: [...], user_id: ... }
4. Return 202 Accepted + order_id to the client.
```

Response time to user: ~50 ms. Frontend can poll `GET /api/orders/uuid` or use WebSocket/SSE for status updates.

### 2. Payment service handles charge

```
on "order.placed":
   read order from DB (or use payload)
   call Stripe with idempotency_key = order_id
   if success:
      db.insert(payment, status='succeeded', stripe_charge_id=...)
      publish "payment.succeeded"
   else:
      db.insert(payment, status='failed', reason=...)
      publish "payment.failed"
```

Idempotency key is `order_id` — re-delivery of the message won't double-charge.

### 3. Inventory service reserves stock

```
on "order.placed":
   for each item:
      UPDATE stock SET reserved = reserved + qty
      WHERE sku = ? AND (available - reserved) >= qty
      RETURNING reserved;
   if all succeeded:
      publish "stock.reserved"
   else:
      publish "stock.unavailable" (with details of which sku failed)
```

The conditional UPDATE is the **invariant** — it cannot oversell. Combined with idempotency keys, it's safe to re-run.

### 4. Order service correlates the two events

```
on "payment.succeeded" AND "stock.reserved":
   UPDATE orders SET status = 'confirmed' WHERE id = ? AND status = 'pending';
   publish "order.confirmed"

on "payment.failed" OR "stock.unavailable":
   UPDATE orders SET status = 'cancelled' WHERE id = ? AND status = 'pending';
   if payment had succeeded: publish "payment.refund.required"
   if stock was reserved: publish "stock.release.required"
   publish "order.cancelled"
```

This is the **saga** pattern (more in step 07). The Order Service is the **orchestrator** — it tracks which sub-events have arrived and decides the next step.

Alternative: **choreography** — no orchestrator, each service reacts to events and emits its own. Simpler at small scale, harder to reason about as the flow grows.

### 5. Downstream services react

- Email Service sends confirmation.
- Fulfillment Service queues a task for the warehouse.
- Loyalty Service awards points.

Each is independent. Each can be down without affecting the others. The Order Service's "order.confirmed" event is persistent (in Kafka), so a temporarily-down service catches up when it returns.

## Why this is better than the monolith

### Resilience

If Recommendation Service is down, **the order still succeeds**. The event sits in Kafka for that consumer group, and the recommender catches up when it's back. No partial failure cascading to the user.

In a monolith: the whole transaction might fail or the entire HTTP call hangs.

### Speed

The user gets a `202 Accepted` in 50 ms. The expensive work (Stripe API, stock lookups, emails) happens async. The frontend can show "Order placed! We'll email you when it's confirmed."

### Decoupling

Want to add a new downstream — say a fraud-detection service? Just add a consumer that subscribes to `order.placed`. The Order Service doesn't change at all.

### Auditability

Every event is in Kafka with full history. "Show me everything that happened for order #123" → query event log. Stupendously valuable for debugging.

### Replay

Bug found in the loyalty service? Fix it, reset the consumer group's offset to before the bug, replay the last 24 hours of events. The bug is undone.

## What this costs you

### Complexity

Many services, many topics. You need to draw a diagram. New engineers take longer to understand the flow.

### Eventual consistency

Order is `pending` for a few seconds after the user clicks. Frontend must show "processing" state. Some users will refresh and see "pending" briefly.

### Operational overhead

Kafka cluster (or managed service). Monitoring per-service. Dead-letter queues per consumer. Schema registry to avoid breaking changes.

### Idempotency is non-negotiable

Every consumer must be idempotent. Every external call must use idempotency keys. **The whole pattern unwinds without this discipline.**

### Distributed tracing becomes essential

Tracking "what happened to order #123" across 6 services and 12 events is impossible without distributed tracing (Jaeger, OpenTelemetry). You're not optional on this.

## The transactional outbox pattern

A subtle correctness issue: the Order Service does:

```
db.insert(order)
publish "order.placed" to Kafka
```

What if the DB write succeeds but the Kafka publish fails? Or vice versa? Now the system's state is inconsistent.

**Outbox** fixes this:

```
1. In one DB transaction:
   - INSERT INTO orders ...
   - INSERT INTO outbox (event_type, payload, status='pending')
2. Commit. Both rows live or die together. ✓
3. Background relay reads outbox, publishes to Kafka, marks as 'sent'.
```

Atomicity is preserved at the DB level. The relay can crash and resume — at-least-once delivery downstream, idempotency consumers absorb the dups.

This is the **right pattern** for event publishing in production. Don't publish to Kafka directly from inside a transaction — the operation isn't atomic with the DB write.

## Architect's takeaway

- **Event-driven architecture trades simplicity for resilience and flexibility.** Pick it when the trade is worth it.
- **The orchestrator/choreography choice** is real. Orchestrators are easier to debug. Choreography is more decoupled.
- **The outbox pattern** is the production way to publish events safely.
- **Idempotency, dead-letter queues, distributed tracing** are mandatory. Not optional. Don't start without them.
- This pattern scales to **billions of events/day** in real production systems. It's the architecture behind Amazon, Uber, Stripe, Shopify.
