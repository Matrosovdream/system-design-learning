# Step 11 — Architecture Patterns

The vocabulary of senior architects. CQRS, event sourcing, hexagonal architecture, DDD, clean architecture — these aren't competing styles, they're complementary tools. You'll use them in combination. The trick is to apply each only when its problem actually exists in your system.

## Goals

- Distinguish **CQRS** from **event sourcing** (often conflated, actually independent).
- Understand **event sourcing** trade-offs, not just its appeal.
- Draw a **hexagonal architecture** (ports and adapters) and explain what it buys you.
- Identify **bounded contexts** in a domain — the DDD core concept.
- Sketch the layers of **clean architecture** and what flows where.
- Pick patterns to apply in a real system — and which to skip.

## Key concepts

1. **CQRS** — Command Query Responsibility Segregation: separate the write model from the read model.
2. **Event sourcing** — store the sequence of events that produced the state, not the state itself.
3. **Hexagonal architecture (ports & adapters)** — core domain isolated behind interfaces; adapters for DBs, APIs, queues.
4. **Domain-Driven Design (DDD)** — bounded contexts, aggregates, ubiquitous language.
5. **Clean architecture** — concentric layers, dependencies point inward.
6. **Anti-corruption layer** — translate between bounded contexts.
7. **Event-driven architecture** — services react to events (relates to step 06).
8. **Modular monolith** — all the above without microservice overhead.

## Reading

- **Eric Evans**: *Domain-Driven Design* — the canonical DDD book (dense).
- **Vaughn Vernon**: *Implementing Domain-Driven Design* — more practical.
- **Martin Fowler**: blog posts on CQRS, event sourcing, hexagonal.
- **Robert C. Martin**: *Clean Architecture* — the layered model.
- **Alistair Cockburn**: original hexagonal architecture paper.

## Examples in this folder

- `01-cqrs.md` — separating reads and writes.
- `02-event-sourcing.md` — events as the source of truth.
- `03-hexagonal-architecture.md` — ports and adapters.
- `04-ddd-bounded-contexts.md` — drawing the right service boundaries.
- `05-clean-architecture.md` — the dependency rule.
- `06-when-to-apply-which.md` — picking the right combo without over-engineering.

## Self-check

1. CQRS and event sourcing are often discussed together. Why are they actually independent?
2. Hexagonal architecture's main benefit is testability. Why?
3. You have an "Order" concept that means different things in the sales context vs the warehouse context. What does DDD recommend?
4. In clean architecture, why do "use cases" depend on "entities" and not the other way around?
5. You're building a 5-engineer startup MVP. Which of these patterns apply? Which don't?
