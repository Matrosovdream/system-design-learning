# Case Study 03 — Design a real-time chat system (WhatsApp / Slack DMs)

The canonical real-time problem. Probes your understanding of long-lived connections, message delivery guarantees, presence, and scaling stateful servers.

## Problem statement

Build a chat system supporting:
- 1-on-1 and group messaging.
- Real-time delivery (messages appear within ~1 second).
- Message persistence (history across devices, app reinstalls).
- Read receipts, typing indicators, online/offline status.
- Multiple devices per user.

## Clarifying questions

1. **Scale**: total users, daily active, messages/day, peak concurrent connections.
2. **Group sizes**: small chats vs huge channels (Slack channels can be 10k+).
3. **Media**: just text or also images/files?
4. **Delivery guarantees**: at-most-once, at-least-once, exactly-once?
5. **End-to-end encryption?**
6. **History**: retain forever or N days?
7. **Search**: server-side search of messages?

**Assumed answers:**

- 1B total users, 500M DAU, 50B messages/day, 50M peak concurrent connections.
- 1-on-1 and groups up to 256 members (WhatsApp-style).
- Text + media.
- At-least-once delivery, dedup on client by message ID.
- E2E encryption out of scope for this design (huge separate topic).
- Forever retention.
- Server-side search out of scope.

## Functional requirements

- Send/receive messages in real time.
- Group conversations.
- Message ordering preserved per conversation.
- Read receipts.
- Online/offline/last-seen status.
- Multi-device sync.

## Non-functional requirements

- p95 end-to-end delivery latency < 1 second.
- 99.99% uptime.
- Messages durable (no loss).
- Scale to millions of concurrent connections.

## Capacity estimation

```
50M concurrent connections.
  - One server handles ~64K concurrent WebSocket connections (TCP file-descriptor limits, memory).
  - Need ~800 connection-handling servers (with headroom: 1000-2000).

50B messages/day = ~580K messages/sec average.
  Peak (3x): ~1.7M messages/sec.

Storage per year:
  50B × 365 × ~100 bytes/msg = ~1.8 PB/year of text messages.
  Plus media: ~10 PB+/year (in S3-like).
```

## API design

REST for user/auth/group management; WebSocket for real-time messaging.

### REST

```
POST   /api/v1/conversations            create chat/group
GET    /api/v1/conversations            list user's chats
GET    /api/v1/conversations/{id}/messages?before=…&limit=50    paged history
POST   /api/v1/conversations/{id}/members    add to group
DELETE /api/v1/conversations/{id}/members/{user_id}
```

### WebSocket

```
client → server:
  { type: "send", conv_id, client_msg_id, body }
  { type: "read", conv_id, up_to_msg_id }
  { type: "typing", conv_id }
  { type: "presence", status }

server → client:
  { type: "message", conv_id, msg_id, from, body, ts }
  { type: "read_receipt", conv_id, user_id, up_to_msg_id }
  { type: "typing", conv_id, user_id }
  { type: "presence", user_id, status }
  { type: "ack", client_msg_id, msg_id, ts }
```

## High-level architecture

```
[mobile/web]
   │
   │ WSS (TLS-encrypted WebSocket)
   ▼
[LB: maintains stickiness on connection]
   │
   ▼
[Chat servers — 1000+]    ← stateful: own the open WebSockets
   │
   │ publish/consume events
   ▼
[Kafka: messages topic, presence topic, receipts topic]
   │
   ├─► [Message store: Cassandra/ScyllaDB] (writes)
   │
   ├─► [Presence service: Redis] (user → online, last_seen)
   │
   └─► [Push notification service] (for offline users → APNs/FCM)
```

## Data model

### Messages: write-heavy, time-ordered per conversation

Cassandra/ScyllaDB schema:

```sql
CREATE TABLE messages (
    conversation_id  UUID,
    message_id       TIMEUUID,     -- timestamp-ordered UUID
    sender_id        UUID,
    body             TEXT,
    created_at       TIMESTAMP,
    PRIMARY KEY (conversation_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
```

Partition key = `conversation_id`. All messages for one chat are co-located. Sort by `message_id` descending for "fetch recent messages" queries.

### Conversations metadata

```sql
CREATE TABLE conversations (
    id          UUID,
    type        TEXT,    -- "direct" or "group"
    name        TEXT,    -- for groups
    created_at  TIMESTAMP
);

CREATE TABLE conversation_members (
    conversation_id UUID,
    user_id         UUID,
    joined_at       TIMESTAMP,
    last_read_msg_id TIMEUUID,
    PRIMARY KEY (conversation_id, user_id)
);

CREATE INDEX ON conversation_members (user_id);    -- "what chats am I in?"
```

For Slack-style channels with 10k members, the `conversation_members` table is large but reasonable. For groups of 256, trivially small.

### Presence: Redis

```
key: presence:user:42
value: { status: "online", last_seen: 2026-05-27T14:23:01Z }
TTL: 60 seconds (refreshed by client heartbeat)
```

When TTL expires (client disconnected), user is offline. `last_seen` captured before expiry.

## Deep dive: connection management

50M concurrent connections need ~1000-2000 chat servers. Each server holds a slice of users' WebSockets.

When user U connects:
1. LB picks a chat server S (with consistent hash on user_id, so reconnects land on same S).
2. S authenticates U.
3. S registers in `routing_table`: `user_id → server_address`. Stored in Redis or similar.
4. S subscribes to Kafka topics for U's conversations (to receive messages).

When messaging:

```
User Alice (on server S1) sends message to conversation C:
  S1 receives WS message.
  S1 assigns server-generated msg_id (timestamp + node ID).
  S1 inserts into Cassandra.
  S1 publishes to Kafka topic for conversation C.
  S1 returns ack to Alice's client.
  
  All servers holding members of conversation C (S2, S5, S37, ...) consume the Kafka message.
  Each sends it down the WebSocket to the connected user.

For offline users: a dedicated consumer reads from Kafka, looks up devices, sends to APNs/FCM (push notifications).
```

## Deep dive: message ordering

Within one conversation, all messages must appear in the same order to all participants.

This is achieved by:
- Server-generated `message_id` is a **timestamp-ordered UUID** (TIMEUUID). One sender's messages are monotonic.
- All messages for one conversation go through one Kafka partition (key = conversation_id).
- All consumers see them in the same order.
- Clients display by `message_id` order.

Even if messages from two senders interleave, all clients see the same order (because they see the same Kafka stream).

## Deep dive: read receipts

When user reads:

```
client → server:
  { type: "read", conv_id, up_to_msg_id }

server:
  UPDATE conversation_members SET last_read_msg_id = ? WHERE conversation_id = ? AND user_id = ?
  Publish to Kafka: { type: "read_receipt", conv_id, user_id, up_to_msg_id }
```

All other members of the conversation receive the read receipt and update their UI ("seen by Alice at 14:23").

## Deep dive: offline message delivery

User is offline (WebSocket disconnected). Messages still arrive in Kafka.

Two paths:

### Path 1: Push notification (via APNs/FCM)

A dedicated consumer reads messages, looks up the recipient's device tokens, sends a push.

### Path 2: On reconnect, fetch missed messages

When the user reconnects:
- Client knows its `last_received_msg_id`.
- Server: `SELECT * FROM messages WHERE conversation_id = ? AND message_id > ?`.
- Stream all missed messages.

Combined, users see messages live when connected, push when not, and sync up on reconnection.

## Deep dive: multi-device sync

A user has phone + desktop. Both have open WebSockets.

- Both register in the routing table; the message broker delivers to both.
- Read receipts and message status sync via Kafka (each device sees the events).
- "Already read on phone" → desktop unread badge clears.

The model: a user has multiple connections; treat them as siblings; all see the same stream.

## Deep dive: presence at scale

Presence is **expensive** to broadcast naively. If user U has 1000 friends and U comes online, 1000 messages must go out.

Mitigations:
- **Presence is pull-based** when a user opens a chat — query "is X online right now?".
- **Push only to active subscribers**: notify online users in the same conversation, not all friends.
- **Batched updates**: a presence service aggregates changes; broadcasts every few seconds, not on every event.
- **TTL on heartbeats**: client sends a ping every 30s; if no ping for 60s, mark offline. Reduces actual events.

## Trade-offs discussion

### WebSocket vs SSE vs long polling

WebSocket: bidirectional, low overhead, ideal. SSE (server-sent events): one-way, simpler, no client-to-server stream over the same connection. Long polling: workable, higher overhead, used as a fallback.

WhatsApp / Slack use WebSocket (or proprietary). Modern browser + mobile = WebSocket.

### Sticky load balancing

The LB needs to route a user's reconnect to the same chat server (so the in-memory state matches). Consistent hashing on `user_id` does this. Alternatively, store all state in Redis so any server can pick up — but in-memory caching is faster.

### Cassandra vs Postgres for messages

Messages are append-heavy time-series-shaped. Cassandra (or ScyllaDB) handles this brilliantly. Postgres would struggle at 1M+ writes/sec without serious effort.

For smaller-scale chat (hundreds of thousands of users), Postgres with `(conversation_id, created_at)` index works fine.

### E2E encryption

Outside this design. WhatsApp's Signal protocol: keys per device pair, server stores ciphertext only. Adds key management complexity; server can't search messages.

### Kafka partitioning for fairness

Use `conversation_id` as the partition key. All messages of one conversation go in order. Different conversations parallelize across partitions for throughput.

## Common follow-up questions

1. **"How do you handle a group with 10k members?"**
   Same architecture, but fan-out cost grows. Some chats become "broadcast-only" (channel-style — most members are passive). Optimize: don't push live to inactive members; rely on lazy fetch on open.

2. **"What happens during a chat server crash?"**
   Connected users' WSs disconnect. Clients reconnect to a new server (via LB). New server queries Cassandra/Redis for missed messages. Brief delivery delay; no data loss because Cassandra is the source of truth.

3. **"How would you build typing indicators?"**
   Lightweight Kafka topic with high churn, short retention. Or even Redis Pub/Sub for ephemeral events. Don't persist; only deliver to currently-connected recipients in the same conversation.

4. **"How does the unread badge count work?"**
   Maintained per (user, conversation) in `conversation_members.last_read_msg_id`. Unread = count of messages with `message_id > last_read_msg_id`. Update on each `read` event; computed lazily on client open.

5. **"What if Kafka is down?"**
   Use Kafka in HA cluster mode. If a complete outage: messages still write to Cassandra; delivery is delayed; users see them on next refresh. Don't lose data.

6. **"How do you do message search?"**
   Stream messages to Elasticsearch (a separate consumer of the same Kafka stream). Search service uses ES. Update lag: seconds.

7. **"How does mobile push notification work for offline users?"**
   Recipient's device registers FCM/APNs token at login. A dedicated push consumer reads messages for offline users, sends to FCM/APNs. Phone wakes, displays notification. Tap → open app → reconnects and syncs.

## Key takeaways

- **WebSocket + sticky routing** for real-time delivery.
- **Kafka as the message bus** between chat servers and downstream consumers.
- **Cassandra/ScyllaDB for messages** (append-heavy, time-series shape).
- **Redis for presence and routing** (small, hot, ephemeral state).
- **Message ordering preserved via partition key + TIMEUUID**.
- **Offline delivery via push + reconnection catch-up.**
- **The real hard problems**: connection state, presence at scale, multi-device sync, group fan-out.
