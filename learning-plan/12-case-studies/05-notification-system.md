# Case Study 05 — Design a notification system

A backend that delivers notifications to users via email, SMS, push (APNs/FCM), and in-app channels. The challenges are throughput, reliability, scheduling, multi-channel routing, and user preferences.

## Problem statement

Build a service that:
- Accepts notification requests from many internal services.
- Routes each notification to the appropriate channel(s): email, SMS, push, in-app.
- Honors user preferences (Do Not Disturb, channel opt-outs).
- Schedules delivery (immediate or future).
- Retries failures.
- Tracks delivery and engagement.

## Clarifying questions

1. **Scale**: notifications/day, peak rate.
2. **Channels**: email, SMS, push only? Slack, Discord, in-app?
3. **Templates**: predefined or fully dynamic?
4. **Localization**: multiple languages?
5. **Audience**: 1-on-1 or broadcast (millions in one go)?
6. **Latency**: real-time (within seconds) or batch?
7. **Engagement tracking**: opens, clicks, conversions?

**Assumed answers:**

- 1B notifications/day across all channels.
- Channels: email, SMS, push (iOS APNs, Android FCM), in-app.
- Predefined templates with variables.
- Multi-language.
- Both 1-on-1 (most) and broadcast (occasional, millions per blast).
- Mix of real-time and scheduled.
- Track delivery, opens, clicks.

## Functional requirements

- Producer APIs: any internal service can request a notification.
- Multi-channel delivery based on user preferences.
- Templating + localization.
- Scheduling and rate-limiting per channel/provider.
- Retries with backoff.
- Engagement tracking.

## Non-functional requirements

- p95 send latency < 5 seconds for real-time.
- 99.99% delivery rate (excluding hard-bounced or invalid recipients).
- Scale to 1B+/day, with 10× spikes for broadcasts.
- Idempotent — duplicates aren't sent.

## Capacity estimation

```
1B/day ≈ 12k/sec average.
Peak (broadcast spikes): 100k+/sec.

Per notification: ~1-5 KB metadata + template variables.
Daily volume: ~5 TB metadata; ignore email body in app store (handled by SendGrid).

External provider quotas:
- SendGrid: ~10k/sec out of the box; can buy more.
- Twilio SMS: ~100/sec per phone number; scales with multiple numbers.
- APNs: high throughput; managed by Apple.
- FCM: high throughput; managed by Google.
```

## API design

Producers post a notification request:

```
POST /api/v1/notifications
{
  "user_id": "abc-123",
  "template": "order_shipped",
  "variables": { "order_id": "X", "tracking_url": "..." },
  "channels": ["email", "push"],   // optional; can use user prefs
  "scheduled_at": null,             // null = now; or future timestamp
  "idempotency_key": "...",
  "priority": "high" | "normal" | "low"
}
→ 202 Accepted
   { "notification_id": "..." }
```

Read API for tracking:

```
GET /api/v1/notifications/{id}/status
→ { status, channels: [{ channel, delivered_at, opened_at, ... }] }
```

Preferences API:

```
GET  /api/v1/users/{id}/preferences
PUT  /api/v1/users/{id}/preferences
{
  "email": { "opted_in": true, "categories": ["transactional", "promotional"] },
  "sms":   { "opted_in": false },
  "push":  { "opted_in": true, "quiet_hours": "22:00-08:00" },
  "in_app": { "opted_in": true }
}
```

## High-level architecture

```
[internal services]
        │
        ▼
[Notification API]
        │
        ▼
[Kafka: notifications.requested]
        │
        ▼
[Resolver service]
   ├─► load user preferences
   ├─► render templates
   ├─► determine channels
   ├─► check rate limits / quiet hours
   ↓
[Kafka: per-channel topics]
   email.outbound
   sms.outbound
   push.outbound
   in_app.outbound
        │
        ▼
   [Channel workers]
        ├─► email worker → SendGrid / SES
        ├─► sms worker → Twilio
        ├─► push worker → APNs / FCM
        └─► in_app worker → write to user inbox
        ↓
   [delivery results → Kafka: notifications.events]
        ↓
   [Tracker service: persist delivery + engagement]
```

## Deep dive: resolver service

The resolver decides:
- **Channels**: based on user prefs + producer's request.
- **Templates**: load by name + locale; substitute variables.
- **Rate limits / quiet hours**: e.g., don't push at 3am.

```
on event "notifications.requested":
  if has_seen(event.idempotency_key): drop (idempotency)
  prefs = load_preferences(event.user_id)
  for channel in requested_channels:
    if not prefs[channel].opted_in: skip
    if quiet_hours_now(prefs[channel]): defer to end of quiet hours
    template = templates.load(event.template, prefs.locale, channel)
    body = render(template, event.variables)
    publish to channel-specific Kafka topic with payload
```

Idempotency is critical: an upstream service might retry; we shouldn't send the email twice. Use the producer's `idempotency_key` to dedupe (Redis with 24h TTL).

## Deep dive: channel workers

Each channel is a pool of workers consuming from a dedicated Kafka topic.

### Email worker

```
on email.outbound:
  call SendGrid API with email body, idempotency_key
  on success: publish notifications.events { status: "delivered", channel: "email", ... }
  on failure: retry (exponential backoff); after N: DLQ
```

Use `X-Smtpapi` headers (SendGrid) to track per-email metadata. Webhooks from SendGrid → events for opens, clicks, bounces.

### Push worker

```
on push.outbound:
  lookup user's device tokens
  for each token:
    call APNs/FCM with the message
    on success: ack
    on invalid token: mark device inactive (don't try again)
```

Push has unique failure modes — `invalid token`, `not registered`, etc. Track these per device to keep token lists clean.

### SMS worker

```
on sms.outbound:
  rate-limit per phone number / per country
  call Twilio API
  on success: ack
  on failure: retry
```

Twilio enforces strict rate limits per "from" number; you may need multiple numbers (with messaging-service routing) for high throughput.

## Deep dive: scheduling

Requests with `scheduled_at` in the future shouldn't be processed immediately.

Approach 1: separate "scheduled" store

```
- API writes to scheduled_notifications table with scheduled_at.
- A scheduler job polls for "scheduled_at <= now", removes from table, publishes to Kafka.
```

Approach 2: delay queue

Kafka doesn't natively delay messages. Use a separate delay queue (SQS supports delays up to 15 min; for longer, use a TimescaleDB / specialized store).

Approach 3: bucket by hour

Store scheduled notifications in hourly buckets in DynamoDB; a worker reads the current hour and dispatches due ones.

For most: Approach 1 (DB + polling) is the simplest scalable solution.

## Deep dive: broadcasts

A broadcast: "send notification N to all 100M users".

Two ways:

### Approach 1: explode upstream

Producer sends 100M individual requests. Kafka and workers handle them. Each is independent.

This works but:
- Producer generates 100M Kafka writes — significant.
- The fan-out is at the wrong layer.

### Approach 2: dedicated broadcast service

Producer sends one "broadcast" request with criteria ("all users in region X"). A broadcast service:

```
- queries the audience: SELECT user_id FROM users WHERE ...
- writes to a "broadcast jobs" table with chunks of users
- worker processes chunks: for each user, publishes a regular notification event
- this is fan-out on read (broadcast time), one source of truth
```

Includes rate-limiting (don't blast 100M push notifications in 1 minute — APNs would rate-limit you anyway).

## Deep dive: engagement tracking

Per notification: `created`, `sent`, `delivered`, `opened`, `clicked`, `bounced`.

```
notifications_log
  notification_id, user_id, channel, status, created_at, ...
```

Events from delivery providers (SendGrid webhooks, APNs feedback) update this row. Used for:
- Customer support ("did Alice's order email get delivered?").
- Reporting ("our email open rate is 22%").
- Auto-suppression ("if user hard-bounces 3 emails, mark email channel inactive").

## Trade-offs discussion

### Why Kafka, not RabbitMQ?

- Need replay (if a worker has a bug, replay events from yesterday).
- High throughput.
- Multiple consumers per topic (one for delivery, one for engagement tracking, one for analytics).

RabbitMQ works for smaller scale; once you need replay or 100k+ events/sec, Kafka wins.

### Why separate channels into separate topics?

Different scaling profiles: SendGrid handles 10k+/sec; Twilio is rate-limited per number. Keeping them in separate topics lets workers scale and throttle independently.

### Why an external email/SMS provider?

Don't build SMTP from scratch. Email/SMS deliverability is its own art (SPF, DKIM, IP reputation, carrier relationships). Use SendGrid, SES, Twilio, etc.

### Idempotency: where does it live?

In the resolver (entry point). Each request has an `idempotency_key`; resolver dedupes via Redis. Cheaper than checking at every downstream step.

### Preferences: real-time or cached?

Preferences cached aggressively (~5-min TTL). Stale prefs = slightly delayed opt-out, acceptable. On preference update, flush the user's cache key.

## Common follow-up questions

1. **"What if SendGrid is down?"**
   Circuit breaker on the email worker. While open, messages back up in Kafka topic; once SendGrid recovers, drain. For sustained outage, switch to a secondary provider (SES) via dynamic config.

2. **"How do you handle bounce / unsubscribe?"**
   Bounces from SendGrid → mark email channel inactive for that user. Unsubscribe link in every email → update preferences when clicked. Both feed back to the preferences store.

3. **"How do you ensure messages aren't sent during quiet hours but aren't lost?"**
   The resolver defers them: writes to scheduled store with `scheduled_at = quiet_hours_end`. Scheduler picks them up at the right time.

4. **"What about A/B testing notification copy?"**
   Resolver supports template variants. A/B service decides variant per user; resolver picks the right template. Engagement metrics tied back to variant.

5. **"What's the cost?"**
   Email via SendGrid: ~$0.0001/email at scale. 1B emails/day = ~$100k/day. Real money.
   SMS via Twilio: ~$0.0075/SMS in US. 100M SMS/day = $750k/day. Expensive.
   These costs drive design decisions — push first, email second, SMS only for critical alerts.

6. **"How do you prevent flooding the user?"**
   Per-user, per-day rate limit (e.g., "max 10 promotional emails per week"). Stored as Redis counters. Resolver checks before publishing.

7. **"How do you handle the broadcast of 100M push notifications?"**
   Throttle to a few thousand per second. APNs/FCM are fine with this. Spread over many seconds. Use the broadcast service approach.

## Key takeaways

- **Channel-specific workers + per-channel topics** = independent scaling.
- **Idempotency at the entry point** prevents duplicates.
- **Use proven providers** (SendGrid, Twilio, APNs, FCM) — don't build SMTP.
- **Preferences and rate limits** are first-class citizens; don't bolt on later.
- **Engagement tracking via provider webhooks** + your own event log.
- **Broadcasts via dedicated path** to avoid swamping the normal flow.
- The system is mostly a **flexible routing + retries + observability** problem.
