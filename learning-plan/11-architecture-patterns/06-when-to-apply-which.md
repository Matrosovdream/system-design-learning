# Example 06 — When to apply which pattern (without over-engineering)

You now know CQRS, event sourcing, hexagonal/clean architecture, DDD bounded contexts, microservices, sagas, outbox, anti-corruption layers, and more. The trap is applying all of them by default. That's how you produce a "clean" system that nobody understands and that takes a year to add a button.

This example is the meta-architect's guide: **which patterns earn their keep, when**.

## The cost-benefit ledger of each pattern

| Pattern                       | Helps when                                       | Adds cost in                          | Skip when                                 |
|-------------------------------|---------------------------------------------------|----------------------------------------|--------------------------------------------|
| Hexagonal / Clean arch.       | Complex domain, need testability                  | Extra layers, more files               | Trivial CRUD                              |
| DDD bounded contexts          | Multiple teams, complex domain                    | Discipline, modeling effort            | Simple single-team product                |
| CQRS                          | Read/write patterns wildly differ                 | Eventual consistency, more infra       | CRUD where read = write shape             |
| Event sourcing                | Audit, temporal queries, complex workflows        | Schema evolution, GDPR pain            | CRUD; no audit need                       |
| Microservices                 | Multiple teams, independent scaling needs         | Operational footprint, distributed pain | Single team, small product               |
| Service mesh                  | 10+ services, mTLS need                           | Operational complexity                 | Small system                              |
| Sagas                         | Multi-service workflows                           | State machines, compensations          | Single-service operations                 |
| Outbox pattern                | At-least-once event publishing safety             | Extra table, relay job                 | No cross-service events                   |
| BFF                           | Multiple distinct clients                         | Extra service per client               | Single client (mobile only or web only)   |
| Anti-corruption layer         | Cross-context model translation                   | Mapping code, performance hit          | Same context                              |

## The "what stage are you?" heuristic

Most products go through these stages. The right patterns at each:

### Stage 1: MVP / Pre-product-market-fit

**Goal**: ship fast, learn what the product actually is.

**Apply**: nothing structural. Monolith. Single DB. Simple CRUD. Maybe rudimentary tests. Speed wins everything.

**Skip**: every pattern listed above. Yes, even hexagonal. You're discovering the domain; pattern-locked code resists discovery.

You can refactor *later* when the shape is clear. You can't undo months of wasted velocity.

### Stage 2: Early traction, growing team

**Goal**: stabilize, build velocity, on-board new engineers.

**Apply**:
- **Tests** for critical paths.
- **Clear module boundaries** within the monolith (modular monolith).
- **Hexagonal-lite** for complex domain code (the order workflow, the billing logic) — to keep tests fast and code clean.
- **API versioning** even on internal APIs (you'll regret skipping it).

**Skip**:
- Microservices.
- Event sourcing.
- CQRS.
- Service mesh.

Resist the temptation. You don't have the scale or the team size to amortize the cost.

### Stage 3: Established product, multiple teams forming

**Goal**: enable parallel team work without stepping on each other.

**Apply**:
- **DDD bounded contexts**: identify the natural seams.
- **Modular monolith with strict boundaries** (or extracted services for the most-painful seams).
- **Event-driven communication** between modules/services (Kafka or simpler).
- **Outbox pattern** for reliable event publishing.
- **API gateway** in front of the API.
- **Observability investment**: logs/metrics/traces all wired up.
- **CQRS for read-heavy endpoints** that need to scale specially.

**Skip**:
- Pure microservices for everything (modular monolith goes far).
- Event sourcing for non-audit data.
- Service mesh (still — wait until 10+ services).

### Stage 4: Scaling org and traffic

**Goal**: independence at scale; reliability under load.

**Apply**:
- **Microservices for bounded contexts** that need independent scaling/deploys.
- **Service mesh** if you have 10+ services and need mTLS / centralized retries.
- **Sagas** for cross-service workflows.
- **Event sourcing** for the few domains that genuinely benefit (audit, financial ledger).
- **CQRS extensively** for read-scale.
- **BFF per client type**.
- **Multi-region** if traffic/regulations require.
- **Comprehensive observability** including SLOs, error budgets, runbooks.

This is what a 100+ engineer company looks like.

### Stage 5: Multi-region, mission-critical

**Goal**: extreme reliability, regulatory compliance.

**Apply** everything plus:
- **Active-active multi-region** databases.
- **Chaos engineering** in production.
- **Detailed compliance controls**.
- **Disaster recovery** drills.

You've earned every line of complexity.

## The "the codebase has these symptoms" heuristic

Reverse approach: look at your code, find the pattern that fixes it.

| Symptom                                                                        | Pattern that fixes it          |
|--------------------------------------------------------------------------------|-------------------------------------|
| Can't write a unit test without a DB                                            | Hexagonal architecture              |
| Team A's change broke team B's feature                                          | Bounded contexts / extract module/service |
| The orders read endpoint joins 8 tables and is slow                             | CQRS (read model)                   |
| "What was the cart at 3pm yesterday?" — we don't know                           | Event sourcing                      |
| One bug in service A took down service B                                        | Circuit breakers, bulkheads         |
| Two services double-charge customers on retry                                   | Idempotency + outbox pattern        |
| Mobile clients fetch 5 endpoints per screen                                     | BFF + GraphQL                       |
| The legacy monolith is preventing all changes                                   | Strangler fig                       |
| One Redis node is melting; others bored                                         | Hot-key replication, L1 cache       |
| Cron job stops; nobody notices for a week                                       | Monitoring on absence of data       |
| We can't tell why this one user's checkout failed                               | Distributed tracing                 |
| We have 5 services and they all call the DB directly                            | DDD bounded contexts; restructure   |

If you don't have these symptoms, you don't need the pattern.

## Common over-engineering traps

### "Let's do microservices from day one"

The team is 5 engineers, 1000 users. You have **8 services**, each with their own DB. Every feature requires touching 3 services and writing 200 lines of plumbing.

**Reality check**: this is a distributed monolith. You haven't separated concerns; you've added latency, complexity, and failure modes to a fundamentally simple problem.

### "Let's event-source everything"

Every entity is event-sourced. Querying the current state of a user profile is replaying 47 events. Updating the schema requires writing 5 upcasters.

**Reality check**: event sourcing is amazing for the 5% of your domain that has audit / temporal requirements. Use it there. Use CRUD for the rest.

### "Let's use CQRS for the user profile"

The profile is one entity, you read and write it ~equally. You've now added Kafka, a projection consumer, eventual consistency.

**Reality check**: CQRS earns its keep where read and write models meaningfully differ. A simple profile entity doesn't.

### "Let's add hexagonal layers to this 50-line script"

The "domain" is `multiply two numbers`. The script now has 7 files: `entity.go`, `usecase.go`, `controller.go`, etc.

**Reality check**: layered architecture is for complex domains. Simple things should be simple.

## The patterns you should always apply (even at small scale)

Some patterns are cheap enough that they're worth doing from the start:

- **Some test discipline.** Even just smoke tests on critical paths.
- **Avoid hardcoded secrets.** Env vars or a vault from day one.
- **Idempotency keys** on the API endpoints that matter (Stripe-style).
- **HTTPS / TLS 1.3 everywhere.** Let's Encrypt is free.
- **Basic observability**: structured logging + a metrics dashboard. The 10 lines of code pay back at 3 a.m.
- **Versioning** in your APIs.

These don't add meaningful complexity but save real pain later.

## The architect's mental model

When considering a pattern, ask three questions:

1. **What pain does it solve?** Be specific. "Slow tests because of the DB", "Distributed monolith with no team independence", "Reads scaling separately from writes".
2. **Do I have that pain now?** Not "might have someday". Now or imminently.
3. **What's the cost in operational and engineering overhead?**

If the answer to #1 is vague, or #2 is "not really", or #3 outweighs the benefit — don't apply it.

Equally important: **revisit decisions as the product grows.** What was right at 5 engineers may be wrong at 50.

## The right shape grows organically

You start simple. You add patterns when their pain becomes acute. You **delete** patterns when their pain has gone away (rare, but happens).

Architecture is **an ongoing process**, not a one-time decision. The best architects are constantly tuning the shape to match what the product and team need today.

## Architect's takeaway

- **Apply patterns to solve concrete pain**, not to follow trends.
- **Stage your architecture growth** with team and product size.
- **Modular monolith goes very far** — don't underestimate it.
- **Always-apply, low-cost things**: tests, secrets management, idempotency, HTTPS, observability, API versioning.
- **The cost of premature abstraction** is bigger than the cost of refactoring later. Stay flexible.
- **Re-evaluate patterns over time** — what fit yesterday may not fit today.
- **Your job as an architect is mostly saying "no, not yet"** to patterns the team is excited to apply but doesn't need.
