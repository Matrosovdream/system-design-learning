# Example 04 — Idempotency: the cure for at-least-once duplication

A message handler is **idempotent** if calling it multiple times with the same input produces the same effect as calling it once. This is the universal answer to "the broker delivered twice".

## What makes something idempotent (or not)

### Naturally idempotent

```sql
UPDATE users SET email = 'alice@x.com' WHERE id = 1;
```

Run this 100 times → same result. The state converges. Idempotent.

```sql
INSERT INTO users (id, email) VALUES (1, 'alice@x.com')
ON CONFLICT (id) DO NOTHING;
```

Run 100 times → row exists exactly once. Idempotent.

```http
PUT /resources/123  { "name": "..." }
```

PUT replaces. Same body = same state. Idempotent.

### Not idempotent by default

```sql
UPDATE accounts SET balance = balance + 100 WHERE id = 1;
```

Run twice → balance changes twice. **Not idempotent.**

```http
POST /charge  { "amount": 100, "customer": "abc" }
```

POST creates. Two POSTs → two charges. **Not idempotent.**

```python
send_email(to="alice@x.com", subject="Welcome!")
```

Two emails sent. **Not idempotent.**

## The four common patterns to make handlers idempotent

### 1. Use a deduplication key (idempotency key)

The producer assigns a unique ID to each *logical* operation. The consumer remembers which IDs it has seen and refuses duplicates.

```python
def handle(message):
    if redis.set(f"seen:{message.id}", "1", nx=True, ex=86400):
        # first time
        process(message)
    else:
        # duplicate, skip
        log.info(f"skipping duplicate {message.id}")
```

Use **Redis with TTL** or a dedicated DB table. The TTL bounds how far back you can detect duplicates — 24 hours is usually enough.

**Stripe API** uses this: every POST has an optional `Idempotency-Key` header. The server stores the key + response for 24 hours. Same key → same response, no double charge.

### 2. Atomic operations with conditional checks

Use the DB's "if not exists" or "compare and set" semantics.

```sql
-- Charge an order, but only if it hasn't been charged yet
UPDATE orders
SET payment_status = 'paid', paid_at = now()
WHERE id = ? AND payment_status = 'pending';
```

If the row was already `paid`, the UPDATE affects 0 rows. The handler can re-run safely.

For domain events: use the **event ID as the primary key** of an "events_processed" table. Insert. Conflict = already processed.

```sql
INSERT INTO events_processed (event_id, processed_at)
VALUES (?, now())
ON CONFLICT (event_id) DO NOTHING;
-- if 0 rows inserted, the rest of the handler returns early
```

### 3. Idempotency via upsert / state machines

Model business events as **state transitions**. The handler reads current state and only acts if the transition is valid.

```
state: pending → processing → completed
                            ↘ failed

handler "complete_order":
  current = db.read_state(order_id)
  if current in (completed, failed):
      return  # already terminal
  if current == pending:
      db.set_state(order_id, processing)
  do work
  db.set_state(order_id, completed)
```

Even if the handler runs twice, the second run sees the state as already completed and returns immediately.

### 4. Composite keys for derived data

If you're generating, e.g., a notification per (user, event), make `(user_id, event_id)` the unique key.

```sql
CREATE TABLE notifications (
  user_id INT,
  event_id UUID,
  ...
  PRIMARY KEY (user_id, event_id)
);

INSERT INTO notifications (...) VALUES (...)
ON CONFLICT DO NOTHING;
```

Replay-safe.

## A real example: order paid → send receipt + decrement stock

Bad (not idempotent):

```python
def on_order_paid(event):
    send_email(event.user_email, "Receipt", ...)
    decrement_stock(event.product_id, event.qty)
```

A duplicate delivery sends two emails and decrements stock twice. Bad.

Good (idempotent):

```python
def on_order_paid(event):
    # 1. Record this event was processed (atomic)
    if not record_event_processed(event.id):
        return  # already done

    # 2. Side effects with idempotency keys passed through
    send_email(event.user_email, "Receipt", idempotency_key=f"receipt:{event.id}")
    decrement_stock(event.product_id, event.qty, idempotency_key=f"stock:{event.id}")
```

The email service has its own dedup table keyed on `idempotency_key`. The stock service does too. Each component is independently safe.

## Idempotency at the API boundary

Expose idempotency keys to your clients. This is what Stripe, AWS, and most modern APIs do.

```http
POST /v1/payments
Idempotency-Key: 4e2b8a9c-...
Content-Type: application/json

{ "amount": 5000, ... }
```

The server stores the key + response. Retries by the client → same response, no duplicate charge. **Critically: this also defends against client bugs**, not just message-broker bugs.

## The pitfalls

### Pitfall 1: dedup key in the wrong scope

If you key dedup by `(consumer_id, message_id)`, and you redeploy the consumer (new ID), duplicates leak through. Use **a key that's stable across deploys** — usually a domain-level UUID generated by the producer.

### Pitfall 2: TTL too short

You dedup with a 1-hour TTL. A duplicate arrives 2 hours later (broker had a glitch, retried after long delay). Boom, processed twice.

Use TTLs longer than your worst-case re-delivery window. 24 hours is a safe default for most queueing systems.

### Pitfall 3: forgetting external side effects

You dedup at your consumer entry point but call out to a payment API or send-email API that doesn't dedup. The dedup at your side only stops *your* DB writes; the external call already happened.

Use idempotency keys on the external call too.

### Pitfall 4: partial-failure between dedup and side-effect

```
1. record_event_processed(event.id)  ← committed
2. send_email(...)                    ← crash here
3. (never executes)
```

Now the event is marked processed but the email never went. On retry, dedup blocks the retry. Email lost forever.

**Fix:** record the event-processed marker as part of the same atomic transaction as the side effect, **or** the marker should record "completed" only after the side effect succeeds, **or** the side effect should itself be retryable and idempotent.

## Architect's takeaway

- **Idempotency is the way to make at-least-once delivery safe.** Don't try to make the broker exactly-once.
- **Every external call should accept an idempotency key.** Stripe is the design pattern.
- **State-machine your domain events.** Each transition is a guard that prevents double-processing.
- **Dedup TTL must exceed worst-case redelivery.** 24 hours is a fine default.
- **Test idempotency by intentionally re-delivering messages** in staging. It's the only way to be sure.
