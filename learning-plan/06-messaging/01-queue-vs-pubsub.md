# Example 01 — Queue vs Pub/Sub: one-to-one vs broadcast

The two fundamental shapes of messaging. Picking wrong locks you in.

## Queue (point-to-point)

One message → **one** consumer (whoever pulls it first).

```
[producer] ──► [queue] ──► [consumer A]   ← consumer A takes msg-1
                       ──► [consumer B]   ← consumer B takes msg-2
                       ──► [consumer C]   ← consumer C takes msg-3
```

The consumers form a **competing-consumers** group. Adding more consumers spreads the work — the queue is automatic load balancing.

**Example use cases**
- Background jobs: send email, resize image, generate PDF.
- Order processing: each order processed exactly once.
- Webhook delivery: deliver to recipient once, retry on failure.

**Brokers**
- RabbitMQ (classic queue).
- AWS SQS.
- Redis lists with `BLPOP` / `BRPOP`.
- Beanstalkd.

## Pub/Sub (broadcast)

One message → **every** subscriber gets a copy.

```
                   ┌──► [subscriber A]
[publisher] ──► [topic] ──► [subscriber B]
                   └──► [subscriber C]
```

Subscribers are **independent**. Each maintains its own position in the stream.

**Example use cases**
- "user.signup" event → email service + analytics + CRM + welcome-tutorial all act on it.
- Cache invalidation broadcasts.
- Real-time updates to many UIs (WebSocket fan-out).

**Brokers**
- Kafka (topics, consumer groups).
- AWS SNS.
- Google Pub/Sub.
- Redis Pub/Sub.
- NATS.

## Hybrid: pub/sub + consumer groups (Kafka's model)

Kafka unifies both: a topic has **partitions**, and a **consumer group** distributes those partitions among its members.

- **Within one consumer group**: each message goes to exactly one consumer (queue behavior).
- **Across multiple consumer groups**: each group gets a full copy (pub/sub behavior).

```
            topic: user-events (partitions 0, 1, 2, 3)
              ↓
   ┌──────────────────────────────────────────┐
   │ group "email-service"  : consumers A, B  │  ← each msg to one of {A, B}
   │ group "analytics"      : consumers C, D  │  ← each msg to one of {C, D}
   │ group "search-indexer" : consumer E       │  ← every msg to E
   └──────────────────────────────────────────┘
```

This single model covers both patterns. It's why Kafka has eaten so much of the messaging world.

## When each pattern fits

### Use queue (or single consumer group on Kafka) when

- The work is a **task**, not an event ("send this email", "process this payment").
- Exactly one worker should do the job.
- You want to scale workers horizontally without coordination.

### Use pub/sub (multiple consumer groups on Kafka, or SNS) when

- The same event has **multiple independent consumers**.
- New consumers can be added later without producer changes.
- You don't want producers to know who's listening.

### Use both at once

In a typical architecture you'll have:
- A "events" Kafka topic for cross-service broadcasts.
- An SQS or RabbitMQ queue per worker pool for specific task batches.

## A real example: an order is placed

```
1. Order service: db.insert(order) → publish "order.placed" event to Kafka topic.

2. Multiple independent consumers (pub/sub):
   - inventory-service: decrements stock
   - email-service: sends confirmation
   - analytics-service: records funnel event
   - recommendation-service: updates user profile

3. inventory-service publishes "stock.allocated" → fulfillment task queue (SQS).

4. fulfillment workers (queue, competing consumers):
   - worker 1: picks up order 1
   - worker 2: picks up order 2
   - worker N: scales horizontally
```

Both patterns coexist. Pub/sub for the broadcast; queue for the load-balanced task.

## Subtleties to know

### Subscription = remembered state

In pub/sub, the broker **remembers each subscriber's position**. If a subscriber is offline for an hour, when it comes back it can replay the messages it missed (if retention allows).

In a queue, an offline consumer is just... gone. Other consumers absorb its share.

### Persistence

- **Persistent queues** (RabbitMQ default, SQS, Kafka): survive broker restart.
- **Transient pub/sub** (Redis Pub/Sub, in-memory NATS): if a subscriber is offline, the message is **lost**.

Choose persistent unless you have a specific reason not to.

### Fanout cost

Pub/sub with many subscribers fans out write effort. 100 subscribers = effectively 100× write multiplication inside the broker (or in delivery records). Kafka handles this well because subscribers are passive readers from a shared log. Brokers that push to subscribers (like SNS) charge per delivery.

## Architect's takeaway

- **Queue = task; Pub/Sub = event.** Get the word right and the design follows.
- **Kafka's hybrid model** (topics + consumer groups) is the modern standard — it does both.
- **Pub/Sub without persistence is a footgun.** Use Kafka or persistent SNS+SQS if subscribers must catch up after downtime.
- For **internal service-to-service events**, prefer **persistent pub/sub** so adding a new consumer later is just "create another consumer group, replay from offset X".
