# Example 05 — Stateless services: the precondition for horizontal scale

A service is **stateless** when any instance can handle any request without prior context. State lives in a shared store (DB, cache, queue) — not in the process's local memory or disk.

This single property is what lets you scale horizontally.

## What "state" means here

**State** = anything one request leaves behind that the next request needs:
- User session ("which user am I?").
- Shopping cart contents.
- Multi-step wizard progress.
- In-progress upload chunks.
- Rate-limiter counters.
- Cached database rows local to that process.

## Stateful vs stateless: same feature, two designs

### Stateful (PHP example, in-memory session)

```php
session_start();
$_SESSION['cart'][] = $productId;  // stored in /var/lib/php/sessions on THIS server
```

If the load balancer routes the next request to a different PHP server, `$_SESSION['cart']` is empty. The user's cart "disappeared".

**Fix with sticky sessions:** LB pins the user to one server via cookie. Works, but:
- That server is now a SPOF for that user — if it crashes, cart is gone.
- Rolling deploys evict everyone's sessions.
- Capacity planning becomes per-user, not per-cluster.
- Auto-scaling fights stickiness.

### Stateless (PHP example, session in Redis)

```php
// Use Redis-backed session handler
ini_set('session.save_handler', 'redis');
ini_set('session.save_path', 'tcp://redis.internal:6379');

session_start();
$_SESSION['cart'][] = $productId;  // stored in Redis, visible to all servers
```

Any PHP server can now serve any request. Add 10 more servers? They all work immediately.

### Stateless (Go example, JWT instead of server-side session)

```go
// On login: server signs a JWT containing user_id and minimal claims
token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims).Sign(secret)

// On each request: server validates the token, no DB lookup needed for auth
parsed, err := jwt.Parse(tokenString, keyFunc)
userID := parsed.Claims.(jwt.MapClaims)["user_id"]
```

The server stores **zero session state**. The user carries it (cryptographically signed) in the JWT. Any server can validate it.

Trade-off: revocation is harder (you can't "delete" a JWT — it's valid until expiry).

## Where to put the state

| Type of state              | Where it goes                              |
|----------------------------|--------------------------------------------|
| Sessions                   | Redis / Memcached / signed JWT             |
| Shopping carts             | Redis (persistence on) or DB               |
| Auth tokens                | DB (revocable) or JWT (not revocable easily) |
| Uploaded file chunks       | Object store (S3) with multipart upload    |
| Rate limit counters        | Redis with TTL                             |
| Wizard / multi-step flows  | DB rows keyed by flow_id                   |
| Cached DB results          | Redis (shared) — not local memory!         |

The pattern: **push state out of the process** into something shared, durable, and itself scalable.

## What about caching that *is* local to a process?

In-process caches (e.g., Go's `sync.Map`, PHP-FPM opcache, an LRU in app memory) are fine as long as they're:
- **Best-effort** (a miss falls back to the shared store).
- **Not the source of truth** (no writes that only live in local memory).
- **Tolerable of inconsistency** between servers.

E.g., caching "the list of feature flags" in process memory for 30 seconds is fine — even if two servers briefly disagree, no user data is lost.

## The exception: legitimately stateful services

Some services **must** keep state. They just don't pretend otherwise:

- **Databases** (Postgres, MySQL, MongoDB) — state is their job.
- **Message queues** (Kafka, RabbitMQ) — they hold messages.
- **Caches** (Redis, Memcached) — they hold cached values.
- **WebSocket servers** — they hold open connections (the connection itself is state).

These scale horizontally too, but with **explicit data partitioning** (sharding, consistent hashing) rather than "any instance does anything". That's covered in step 05.

## Smoke test: is my service stateless?

Ask:
1. If I randomly killed any one instance, would any user's in-flight workflow break? *(other than retrying the current request)*
2. If I added 5 new instances right now, would they immediately serve traffic correctly?
3. If I rolled out a new version and replaced every instance one by one, would anything be lost?

If yes / yes / no → you're stateless. If any "no" / "yes" — there's state living in a process that shouldn't be there.

## Architect's takeaway

- **Stateless is the default for application services.** Anything else needs justification.
- **Sticky sessions are a smell.** They work, but they prevent every horizontal-scaling property you actually want.
- **Move state to Redis, the DB, the queue, or the client (JWT/cookie).** Then the app tier becomes interchangeable.
- **Stateful services exist**, but they pay for their state with explicit partitioning, replication, and operational care.
- The architect's job: keep the application tier stateless, push state into a small number of well-managed stateful systems.
