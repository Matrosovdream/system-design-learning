# Example 02 — Kafka vs RabbitMQ vs SQS: picking the right broker

The three most common messaging systems. Each was designed for a different problem. Picking wrong locks you into operational pain.

## At a glance

| Property                | Kafka                              | RabbitMQ                          | AWS SQS                          |
|-------------------------|------------------------------------|-----------------------------------|----------------------------------|
| Model                   | Distributed log + consumer groups  | Smart broker + dumb consumer      | Managed queue                    |
| Best at                 | High-throughput streams, replay    | Complex routing, low latency      | Simplicity, decoupling, scale    |
| Throughput              | Millions of msg/sec                | 10s of thousands msg/sec          | Effectively unlimited (managed)  |
| Latency                 | Low (ms)                           | Very low (sub-ms possible)        | ~10-50ms                         |
| Ordering                | Per-partition                      | Per-queue                         | FIFO queues: per-group; standard: best effort |
| Persistence             | Always (retention configurable)    | Optional                          | Always                           |
| Replay                  | Yes (rewind by offset)             | No (consume = delete)             | No                               |
| Routing                 | Topics + partitions only           | Exchanges, bindings, headers      | Simple queue                     |
| Operational overhead    | High (run a cluster)               | Medium                            | Zero (managed)                   |
| Cost model              | Self-hosted infra, or Confluent    | Self-hosted, or CloudAMQP         | Per-million-message (cheap)      |

## Kafka: when the goal is "the log is the source of truth"

Kafka is a **distributed append-only log** with consumer groups. Producers append. Consumers read from offsets they track themselves. Messages live in the log for hours, days, or forever (compacted topics).

### When Kafka wins

- **Event-sourced systems** — the log *is* the data.
- **Stream processing** — Kafka Streams, Flink, Spark Streaming.
- **High-throughput pub/sub** — millions of events/sec.
- **CDC pipelines** — DB changes streamed to many destinations (search index, warehouse, ML pipeline).
- **Need to replay** — bug found in consumer? Rewind, reprocess.
- **Auditable history** — every event preserved.

### Where Kafka hurts

- **Complex routing** — Kafka is dumb pipes, smart consumers. If you need "send to queue A if header.priority == 'high'", you build that in the consumer.
- **Low-latency request/response** — Kafka is good but not the best for sub-millisecond latency.
- **Selective retry / dead-letter** — possible but takes machinery.
- **Operational complexity** — running a Kafka cluster (Zookeeper or KRaft, brokers, monitoring) is real work. Use Confluent Cloud or MSK if you can.

## RabbitMQ: when routing is complex and per-message logic matters

RabbitMQ is the AMQP reference implementation. Producer → **exchange** → bound to **queues** by **routing rules**. The broker is smart; routing happens server-side.

### When RabbitMQ wins

- **Complex routing** — fanout, direct, topic, headers exchanges all built-in.
- **Per-message TTL, priority queues, dead-letter exchanges** out of the box.
- **Low-latency RPC patterns** — RabbitMQ does request-reply queues elegantly.
- **Smaller scale** — tens of thousands of msg/sec on a small cluster is comfortable.
- **Polyglot environments** — AMQP libraries exist everywhere.

### Where RabbitMQ hurts

- **Massive throughput** — Kafka handles 10-100× more per node.
- **Long retention / replay** — RabbitMQ deletes consumed messages. Stream plugin exists but isn't its native model.
- **Operational complexity at scale** — cluster partitions, mirrored queues, are real concerns above ~100k msg/sec.

## SQS: when you want to not think about it

AWS SQS is a fully managed queue. You don't run anything. You pay per million messages. It scales effectively forever.

### When SQS wins

- **AWS-native architectures.**
- **You don't want to run brokers** — saves an engineer's time per quarter.
- **Variable / bursty workloads** — autoscales for free.
- **Simple competing-consumers queues** — the common 80% case.
- Combined with **SNS** for pub/sub fanout (SNS → multiple SQS queues).

### Where SQS hurts

- **No replay** — once consumed, gone.
- **Standard SQS gives no ordering**; FIFO SQS is slower and capped at 3000 msg/sec per group.
- **Higher latency** than self-hosted RabbitMQ (~10-50ms typical).
- **Lock-in** to AWS.
- **No real pub/sub** without SNS+SQS pattern.

## A decision tree

```
Do you need to replay events / event-sourcing / CDC?
   yes → Kafka
   no  → continue

Do you need complex routing / priority / per-message TTL?
   yes → RabbitMQ
   no  → continue

Do you want zero-ops?
   yes → SQS (+ SNS if fanout needed)
   no  → continue

Need >100k msg/sec sustained?
   yes → Kafka
   no  → RabbitMQ or SQS — pick on team familiarity
```

## Honourable mentions

- **NATS / NATS JetStream** — extremely fast, simple, great for service-to-service.
- **Apache Pulsar** — Kafka-like, with multi-tenancy and tiered storage.
- **Redis Streams** — lightweight queue-and-stream in your existing Redis.
- **Google Pub/Sub** — like Kafka + SQS combined, fully managed.
- **Azure Service Bus** — RabbitMQ-style managed broker.

## A real-world architecture using all three

Big SaaS company:

- **Kafka** as the event spine: every domain service emits its events here, ML/analytics/search all consume from Kafka topics.
- **RabbitMQ** for low-latency inter-service RPC and complex routing in legacy parts.
- **SQS** for background tasks (email queue, image processing queue), because nobody wants to run more clusters than necessary.

All three coexist because they solve different problems. The mistake is assuming any one of them covers everything.

## Architect's takeaway

- **Kafka = log + replay + streams.** Default for events.
- **RabbitMQ = routing-rich, low-latency broker.** Default when routing is the hard part.
- **SQS = "I don't want to run anything".** Default for background tasks in AWS shops.
- **Don't fight the tool.** Forcing Kafka to do RabbitMQ-style per-message routing is misery; forcing RabbitMQ to do Kafka-scale replay is worse.
- **Pick by access pattern, not by hype.**
