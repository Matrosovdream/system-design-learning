# Example 01 — Monolith vs microservices: when and why

The most over-decided choice in modern software architecture. The right answer is almost always context-dependent, and "microservices everywhere" is a costly mistake at small scale.

## Monolith — one process, one codebase

```
[client] → [load balancer] → [app instance (PHP/Go monolith)] → [shared DB]
```

- One repository.
- One deployment unit.
- One database.
- One language usually.
- Internal calls are function calls in the same process.

## Microservices — many small processes, one product

```
[client] → [API gateway] → [service A]
                       ├──► [service B]
                       └──► [service C]
                              │
                             [service D]
                              │
                             [DB owned by D]
```

- Many repositories (or a monorepo with many services).
- Each service: own codebase, own database, own deploy cadence, possibly own language.
- Internal calls are network calls (HTTP/gRPC/messages).

## What microservices actually buy you

### 1. Independent deploys

Team A ships service A every hour. Team B ships service B once a week. They don't coordinate. **This is the single biggest reason** large companies adopt microservices.

In a monolith, every deploy is everyone's deploy.

### 2. Independent scaling

Service A is CPU-bound; you run 50 instances. Service B is memory-heavy; you run 5 big instances. Service C is barely used; you run 1.

In a monolith, you scale the whole thing as one unit.

### 3. Fault isolation

Service A's bug doesn't crash service B's process. (Sometimes.)

In a monolith, a memory leak anywhere kills everything.

### 4. Technology diversity

Service A in Go (performance), service B in PHP (existing), service C in Python (ML libs). Each team picks its tools.

In a monolith, the tech is decided once for everyone.

### 5. Team scaling

10 teams of 5 engineers each can work on 10 services without code-conflict pain. Conway's law in reverse: the architecture matches the org.

In a monolith, 50 engineers in one repo is a merge-conflict story.

## What microservices actually cost you

### 1. Network is unreliable, slow, and complex

What was a function call (~ns) is now a network call (~ms) that can fail. You need timeouts, retries, circuit breakers, idempotency.

### 2. Distributed transactions are hard

You can't `BEGIN TRANSACTION` across services. You need sagas, outbox, idempotency keys (covered in steps 06-07).

### 3. Deployments multiply

10 services × deploys/week = 10× more pipelines, more rollback artifacts, more "is this service compatible with that one's old contract?"

### 4. Observability is harder

A single request now flows through 5 services. You need distributed tracing or you're blind.

### 5. Operational footprint explodes

Was 1 app + 1 DB. Now 10 services + 5 DBs + 1 message broker + 1 service mesh + 1 API gateway + 1 secrets manager + ...

### 6. The "wrong boundary" trap

Carve service boundaries the wrong way and every feature touches 5 services. You've created **a distributed monolith**: all the operational cost of microservices, none of the independence. The worst architecture.

## When monolith is right

- **Team < 20 engineers.**
- **Single product, single bounded context.**
- **You don't yet know the access patterns.**
- **You're early in product-market fit** — pivots are common, and refactoring a monolith is easier than re-drawing service boundaries.
- **You don't have a sophisticated platform team** to run microservices infra.

For an early-stage startup: **monolith. Almost always.**

## When microservices are right

- **Org has multiple independent teams** (8+ teams) wanting to ship without coordination.
- **Stable, well-understood domain** — you can draw the boundaries with confidence.
- **Scale dictates it** — one component is constrained by another's resource needs.
- **You have a platform team** running k8s, observability, service mesh.
- **Existing monolith hurts to deploy** — minutes-long CI, rollbacks rare, fear of touching shared code.

## Modular monolith: the sweet spot many teams miss

Between "one big mess" and "20 microservices" lies the **modular monolith**:

- One deployment unit.
- Internally split into clear modules with well-defined APIs.
- Each module has its own folder structure and ownership.
- All run in one process; calls between modules are function calls.

You get most of the **architectural** benefits of microservices (clear boundaries, team ownership) without the **operational** cost.

You can extract a module to a microservice later, **if and when** you need to.

This is what Shopify, Basecamp, GitHub (historically), and many high-scale Rails/PHP/Go shops actually run.

## The migration pattern

If you're moving from monolith to microservices, you don't do it in one go. You **strangle** (see example 06).

```
year 0: 100% monolith
year 1: monolith + 1 extracted service (highest-pain area)
year 2: monolith + 4 services
year 3: monolith shrunk to a "legacy core" + 10 services
year 5: monolith fully replaced (or surviving as a small auth/billing core)
```

Each extraction must justify its complexity. "We took this out as a service" should mean "and our deploy frequency for this area is 5× higher now".

## Common mistakes

### Microservices at startup

"We want to be web-scale ready" — but you have 8 engineers and 2,000 users. You're paying microservices tax for nothing. Build a monolith.

### Microservices for resume-driven development

"I want to put k8s on my resume." Predictable result: 6 services for a TODO app. Months of yak-shaving instead of shipping features.

### One service per table

Extracting a "users service" because there's a `users` table. The service has no useful domain logic; every API call to it is just SQL pass-through. You added network hops for nothing.

### Distributed monolith

Service A can't deploy without service B redeploying. Service C's DB calls go through service B. Service D's tests need all services running.

Symptoms: you can't deploy independently, even though you have microservices. Worst of both worlds.

## The architect's heuristic

Ask: **"Can these two teams ship their work without coordinating with each other?"**

- If yes, they can each own a service.
- If no, you've drawn the boundary in the wrong place.

This is **Conway's Law applied deliberately**: your service boundaries should reflect your team boundaries.

## Architect's takeaway

- **Monolith is the right starting point** for almost every product.
- **Microservices solve organizational problems**, not technical ones. Adopt them when org scale demands it.
- **Modular monolith** gives most of the benefits with little of the cost. Underrated.
- **Service boundaries should match team boundaries.** If they don't, extract differently.
- The **distributed monolith** is worse than either pure shape. Be vigilant.
