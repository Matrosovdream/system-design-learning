# Example 01 — Vertical vs horizontal scaling

Two ways to handle more load. They're not interchangeable.

## Vertical (scale up)

Same number of boxes, bigger box. Move from a 4-vCPU / 16 GB RAM machine to a 32-vCPU / 256 GB RAM machine.

**Pros**
- **Zero code change.** Your monolith is already running; just pick a bigger instance.
- **No distributed-systems complexity.** Still one process, one local memory, one source of truth.
- **Latency stays the same** — no extra network hops.

**Cons**
- **Hard ceiling.** Even the biggest cloud instance (e.g., AWS u-24tb1, ~24 TB RAM, ~448 vCPUs) is finite.
- **Cost curve is sub-linear above a point.** Doubling the box rarely doubles performance — bus, memory bandwidth, NUMA effects kick in.
- **Single point of failure.** When that one big box crashes, you're down.
- **Downtime to upgrade** (usually — some clouds offer live resize, but not unlimited).

## Horizontal (scale out)

Same box size, more boxes. Go from 1 to 10 to 100 app servers behind a load balancer.

**Pros**
- **No ceiling** in theory — just keep adding machines.
- **Failure isolation** — losing one of 100 boxes is a 1% capacity hit, not an outage.
- **Cheap commodity hardware** — many small boxes are usually cheaper per unit-of-work than one big one.
- **Independent deploys, rolling upgrades** — much easier than scheduling downtime on the One Big Box.

**Cons**
- **Requires statelessness** — if the app stores session state in memory, request #2 might land on a different server and lose context.
- **Distributed systems problems** — partial failure, network partitions, distributed locking, distributed transactions.
- **Coordination overhead** — service discovery, health checks, config management.

## The realistic progression

You don't pick one. Almost every real system uses **both**:

```
Stage 1: 1 box (small)                          ← vertical
Stage 2: 1 box (bigger)                         ← vertical (cheap step)
Stage 3: 2-3 boxes + load balancer              ← horizontal starts
Stage 4: app servers scale out, DB stays one    ← hybrid
Stage 5: DB scales up to a huge box             ← vertical on data tier
Stage 6: DB read replicas (horizontal reads)    ← horizontal on data
Stage 7: DB sharding (horizontal writes)        ← horizontal everywhere
```

The rule of thumb: **scale vertically first** because it's free engineering effort. **Scale horizontally when you hit the ceiling** or when you need redundancy.

## The economic reality

Scaling vertically beyond mid-range cloud instances gets **disproportionately expensive**:

| AWS EC2 (rough US-east on-demand prices, vary over time)        |
|----------------------------------------------------------------|
| `t3.large`   (2 vCPU, 8 GB)    : ~$0.08/hour                   |
| `m5.xlarge`  (4 vCPU, 16 GB)   : ~$0.20/hour    → 2.5× the price for 2× the resources |
| `m5.4xlarge` (16 vCPU, 64 GB)  : ~$0.77/hour    → 9.6× the smallest for 8× resources |
| `m5.24xlarge`(96 vCPU, 384 GB) : ~$4.60/hour    → 57× the smallest for 48× resources |

So 1× `m5.24xlarge` ≈ same cost as 6× `m5.4xlarge`. The 6 small boxes give you redundancy too. **Horizontal often wins on cost above mid-range.**

## When vertical is unavoidable

Some workloads genuinely **don't shard well**:
- **Single huge in-memory dataset** (some analytics engines, graph databases).
- **Strong serial-write requirements** on one entity (e.g., a hot-row counter without sharding).
- **Legacy code that assumes single-process** (often the practical limit).

In these cases, vertical is the only lever — until you redesign the data model.

## Architect's takeaway

- Always **start vertical** unless you already know you'll need 5+ boxes (then start horizontal-ready from day one).
- **Move horizontal at the latest sensible moment**, but design for it earlier (statelessness, externalized session storage).
- **App tier scales horizontally easily**; **data tier scales vertically first**, then via replicas, then via sharding (much harder).
- The most common production layout: **horizontally scaled app tier + vertically large primary DB + horizontally scaled cache**.
