# Example 04 — DDD bounded contexts: drawing the right service boundaries

The single most important DDD concept. **A bounded context is a portion of the system where a particular model applies.** Outside that boundary, the same word means something different.

Get bounded contexts right → your services and modules align with reality, evolve cleanly.
Get them wrong → distributed monolith, perpetual integration headaches.

## The problem DDD points at

Consider the word "Order" in an e-commerce company:

- **Sales context**: an Order is a checkout transaction with line items and a total.
- **Warehouse context**: an Order is a list of items to pick from shelves, with locations and a pick sequence.
- **Shipping context**: an Order is a package to deliver, with addresses and tracking.
- **Customer Service context**: an Order is an interaction history — when it was placed, where it is, complaints filed.
- **Finance context**: an Order is a recorded sale for accounting purposes.

The same word, **five different concepts.** Each context has its own model, its own attributes, its own lifecycle, its own business rules.

The classic mistake: build a single `Order` entity that tries to satisfy everyone. The result is an unwieldy God-object with 40 fields, some only relevant in some contexts, and constant tension between teams.

## The DDD answer: bounded contexts

Each context has its **own model** of "Order", with the **attributes and behaviors that matter to it**, and **nothing else**.

```
Sales context              Warehouse context           Shipping context
─────────────              ─────────────────           ─────────────────
Order:                     PickList:                   Shipment:
  id                         id                          id
  customer                   warehouse                   carrier
  items[]                    items[]                     tracking_number
  total                      pick_sequence               address
  status                     picker                      delivery_eta
  placed_at                  picked_at                   delivered_at
```

These models are **different things**. They share an ID, but their fields, behaviors, and lifecycles diverge.

Each is implemented in its own bounded context — usually its own service (or module in a modular monolith), its own DB, its own team.

## Ubiquitous language

Inside a bounded context, **everyone uses the same words** — engineers, product managers, domain experts, the code.

- In the Sales context: "place order", "cancel order", "discount", "checkout".
- In the Warehouse context: "pick", "stage", "pack", "ship".
- In the Shipping context: "carrier", "tracking", "delivery", "return".

A "shipment" in Warehouse might be the same concept as a "shipment" in Shipping, or it might not — the language **inside** a context must be precise, even if it overlaps with another context's vocabulary.

The discipline: **never have a meeting where "shipment" means different things to different people in the same room.** Pick a context; speak its language; if you need to talk across contexts, name both terms.

## How bounded contexts talk

Each context is autonomous. They communicate through **explicit, narrow contracts**:

- **APIs** (REST, gRPC).
- **Events** (Kafka, message queue).
- **Shared data via a designated channel** (CDC, ETL).

**Never** do they share databases or models directly. The boundary is a hard wall.

### Anti-corruption layer (ACL)

When you call another context, you don't let its model leak into yours. You translate at the boundary.

```
Sales context wants to know "is this item in stock?"

Sales code: stock_check_service.is_in_stock(sku, quantity)

stock_check_service (in the Sales context):
   call warehouse-context API: GET /inventory?sku=...
   the response uses Warehouse's terms ("storage_location", "available_units", "reserved", ...)
   translate to Sales's terms: { in_stock: true/false }

Sales code only ever sees its own model. Warehouse's concepts stop at the ACL.
```

This prevents Warehouse's model from infecting Sales, and lets each evolve independently. ACL is a Pattern with a capital P in DDD; you'll see it called out in real designs.

## Aggregates: the unit of consistency

Within a bounded context, an **aggregate** is a cluster of objects that are consistent together.

```
Order aggregate:
  - Order (root)
    - LineItem[]
    - ShippingAddress
    - DiscountCode
```

Rules:
- All access goes through the aggregate root (`Order`). You don't get a `LineItem` directly; you get the Order and walk to it.
- The whole aggregate is loaded/saved together. ACID transactions apply to the aggregate.
- Different aggregates communicate via events (within the same context) or via APIs (across contexts).

This is what makes the "save the whole order atomically" pattern work clean.

### Aggregate size matters

Too big → contention, slow loads, hard to maintain.
Too small → every operation touches multiple aggregates → lost ACID guarantees.

Rule of thumb: an aggregate should be small enough to load and save quickly, big enough to enforce its own invariants.

## Mapping contexts to services

Once you've identified bounded contexts, the service boundary often **follows the context boundary**:

```
Bounded Context           →    Service / Module
─────────────────              ──────────────────
Sales                          sales-service (or sales/ module)
Warehouse                      warehouse-service
Shipping                       shipping-service
Customer Service               support-service
Finance                        finance-service
```

This is Conway's Law applied deliberately. Teams own contexts. Services match contexts. Boundaries align with how the business actually thinks.

This also explains why microservices done badly fail: people split services along **technical** lines ("users-service", "products-service") instead of **business contexts**. The result: every business workflow crosses 5 services, all coupled.

Good microservices align with bounded contexts. Wrong microservices align with database tables.

## Identifying bounded contexts

Approaches:

### 1. Event storming

A workshop where domain experts and engineers map out the **events that happen** in the business. Events cluster around contexts.

"Order placed", "Payment received", "Items picked", "Shipment dispatched", "Delivery completed" — the natural groupings reveal the contexts.

### 2. Listen for language drift

When you hear engineers debating "what does 'shipped' mean" — that's a context boundary trying to surface.

### 3. Team boundaries (Conway's Law)

What does each team own? If two teams are colliding on the same model, you have a context boundary that hasn't been recognized.

## What happens without bounded contexts

- One mega-`Order` entity with 50 fields, half null at any given time.
- Every team's changes break the others.
- Every feature request requires negotiating with three teams.
- "Just add a status field" → endless conversation about which status meanings apply.
- Tests require setting up data for contexts that are irrelevant.

You've seen this in every legacy enterprise system. DDD's diagnosis is precise: contexts aren't drawn.

## The context map

A diagram showing all your bounded contexts and how they relate:

- **Customer/Supplier**: one context depends on another (downstream/upstream).
- **Shared Kernel**: a small shared model (rare, risky).
- **Anti-Corruption Layer**: translation at the boundary.
- **Open Host / Published Language**: a context publishes a stable API for everyone else.
- **Separate Ways**: two contexts don't integrate; they live independently.

Drawing this for a real system clarifies a lot about who depends on whom.

## When (not) to apply DDD

DDD shines when:
- **Domain is complex** (insurance, healthcare, finance, logistics).
- **Multiple teams** working in parallel.
- **Long-lived system** that must evolve over years.
- **Stakeholders have real domain knowledge** to bring to the design.

DDD is overkill when:
- **Pure CRUD** with no real business logic.
- **Single small team**, single domain.
- **Throwaway / MVP** code.

Even there, **the bounded-context idea is still useful** — just don't go through the full DDD methodology.

## Architect's takeaway

- **Bounded contexts are the most important DDD concept.** They define where a model applies.
- **The same word means different things** in different contexts. Embrace it.
- **Ubiquitous language** inside each context — everyone uses the same words.
- **Contexts communicate through explicit contracts** (APIs, events), never shared DBs.
- **Anti-corruption layers** at every cross-context boundary keep models from leaking.
- **Services map to bounded contexts**, not to tables. This is the key insight microservices teams need.
- **Apply DDD where complexity warrants it.** It's a discipline, not a religion.
