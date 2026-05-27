# Example 01 — CQRS: separating commands from queries

CQRS = **Command Query Responsibility Segregation**. The idea: the model for **changing** data (commands) is structurally different from the model for **reading** data (queries). Use different models, optimized differently, and possibly stored in different places.

## The traditional, non-CQRS model

One model serves both reads and writes:

```
[client] → [single API] → [single DB / model]
              ↓               ↓
            write             read
```

The same `Order` entity is used to create, update, and display orders. Same SQL table, same ORM model, same validation rules.

For most apps, **this is fine**. Don't reach for CQRS without a real reason.

## The CQRS model

Reads and writes are separate paths:

```
[client]
   │  write
   ▼
[Command API] ──► [Write Model: optimized for transactions]
                       │
                       │ (replicate / sync)
                       ▼
[client]            [Read Model: optimized for queries]
   │  read              ▲
   ▼                    │
[Query API] ───────────┘
```

- **Write model**: normalized, ACID, validations, business rules. "Save an order with these items."
- **Read model**: denormalized, optimized for the queries the UI needs. "Give me the order list with item names, prices, and customer info."
- **Synchronization**: usually async — events from the write side update the read side.

## Why bother

### 1. Reads and writes have wildly different shapes

- Writes update one entity at a time. Validations, constraints, foreign keys.
- Reads need joined, aggregated, denormalized data for screens.

Forcing one model to serve both means either expensive read-time joins (slow) or **read-heavy denormalization on writes** (slow writes, complex code).

CQRS lets each be optimized for its job.

### 2. Read scaling and write scaling are independent

A read-heavy product (50:1 reads vs writes) can scale the read model massively (cache, replicas, denormalized stores) without touching the write side.

### 3. Multiple read models for different views

You can have:
- A normalized SQL read model for transactional reads.
- An Elasticsearch read model for search.
- A Redis read model for hot lookups.
- A BigQuery read model for analytics.

All driven by the same events from the write side.

### 4. Read model can use a completely different DB

Write to Postgres (ACID, transactions). Read from Elasticsearch (search), Redis (hot cache), ClickHouse (analytics). All updated by listening to events from the write side.

## CQRS without event sourcing

A common confusion: CQRS does NOT require event sourcing. They're independent ideas.

You can do CQRS by:
- Writing to Postgres.
- After the write commits, **publishing an event** ("OrderCreated").
- Consumers update the read models.

The "write store" is just a regular RDBMS. The events are the **synchronization** mechanism. Storage is conventional.

This is the common CQRS variant in practice. Event sourcing adds further complexity that you don't need just to get CQRS.

## CQRS with event sourcing

If you also use event sourcing (next example), the write side is itself the log of events. The read models are projections built by replaying events. Powerful but more complex.

## A worked example

E-commerce orders. The product team needs three views:

1. **Order detail page**: order header, items, customer, shipping address, payment status.
2. **Order list page** (per customer): summary of recent orders.
3. **Analytics dashboard**: orders per region per hour.

### Without CQRS

One Postgres DB. The order-detail page joins 5 tables. The list page filters and aggregates. The analytics query scans millions of rows.

All compete for the same DB. Analytics queries slow down user-facing pages. You add read replicas; you partially fix this but the JOIN-heavy queries are still slow.

### With CQRS

Write side:
- Postgres for orders, items, customers (normalized, ACID).
- API accepts `CreateOrder`, `UpdateOrderStatus`, etc. — these are **commands**.
- On commit, publish events ("OrderCreated", "OrderStatusChanged") to Kafka.

Read side:
- "Order details" view: denormalized JSON document per order in a NoSQL store (or materialized view in Postgres).
- "Order list" view: pre-aggregated rows in Redis sorted sets.
- "Analytics" view: rolling aggregates in ClickHouse.

Each consumer subscribes to the events and updates its own view.

Result:
- Order detail page: one document lookup, <5 ms.
- Order list: O(1) read from sorted set.
- Analytics: separate engine, doesn't impact prod.

## The cost of CQRS

### Eventual consistency

The user writes an order. The order list page reads from the read model. The read model might be **0-500 ms behind**. The user might see a brief "empty" cart after their write.

You mitigate with read-your-writes (step 05) or by reading from the write model briefly after a user's write.

### More moving parts

You now have:
- A write store (Postgres).
- An event bus (Kafka).
- One or more read stores.
- Consumers that keep read models updated.

That's more services, more operational surface, more places for things to go wrong.

### Reprocessing / fixing the read model

If a consumer has a bug, the read model is wrong. You need to **rebuild it** — usually by reprocessing all events from the start (or a snapshot).

This is operational work. With event sourcing it's straightforward; with conventional CQRS-via-events, you might need to dump the write store and re-emit synthetic events.

## When to apply CQRS

- **Read and write models genuinely differ** in shape, scale, or access patterns.
- **Read scaling is hard** to achieve with a single model.
- **Multiple read views** (each optimized differently) are valuable.
- **The team is mature enough** to operate the additional infrastructure.

## When NOT to apply CQRS

- **Simple CRUD app.** The two models are basically the same. Forget it.
- **Strict consistency required everywhere** (e.g., financial ledgers — but then you can apply CQRS to non-critical views while keeping the core consistent).
- **Small scale.** The added complexity isn't worth it.
- **Team is new to distributed systems.** Eventual consistency, debugging async event flows is harder than it looks.

## A common pattern: CQRS for hot read paths only

You don't have to apply CQRS to every entity. A common pragmatic split:

- **Most entities**: use a normal model, both reads and writes hit the same DB.
- **Specific high-read endpoints** (feed, search, leaderboard): build a separate read model fed by events.

Only the endpoints that need it pay the complexity tax.

## Architect's takeaway

- **CQRS = separate the read and write models** for different optimization goals.
- **CQRS does NOT require event sourcing** — they're independent patterns.
- **Eventual consistency is the cost.** Mitigate with read-your-writes routing.
- **Apply CQRS selectively** — to the specific endpoints that need it, not everywhere.
- **Use multiple read models** for different query shapes (transactional, search, analytics, hot cache).
- **Don't reach for CQRS** until you have a real read/write divergence problem.
