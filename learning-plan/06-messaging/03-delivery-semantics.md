# Example 03 — Delivery semantics: at-most-once, at-least-once, exactly-once

Every messaging system makes a promise about delivery. The promises differ. The cost differs. Picking wrong leads to either lost data or double charges.

## The three guarantees

```
at-most-once   :  0 or 1 deliveries.  Loss is possible. Duplicates are not.
at-least-once  :  1 or more deliveries. Duplicates are possible. Loss is not.
exactly-once   :  Exactly 1 effective delivery. (No loss, no duplicates from the consumer's perspective.)
```

## Why anything other than "exactly once" exists

Because the network can fail at every step:

```
producer → broker:   broker received? did the ack reach the producer?
broker stores:       was it durable before the broker crashed?
broker → consumer:   delivered? consumer acked? did ack arrive?
```

Each of those hops can drop a message **or** drop an ack. If you re-send on no ack, you might create a duplicate. If you don't re-send, you might lose a message.

You can't have both perfect uniqueness *and* perfect delivery from a single mechanism. You need extra coordination — and that coordination is what "exactly-once" really means.

## At-most-once

The producer fires the message once and doesn't retry on failure. Or the consumer acks **before** processing.

**Result:** the message might arrive zero times. It will never arrive twice.

### Use when

- **Loss is cheap, processing is expensive** — telemetry samples, partial logs, "best effort" notifications.
- **Old data is useless** — real-time game state, sensor readings.

### Implementation

- UDP-based protocols.
- Producer: `send` and discard on failure.
- Consumer: ack on receive, process after — if you crash, the message is gone.

## At-least-once (the default)

Producer retries until acked. Consumer acks **after** processing.

**Result:** the message will be processed one or more times. Duplicates are possible.

### Use when

- This is the default. Most brokers default to this (RabbitMQ, Kafka, SQS).
- **You can make the consumer idempotent** (next example).

### Why duplicates happen

```
Consumer:  receive msg → process → write DB → ack broker
                                  ↑
                                  crashes here

Broker re-delivers to another consumer.
Effect: DB write happened twice.
```

Or:

```
Producer: send msg → broker stores → broker acks → producer never gets ack (network)
Producer retries. Now broker has two copies.
```

## Exactly-once: the holy grail (and the marketing word)

Exactly-once is achievable, but it requires either:
1. **Transactional coordination** between broker and consumer's side effects (Kafka transactions), or
2. **Idempotent consumers** — duplicates are detected and discarded.

The clearer term is **"effectively once"**: the broker may deliver multiple times, but the *effects* happen once.

### Kafka transactions (one approach)

Kafka supports producer-side transactions:

```
producer.beginTransaction()
producer.send(msg1, topic1)
producer.send(msg2, topic2)
consumer.commitOffsets(...)  // in same transaction
producer.commitTransaction()
```

If anything fails, the entire batch is rolled back. Consumers using `read_committed` isolation only see committed messages.

This gives **exactly-once across Kafka topics**. It does *not* give exactly-once to **external side effects** like sending an email or charging a credit card — those require idempotency on your side.

### Idempotent consumer (the universal approach)

Make your consumer **idempotent**: running it twice produces the same result as once.

This is the right answer for most systems. Details in the next example.

## Why "exactly-once to an external system" is impossible without idempotency

Suppose you receive `charge_customer($50)`. You:

1. Charge the customer via Stripe API.
2. Ack the broker.

What if step 1 succeeds but the ack in step 2 fails? Broker re-delivers. You charge twice.

What if you swap the order? Ack first, then charge? Then a crash between leaves the customer un-charged. Worse.

**There is no ordering that fixes this without consulting external state.** You must:

- Use a deduplication key (`idempotency_key` in Stripe, for example).
- Track "have I already charged for this event_id?" in your DB.

Once you do that, the underlying broker can deliver as many times as it wants. The effect happens once.

## Trade-off summary

| Guarantee       | Throughput      | Producer config        | Consumer config           | Cost                         |
|-----------------|-----------------|-------------------------|---------------------------|------------------------------|
| At-most-once    | Highest         | Fire and forget         | Ack before process        | Possible data loss           |
| At-least-once   | High (default)  | Retry until acked       | Process then ack          | Duplicate processing         |
| Exactly-once    | Lower           | Transactional / idem ID | Dedup + atomic side-effect | Coordination overhead       |

## What does each broker actually offer?

- **Kafka**: at-least-once by default. Idempotent producer (no dup on retry) + transactions (cross-topic) gives exactly-once *within Kafka*.
- **RabbitMQ**: at-least-once by default (manual ack). At-most-once if you auto-ack.
- **SQS standard**: at-least-once.
- **SQS FIFO**: exactly-once *within the queue* (5-minute deduplication window by message ID).
- **Pulsar**: at-least-once by default; "effectively once" with transactions.

## Architect's takeaway

- **At-least-once + idempotent consumer = the right default.** This is what 95% of production systems do.
- **Don't trust "exactly-once" claims.** They mean "exactly-once within our scope", not "your Stripe charges will never duplicate".
- **At-most-once** is fine for telemetry and similar best-effort data.
- **The hard problem is idempotency** (next example), not the broker.
- **Transactional outbox + at-least-once + idempotent consumer** is the gold standard for production reliability.
