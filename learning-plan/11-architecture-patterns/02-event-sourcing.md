# Example 02 — Event sourcing: storing the history, not the state

Event sourcing flips the usual storage model. Instead of storing **current state** ("user X has 5 items in cart"), you store the **sequence of events** that produced it ("user X added item A; added item B; ...; removed item A; ..."). Current state is computed by replaying events.

It's a powerful, **niche** pattern. Used in banking, audit-critical systems, complex domains. **Misapplied frequently** because of its appeal.

## Traditional storage (state-based)

```sql
UPDATE cart SET items = '[item_b, item_c]' WHERE user_id = 42;
```

You see only the result. You don't know how the state got there. To know history, you need a separate audit log.

## Event sourcing

```sql
INSERT INTO events (stream_id, sequence, type, payload) VALUES
  (42, 1, 'ItemAdded', '{ "sku": "A", ... }'),
  (42, 2, 'ItemAdded', '{ "sku": "B", ... }'),
  (42, 3, 'ItemAdded', '{ "sku": "C", ... }'),
  (42, 4, 'ItemRemoved', '{ "sku": "A" }');
```

To know the current cart: replay events 1-4 → [B, C]. The events are the source of truth; state is derived.

## What event sourcing buys you

### 1. Complete audit trail

Every change is a recorded event. "Why is this number 47?" → replay events to see. Compliance gold, debugging gold.

### 2. Temporal queries

"What was the cart at 14:32 yesterday?" → replay events up to that timestamp. Without ES, you've thrown that information away.

### 3. New read models for free

Want a new view? Build a new projection that consumes the event stream from the beginning. The data has always been there.

### 4. Easy CQRS

The events are the natural way to feed read models. ES + CQRS is a popular combo.

### 5. Event-driven naturally

Your event log is exactly what other services want to react to. No separate change-data-capture mechanism needed.

## What event sourcing costs

### 1. Querying current state is expensive

To get the current cart, you replay events. For long streams, this is slow.

Mitigation: **snapshots** — periodically save a snapshot of state and only replay events since then.

### 2. Schema evolution is hard

Events are immutable history. If you change the shape of `ItemAdded`, you must still handle old events forever.

Strategies:
- **Versioned event types** (`ItemAddedV1`, `ItemAddedV2`).
- **Upcasters** — code that converts old events to new shapes on read.

This adds discipline and complexity over time.

### 3. Eventual consistency between event store and projections

Same as CQRS — the read model lags the writes.

### 4. Privacy / GDPR challenges

"Delete all my data" — but the events are immutable! You can't just delete user data.

Common workarounds:
- **Encrypt event payloads** with per-user keys; on delete, throw away the key. The events still exist but are unreadable.
- **Crypto-shredding**.
- **Compaction** (controversial — violates the "immutable" principle).

This is the most-cited reason teams regret event sourcing.

### 5. Steep learning curve

Engineers familiar with CRUD struggle with "the events are the truth". Modeling commands, events, projections takes mental effort.

## A worked example: bank account

State-based:
```
balance: $1000
```

Event-sourced:
```
events:
  AccountOpened(initial_balance=0)
  Deposit(100)
  Deposit(900)
  Withdraw(50)
  Deposit(50)

balance = replay(events) = $1000
```

Now suppose there's a dispute: "I didn't make this $50 withdrawal."

State-based: you have nothing. You file a manual investigation.

Event-sourced: full audit. You see exactly when and where it happened. You can show the user the event log. You can even **reverse** the event with a "WithdrawReversed" event, leaving history intact.

This is why **banks and accounting systems** are the canonical use case for event sourcing. Auditability is a hard requirement; the immutability matches accounting practice exactly.

## Snapshots

```
events: [e1, e2, ..., e1000]
snapshot at e500: state_at_e500
```

To get current state: load snapshot at e500, replay e501..e1000. Much faster than replaying all 1000.

Snapshots are an optimization — they don't change the principle that events are the truth.

## Building read models (projections)

```
project_cart_view(event):
  if event.type == "ItemAdded":
      cart_view.items.append(event.payload.sku)
  if event.type == "ItemRemoved":
      cart_view.items.remove(event.payload.sku)
  save(cart_view)
```

A projection is a consumer that maintains a derived view from the event stream. Multiple projections can exist for the same event stream, each optimized for its query.

If a projection has a bug, you delete it and rebuild — replay all events from the beginning into the fixed code.

## Tools

- **EventStoreDB** — purpose-built event store.
- **Axon** (Java) — full ES + CQRS framework.
- **Marten** (.NET) — ES on Postgres.
- **Kafka** — many use Kafka as an event log; it works but is not designed for per-aggregate streams.
- **Postgres with discipline** — many implementations are "just" an events table in Postgres + projection consumers.

## When event sourcing is right

- **Audit / compliance** is a hard requirement.
- **Temporal queries** are valuable to the business.
- **Domain is event-shaped naturally**: banking transactions, order workflows, inventory movements, IoT/telemetry streams.
- **Team has experience** with event-driven thinking.

## When event sourcing is wrong

- **CRUD app** that the user thinks of as forms with current state.
- **Strict data deletion requirements** (GDPR) with no path to crypto-shredding.
- **Strong existing model** in a relational DB that's working.
- **Small team, MVP**, no audit requirement.

The biggest mistake: applying event sourcing to **everything** in a system, when only a few domains genuinely need it. Use it for the bounded contexts that benefit.

## A pragmatic hybrid

Many production systems use event sourcing **for specific aggregates** (Account, Order, Inventory — places where audit matters) and regular CRUD for everything else (User profile, blog post drafts).

Don't make it ideological. Apply where it pays off.

## Architect's takeaway

- **Event sourcing stores history, not state.** Current state is computed.
- **Audit, temporal queries, multiple read models** are the big wins.
- **Schema evolution, GDPR, query complexity** are the big costs.
- **Pair with CQRS** for clean separation of write (event store) and read (projections).
- **Use snapshots** to keep read performance reasonable.
- **Apply selectively** — to aggregates that genuinely benefit (banking-style, audited workflows).
- **Don't event-source your whole system.** It's a tool, not a religion.
