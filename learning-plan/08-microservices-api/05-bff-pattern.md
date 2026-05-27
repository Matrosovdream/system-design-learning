# Example 05 — Backend for Frontend (BFF): a tailored API per client

The **Backend for Frontend** pattern: a dedicated backend that serves **one specific client type** (mobile, web, partner) by aggregating multiple downstream services into the exact shape that client needs.

Coined by Sam Newman; popularized by SoundCloud.

## The problem BFF solves

Without a BFF:

```
[mobile app]                       [microservices]
[web app]      → [API gateway] →   [users, orders, payments, ...]
[partners]
```

All clients hit the same API. Each client has different needs:
- **Mobile**: limited bandwidth, wants minimal payloads, has very different screens than web.
- **Web**: more data per page, less concerned about size.
- **Partners**: stable contract, slower change cadence.

You either:
- **Make a "universal" API** — bloated, compromise for everyone.
- **Add 17 query parameters** to every endpoint to filter what each client wants — unreadable mess.
- **Over-fetch on the client** — wasted bandwidth, slow renders.

## The BFF design

One BFF per client type:

```
[mobile app]   → [Mobile BFF]   ──┐
[web app]      → [Web BFF]      ──┼──► [users, orders, payments, ...]
[partners]     → [Partner API]  ──┘
```

Each BFF is **owned by the team that builds that client**. Mobile team owns the mobile BFF; web team owns the web BFF.

Each BFF:
- Aggregates downstream services.
- Returns exactly the shape the client wants.
- Can use the optimal API style for the client (GraphQL for web, REST for mobile, or vice versa).
- Evolves at the client's pace.

## What goes in a BFF

- **Aggregation**: combine multiple service calls into one client response.
- **Filtering / projection**: drop fields the client doesn't need.
- **Format adaptation**: REST → GraphQL or vice versa; convert legacy formats.
- **Client-specific logic**: pagination cursors, mobile-friendly image URLs, cached static menus.
- **Client-specific auth**: short-lived mobile tokens, OAuth flows.

## What does NOT go in a BFF

- **Cross-cutting concerns** like rate limiting, basic auth, TLS — those belong in the API gateway above the BFF.
- **Business rules / domain logic** — those belong in the domain services. The BFF is a presentation layer.
- **Generalized API** — if you find yourself making the BFF "support everything", you've made another gateway. Each BFF should be opinionated for its client.

## A concrete example: mobile app dashboard

The mobile app's dashboard screen needs:
- Username + avatar.
- 5 most recent orders.
- 3 unread notifications.
- Loyalty points balance.

Without BFF (mobile fetches each):
```
GET /api/users/me
GET /api/orders?user=me&limit=5
GET /api/notifications?user=me&unread=true&limit=3
GET /api/loyalty/balance?user=me
```

4 sequential requests over flaky mobile network. Total latency = sum of round-trips. Each request payload includes fields the dashboard doesn't show.

With BFF:
```
GET /m/dashboard
→ Mobile BFF in parallel:
   - user-service.get(me)
   - order-service.list_recent(me, 5)
   - notification-service.unread(me, 3)
   - loyalty-service.balance(me)
→ Combine, filter, return:
   {
     "user": { "name": "Alice", "avatar_url": "..." },
     "orders": [ { "id": ..., "summary": ..., "status": ... } ],
     "notifications": [ ... ],
     "points": 1234
   }
```

One call. Parallel fan-out. Tiny payload. Mobile happiness.

## BFF + GraphQL is a common combo

A common deployment: **GraphQL as the BFF for mobile/web**, REST/gRPC for downstream services.

GraphQL's "client describes what it wants" model is perfect for a BFF — the same GraphQL endpoint serves both web (more fields) and mobile (fewer fields), each requesting only what it shows.

## When to add a BFF

- **Multiple distinct clients** with different needs (especially mobile vs web).
- **Mobile is bandwidth-constrained** and needs payload trimming.
- **Frontend teams want autonomy** in shaping their API without negotiating with every backend team.
- **Aggregation** is becoming a bottleneck — clients making 5+ calls per screen.

## When NOT to add a BFF

- **You have one client.** Just make the API the way that client wants.
- **Your backend already does aggregation** in the right shape (monolith with templated views).
- **Your team is small.** Adding a BFF adds an extra service to operate.

## Pitfall: the BFF becomes a fat monolith

Without discipline, a BFF accretes:
- "Just add this little business rule here..."
- "We need to deduplicate orders here..."
- "Let's compute the loyalty tier here..."

Six months later, the BFF has half the business logic of the system. The next BFF (for partner API) duplicates it.

**Discipline**: BFFs are **thin** layers that **adapt** existing service capabilities for one client. They don't own domain logic.

## Pitfall: too many BFFs

A BFF per platform is reasonable: mobile, web, partner = 3 BFFs.

A BFF per screen is over-engineered: dashboard-BFF, profile-BFF, checkout-BFF = bad.

## A real-world architecture

```
Mobile app  → Mobile BFF (GraphQL) ─┐
Web app     → Web BFF (GraphQL)     ├──► [API gateway]
Partner     → Partner REST API     ─┘     │
                                          ▼
                                   [users, orders, payments, notifications, loyalty]
                                                (REST or gRPC)
```

Each BFF is a small Go/Node/PHP service. Each is owned by the frontend team that consumes it.

## Architect's takeaway

- **BFF = one backend per client type.** Mobile BFF, Web BFF, Partner API.
- **Owned by the client team** — same team writes BFF and client code.
- **BFFs aggregate, project, adapt** — they don't own domain logic.
- **Adds operational complexity** — one more service to deploy and monitor.
- **GraphQL + BFF** is a powerful combo for SPAs and mobile apps.
- **Don't make BFFs per screen** — that's over-engineering. Per client type is the right granularity.
