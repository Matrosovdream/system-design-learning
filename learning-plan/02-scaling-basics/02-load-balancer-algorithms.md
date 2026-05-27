# Example 02 — Load-balancing algorithms

A load balancer (LB) picks which backend server gets the next request. The choice of algorithm shapes performance, fairness, and cache behavior.

## L4 vs L7 first

- **L4 LB** (e.g., AWS NLB, HAProxy in TCP mode): routes on **IP + port**. Cannot see HTTP headers, cookies, or paths. Very fast (millions of conn/s), used in front of TCP/UDP services and as the first hop for huge traffic.
- **L7 LB** (e.g., Nginx, Envoy, AWS ALB, HAProxy in HTTP mode): routes on **HTTP path, headers, cookies, method**. Can route `/api/*` to one pool and `/admin/*` to another. Used for app-aware routing.

Most modern stacks: an L4 LB at the edge (high throughput, TLS pass-through) → L7 LB behind it (smart routing).

## The 5 algorithms you must know

### 1. Round-Robin (RR)

```
req1 → server A
req2 → server B
req3 → server C
req4 → server A
...
```

**Best when:** all servers are identical and requests are roughly uniform in cost.

**Fails when:** requests vary wildly in cost (a `/search` is 10× more expensive than `/health`). A slow request on server A blocks A while B and C sit idle.

### 2. Least-Connections

LB tracks how many open connections each server has. Send the next request to the server with the fewest.

```
A: 5 conns, B: 2 conns, C: 7 conns → next req goes to B
```

**Best when:** request durations vary. Naturally avoids piling onto a slow server.

**Default choice** for most general workloads. Slightly more expensive than RR (LB must track state).

### 3. Weighted Round-Robin / Weighted Least-Connections

Each server has a weight reflecting its capacity. A server with weight 3 receives 3× the traffic of a weight-1 server.

```
A (w=3), B (w=1), C (w=1) → A, A, A, B, C, A, A, A, B, C, ...
```

**Best when:** your fleet is **heterogeneous** — some boxes are bigger than others (e.g., during a migration).

### 4. IP-Hash (or Source-Hash)

LB hashes the client's IP and uses the hash to pick a server.

```
hash(client_ip) % N → server index
```

**Best when:** you need **session affinity** without cookies — same client always routed to the same server.

**Fails when:** you add/remove a server. `% N` changes for everyone → all sessions break. Use **consistent hashing** instead (see step 05).

### 5. Consistent Hashing

Hash both the request key (often client ID or session key) and the servers onto a ring. Each request goes to the next server clockwise on the ring.

```
adding/removing one server only re-routes ~1/N of requests
(vs ~all of them with naive mod-hash)
```

**Best when:** you have a stateful caching layer (e.g., Memcached, sharded Redis) and adding/removing a node should disturb cache as little as possible. Used heavily inside CDNs and DB shards.

This is also the algorithm of choice when the "server" is itself a cache and re-hashing is expensive.

## Comparison table

| Algorithm           | Fairness | Latency aware? | Session affinity? | Re-route on change      | Best for                                  |
|---------------------|----------|----------------|--------------------|--------------------------|-------------------------------------------|
| Round-Robin         | Strict   | No             | No                 | Trivial                  | Identical servers, uniform requests       |
| Least-Connections   | Adaptive | Yes (indirect) | No                 | Trivial                  | Variable request costs                    |
| Weighted RR/LC      | Tunable  | No / Yes       | No                 | Trivial                  | Heterogeneous fleet                       |
| IP-Hash             | Skewed   | No             | Yes                | All clients re-route     | Quick affinity, fixed fleet               |
| Consistent Hashing  | Tunable  | No             | Yes (by key)       | Only 1/N keys re-route   | Cache shards, partitioned data            |

## Health checks (the silent feature)

All these algorithms assume "list of healthy servers". The LB **must** continuously health-check:
- **Active checks**: LB sends `GET /healthz` every N seconds.
- **Passive checks**: LB observes failed responses and ejects the bad server.

A server with a 500-rate above threshold is **ejected** from rotation. After cooldown, the LB tries it again.

Without health checks, even the best algorithm sends 1/N of traffic to a dead server.

## Architect's takeaway

- **Least-connections is the safe default** for app servers.
- **Round-robin** is fine when requests are uniform (e.g., simple static content).
- **Consistent hashing** is the *correct* answer for cache/shard fronting — naive hashing fails on scale changes.
- **Always health-check.** The algorithm is irrelevant if you're routing to a corpse.
- **L4 in front of L7** is a common high-throughput pattern; L7 alone for most apps under ~100k RPS.
