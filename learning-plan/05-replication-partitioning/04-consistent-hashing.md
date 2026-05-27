# Example 04 — Consistent hashing: the ring, virtual nodes, and why naive `% N` fails

You have N cache servers and you want to spread keys evenly across them. Naive answer: `server_index = hash(key) % N`. This works until N changes — then nearly every key is reassigned, your cache is empty, and your DB falls over.

Consistent hashing fixes this.

## The bug with naive modulo

```
N = 4 servers.
hash("alice") = 12345 → 12345 % 4 = 1 → server 1.
hash("bob")   = 67890 → 67890 % 4 = 2 → server 2.
```

Now add a 5th server.

```
hash("alice") = 12345 → 12345 % 5 = 0 → server 0.   (moved!)
hash("bob")   = 67890 → 67890 % 5 = 0 → server 0.   (moved!)
```

**Almost every key moves.** Cache hit rate goes to ~0%. The DB gets crushed by misses while the cache rebuilds.

For a sharded **database** with terabytes of data, this is even worse: every key change means moving the actual row to a new shard.

## The consistent hashing idea

Place both **servers** and **keys** on a single circular hash space (the "ring"). A key is owned by the **next server clockwise** from where the key lands.

```
        Ring positions (0..2^32-1)

         [server A @ pos 100]
            ╱
key 50 → next clockwise is A
key 110 → next is B
key 700 → next is C
key 999 → wraps around → next is A

         [server C @ pos 800]
            ╲
              ╲
                [server B @ pos 200]
```

When a server is added/removed, only the **arc between it and the previous server** is reassigned. That's roughly **1/N** of all keys, not "all keys".

## Pseudocode

```python
def consistent_hash_lookup(key, servers):
    key_pos = hash(key)
    # find smallest server position >= key_pos
    for srv_pos, srv in sorted(servers.items()):
        if srv_pos >= key_pos:
            return srv
    # wrap around
    return servers[min(servers.keys())]
```

With a sorted server map, lookup is O(log S) where S is server count.

## The problem with the basic ring: uneven distribution

If three servers land at positions 100, 105, and 800 on a 0..1000 ring:
- A owns arc [800 → 100] = 300 → 30% of keys
- B owns arc [100 → 105] = 5 → 0.5% of keys
- C owns arc [105 → 800] = 695 → 69.5% of keys

C is melting; B is bored. **A single hash position is too sensitive to placement luck.**

## The fix: virtual nodes ("vnodes")

Each physical server gets **many** positions on the ring (typically 100-1000 vnodes per server).

```
server A: positions 100, 250, 410, 670, ...
server B: positions 50, 180, 320, ...
server C: positions 30, 220, 480, 900, ...
```

Now the arcs are tiny and the distribution evens out by law of large numbers. Each server holds ~1/N of the ring regardless of placement.

Vnodes also let you weight servers (give the bigger server 2× the vnodes).

## What it costs you on rebalancing

Adding a new server N+1 with V vnodes:
- ~V tiny arcs get reassigned.
- Each arc holds ~1/(N×V) of total data.
- New server ends up with ~1/(N+1) of data.
- **Only ~1/(N+1) of data moves, not all of it.**

The new server is "rebalancing in" while existing servers continue serving 100% of their keys minus the ones moving. Cache hit rate stays high during the transition. **This is the property that makes the algorithm worth using.**

## Where consistent hashing is used

- **Memcached client-side sharding** (libmemcached's ketama algorithm).
- **Redis Cluster** — uses 16384 hash slots, which is a coarse-grained form of consistent hashing (slots can be moved between nodes).
- **DynamoDB** internal partitioning.
- **Cassandra** (token-based partitioning, virtual nodes since 1.2).
- **Akamai CDN** (the algorithm's original application).
- **Discord's voice-server allocation**.
- **HAProxy and Envoy** for sticky load balancing of stateful upstreams.

## A worked example with Redis Cluster

Redis Cluster pre-decides on **16384 slots**. Each key hashes to one slot:

```
slot = CRC16(key) % 16384
```

Nodes own ranges of slots:

```
node A: slots [0..5460]
node B: slots [5461..10922]
node C: slots [10923..16383]
```

Adding node D: rebalance ~4096 slots from A/B/C to D. Each slot move is a small range of data. Cluster stays online; only the keys in moving slots see a brief redirect (`MOVED`).

This is exactly the property described above, with the ring discretized into 16384 buckets for operational simplicity.

## Beyond pure consistent hashing — newer ideas

- **Jump consistent hash** (Lamping & Veach 2014) — same property, no ring data structure needed, O(1) space, deterministic. Used in some Google systems.
- **Rendezvous (HRW) hashing** — each key hashes against each server; the highest hash wins. Equally good redistribution properties, often simpler to implement.

For caches and load balancers, **all three** are effectively interchangeable from the architect's perspective; pick what your library implements.

## Architect's takeaway

- **Don't shard with `hash(key) % N`** — adding a node empties the cache.
- **Consistent hashing** moves only ~1/N of keys when topology changes. That's the whole point.
- **Use virtual nodes** to even out the distribution. ~100-1000 per physical server is standard.
- **Redis Cluster's hash-slot approach** is the practical incarnation you'll see most often.
- **You'll rarely implement this from scratch** — pick a library that already does it correctly.
