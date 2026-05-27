# Step 06 — Messaging & queues

Synchronous request/response is the default. But the moment you have multiple services, slow downstream work, retries, or anything async — you need a **message broker**.

This step is about brokers, queues, pub/sub, event streams, and the surprisingly subtle problem of *delivery semantics*.

## Goals

- Tell the difference between a **queue** (point-to-point) and a **pub/sub** (broadcast) and pick correctly.
- Pick Kafka vs RabbitMQ vs SQS for a real workload.
- Define **at-most-once**, **at-least-once**, **exactly-once** delivery and what each costs.
- Make a consumer **idempotent** so at-least-once delivery is safe.
- Know what a **dead-letter queue (DLQ)** is and what belongs in it.
- Build an **event-driven order flow** in your head, end to end.

## Key concepts

1. **Queue vs pub/sub** — one-to-one consumption vs broadcast.
2. **Brokers** — Kafka, RabbitMQ, SQS, Pub/Sub, NATS, Pulsar.
3. **Delivery semantics** — at-most-once (fast, lossy), at-least-once (default, dups), exactly-once (effortful).
4. **Idempotency** — making "ran twice" indistinguishable from "ran once".
5. **Ordering** — global, per-partition, per-key, none.
6. **Backpressure** — slow consumers shouldn't crash producers.
7. **Dead-letter queues** — quarantine for poison messages.
8. **Outbox pattern** — atomic DB-write + queue-publish without 2PC.
9. **Event-driven architecture** — services publish events; others react.

## Reading

- **Primer**: message queues, asynchronism.
- **DDIA**: Chapter 11 (Stream Processing) — the deep theoretical view.
- **Kafka docs**: "Kafka in 5 minutes" + the consumer-groups page.
- **Microsoft cloud patterns**: Queue-Based Load Leveling, Competing Consumers.

## Examples in this folder

- `01-queue-vs-pubsub.md` — one-to-one vs broadcast.
- `02-kafka-vs-rabbitmq-vs-sqs.md` — picking the right broker.
- `03-delivery-semantics.md` — at-most/at-least/exactly-once.
- `04-idempotency.md` — the fix for at-least-once duplication.
- `05-dead-letter-queue.md` — what to do with the unprocessable.
- `06-event-driven-order-flow.md` — a full e-commerce checkout in events.

## Self-check

1. You want every consumer to receive every message. Queue or pub/sub?
2. Why does Kafka call them "topics with partitions" but RabbitMQ calls them "queues with exchanges"?
3. You receive an order-paid event twice. Charge the user twice? Why or why not?
4. What's the simplest way to make a Stripe-payment handler idempotent?
5. A message has been retried 30 times and still fails. What now?
