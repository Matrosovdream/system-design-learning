# Example 06 — The scaling story: 1 user to 100M, step by step

This is the same exercise as Xu V1 Chapter 1. It glues together everything in this step into the *actual evolution* of a system. Read it twice.

## Stage 0 — A single box (1 to ~1k users)

```
[user] ─► [server: app + DB + everything]
```

One VPS. App and DB co-located. PHP/Apache or Go/systemd. Postgres or MySQL on the same machine. Domain points straight at this IP.

**Capacity:** ~1k DAU, maybe ~10 RPS.
**Bottleneck:** none yet.
**Cost:** $20/month.

## Stage 1 — Separate DB tier (~1k to ~10k users)

```
[user] ─► [app server] ─► [database server]
```

Pull the DB onto its own box. Two reasons: DB needs different tuning (RAM-heavy, IOPS-sensitive), and you can scale them independently from here on.

**Capacity:** ~10k DAU.
**Bottleneck:** single app server is the SPOF.

## Stage 2 — Load balancer + multiple app servers (~10k to ~100k)

```
                ┌─► [app server 1] ─┐
[user] ─► [LB] ─├─► [app server 2] ─┼─► [database]
                └─► [app server 3] ─┘
```

Three identical PHP/Go app servers. Nginx or AWS ALB in front. App tier must now be **stateless** — sessions in Redis or a signed cookie, not in PHP's local session files.

**Capacity:** ~100k DAU. App tier scales linearly by adding boxes.
**Bottleneck:** the single DB. All reads and writes go through it.

## Stage 3 — Read replicas (~100k to ~1M)

```
                ┌─► [app] ─┐                      ┌─► [DB replica 1] (reads)
[user] ─► [LB] ─├─► [app] ─┼─► [DB primary] ─repl─┼─► [DB replica 2] (reads)
                └─► [app] ─┘   (writes)            └─► [DB replica 3] (reads)
```

Most workloads are read-heavy (look at the Twitter BoE from step 01 example 03: 50:1 reads). Add 1-3 **read replicas**. App is now aware that writes go to the primary, reads go to a replica.

Beware **replication lag** — a write on the primary may not show up on the replica for tens of milliseconds. For "read your own writes", read from the primary briefly after a user's write.

**Capacity:** ~1M DAU.
**Bottleneck:** primary DB writes; static-asset bandwidth on app servers.

## Stage 4 — Cache layer + CDN (~1M to ~10M)

```
                                        ┌─► [Redis cluster] (hot data)
                                        │
[user] ─► [CDN] ─► [LB] ─► [app pool] ──┼─► [DB primary] (writes)
   (static, media)                       └─► [DB replicas] (reads)
```

- **CDN** offloads static assets and media → app servers stop being bandwidth-bound.
- **Redis cache** offloads hot reads → DB replicas have headroom.

The 80/20 rule: ~80% of reads hit the cache and never touch the DB.

**Capacity:** ~10M DAU.
**Bottleneck:** primary DB writes; cold-cache thundering herd; cache invalidation correctness.

## Stage 5 — Async work via queues (~10M+)

```
[app] ─► [DB primary] (synchronous write)
   │
   └─► [message queue] ─► [worker pool] ─► [DB, email, search index, etc.]
```

Anything that doesn't have to happen *during* the request is moved off-line:
- Send welcome email → enqueue, worker handles.
- Rebuild user timeline → enqueue, worker handles.
- Generate thumbnail → enqueue, worker handles.

The user's request returns in 50 ms instead of 500 ms. Workers can be slow without anyone noticing.

**Bottleneck:** primary DB writes are still single-instance.

## Stage 6 — Database sharding (~10M to 100M+)

```
[app] ─► [shard router]
            ├─► [DB shard 1: users 0-25%]   (primary + replicas)
            ├─► [DB shard 2: users 25-50%]
            ├─► [DB shard 3: users 50-75%]
            └─► [DB shard 4: users 75-100%]
```

Now even **writes** scale horizontally. Each shard is a full DB cluster (primary + replicas) responsible for a slice of the data. Picked by **consistent hashing** on user_id usually.

This is the **big jump**. Adds operational complexity:
- Cross-shard queries are now expensive or impossible.
- Resharding is painful.
- Foreign keys cross shards — you give them up.

This is also where you usually accept giving up some ACID guarantees, embrace **eventual consistency** for some data, and start adopting **CQRS** / **event sourcing** for cross-shard workflows.

## Stage 7 — Multi-region (global product)

```
                     Region: US-East
                                   ┌─► [app pool] ─► [DB primary]
[user, US] ─► [DNS routing] ─► [LB]
                                   └─► [cache]

                     Region: EU-West
                                   ┌─► [app pool] ─► [DB primary]
[user, EU] ─► [DNS routing] ─► [LB]
                                   └─► [cache]
                                                  ↕ async replication
                     Region: Asia-Pacific
                                   ┌─► [app pool] ─► [DB primary]
[user, APAC] ─► [DNS routing] ─► [LB]
                                   └─► [cache]
```

Reasons to go multi-region:
1. **Latency** for global users (Sydney users hitting a Sydney datacenter).
2. **Regulatory** (data residency: EU users' data must live in the EU).
3. **Disaster recovery** (one region down ≠ business down).

Multi-region opens the hardest distributed-systems can of worms: cross-region writes, conflict resolution, consensus across high-RTT links. We tackle this in step 07.

## What we did NOT do

We kept "the architecture" but never:
- Wrote weird code.
- Adopted a buzzword for the sake of it.
- Sharded before we had to.

This staged progression is roughly how every large system grew. You can skip stages if you *know* you're starting at scale (e.g., a viral consumer app at launch needs CDN + cache + replicas on day one). But premature sharding is the architect's classic mistake.

## Architect's takeaway

- **You add infrastructure at the moment the previous design hurts**, not before.
- The order is roughly: **separate DB → LB + multi-app → read replicas → cache + CDN → queues → shard the DB → multi-region.**
- Each new component pays for itself with one bottleneck removed and brings one new failure mode.
- Doing this well is **90% knowing when to stop adding things.** Most apps live happily at stage 4 forever.
