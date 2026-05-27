# Example 05 — Dead-letter queue (DLQ): quarantine for the unprocessable

Some messages will never succeed. Bad data, missing dependency, bug in handler. If you keep retrying, you block all other messages behind them. The DLQ is the fix.

## The problem without a DLQ

```
queue: [msg1: bad, msg2: ok, msg3: ok, msg4: ok, ...]

consumer:
  receive msg1 → process → fail → retry
  receive msg1 → process → fail → retry
  receive msg1 → process → fail → retry
  ... forever
```

Even worse: most queues guarantee in-order processing **per partition**. The bad message blocks every other message in the same partition. Head-of-line blocking.

## What a DLQ does

After N retries, the broker (or consumer) **gives up** and moves the message to a separate queue: the **dead-letter queue**. The main queue continues processing the rest.

```
queue:        [msg1, msg2, msg3, ...]

  msg1 fails 5 times →

queue:        [msg2, msg3, ...]   ← unblocked
DLQ:          [msg1]               ← quarantined for human attention
```

## When messages end up in the DLQ

1. **Retry exhaustion** — N failed attempts.
2. **Message TTL expired** without being acked.
3. **Explicit `reject` with `requeue=false`** — your code knows this can't be retried.
4. **Schema/format errors** — message can't be deserialized.
5. **Permission errors** — the consumer can't reach its dependencies (DB, S3, downstream API).

## How DLQs work in each broker

### RabbitMQ

You declare a queue with `x-dead-letter-exchange` and `x-dead-letter-routing-key`. Messages that are negatively acked (`basic.nack`) or expire get routed to the configured exchange/queue.

### Kafka

No native DLQ. The pattern: your consumer wraps processing in try/catch and explicitly produces failed messages to a separate `*.dlq` topic before committing offset. Frameworks like Spring Cloud Stream, Kafka Connect, and Flink have this built-in.

### AWS SQS

Set a `RedrivePolicy` on the main queue with `maxReceiveCount` and a `deadLetterTargetArn`. After N receive-without-ack cycles, SQS moves it to the DLQ.

### Pub/Sub services (SNS, Google Pub/Sub)

Most managed pub/sub services have DLQ topics built in.

## How to handle messages in the DLQ

The DLQ is **not** "garbage". It's "needs investigation".

### Standard playbook

1. **Alert** when the DLQ is non-empty. PagerDuty / Slack channel.
2. **Inspect** a sample. What error?
3. **Categorize**:
   - **Bug in handler**: fix code, replay DLQ → main queue.
   - **Bad data from producer**: fix producer, drop or replay DLQ.
   - **Transient external failure**: replay DLQ → main queue once the downstream is back.
   - **Permanently bad**: archive and drop.
4. **Replay** with care — the same messages might fail again if you haven't fixed root cause.

### DLQ replay pattern

```
[DLQ] → [DLQ consumer (manual trigger)] → [main queue]
```

Many teams build a tiny tool ("requeue 100 messages from DLQ to main") for this. Most cloud DLQs have a one-click "redrive to source" feature now.

## Important behaviors

### Retry backoff before DLQ

Don't retry immediately N times. Use exponential backoff between retries: 1s, 5s, 25s, 125s, ... Some failures are transient (DB blip) and need time to clear.

```
retry 1: after 1s
retry 2: after 5s
retry 3: after 25s
retry 4: after 125s
retry 5: after 625s → if still failing, DLQ
```

This gives transient failures room to recover and reduces load on a struggling downstream.

### Visibility / timeout

In SQS-style queues, a message taken by a consumer is **invisible** for a "visibility timeout". If the consumer doesn't ack in time, it becomes visible again (potentially to another consumer).

Set this carefully: too short → premature redelivery for slow handlers; too long → recovery from crashed consumer is slow.

### Distinguish "retryable" vs "non-retryable" errors

Not all failures should retry forever. A schema error in the message won't fix itself — send it to DLQ immediately. A timeout fetching a downstream service might fix itself — retry.

```python
def handle(msg):
    try:
        data = parse(msg)             # non-retryable on failure
    except SchemaError:
        send_to_dlq(msg, reason="schema")
        return ack(msg)

    try:
        process(data)                  # retryable on transient errors
    except TransientError:
        raise                          # let broker retry
    except PermanentError:
        send_to_dlq(msg, reason=str(e))
        return ack(msg)
```

## A real story: the silent DLQ

A team configured a DLQ in production but didn't alert on it. Six months later, an unrelated investigation revealed **2.3 million messages in the DLQ**. Customers had been quietly missing notifications for half a year.

The lesson: **a DLQ without monitoring is worse than no DLQ**, because it hides failures.

### Minimal DLQ monitoring

- Alert on DLQ depth > threshold (e.g., > 10 messages).
- Dashboard the DLQ rate (messages per minute entering DLQ).
- Sample inspection — a daily report of the most common DLQ reasons.
- Replay tooling — make it one command, not a research project.

## Architect's takeaway

- **Always configure a DLQ.** A queue without one will eventually wedge on a poison message.
- **Always monitor it.** Silent DLQs mean silent data loss.
- **Distinguish retryable from non-retryable** at the consumer. Schema errors → immediate DLQ. Transient errors → retry with backoff.
- **Build replay tooling early.** You will need it on day one of every incident.
- **DLQ depth is a leading indicator** of system health. Watch it.
