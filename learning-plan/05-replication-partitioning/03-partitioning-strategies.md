# Example 03 — Partitioning strategies: range, hash, directory

When data outgrows a single machine, you partition it. The strategy you pick determines query performance, hotspots, and how painful rebalancing is.

## 1. Range partitioning

Divide the key space into contiguous ranges, one per partition.

```
shard A: keys [a..h]
shard B: keys [i..p]
shard C: keys [q..z]
```

### Pros

- **Range queries are cheap.** `SELECT * WHERE name BETWEEN 'b' AND 'e'` hits one shard.
- Keys close together stay together — useful for time-series, log-structured data, sorted access.

### Cons

- **Hotspots are easy to create.** Keys created at the same time (e.g., `created_at` partition key) all land on the newest shard. The other shards are idle.
- **Manual rebalancing required** — you need to track which ranges are getting too big and split them. Most systems auto-split (HBase, BigTable, CockroachDB).

### Used by

- HBase, BigTable, Spanner (after secondary indexing).
- CockroachDB (default — automatic splits).
- MongoDB sharded collections (when you choose ranged shard key).
- DynamoDB sort key (within a partition key).

### When this wins

- You query by **range** more than by key (time-series, geo locality, ordered scans).

### Pitfall

Partitioning by `created_at` is the classic mistake. The newest range gets 100% of writes. Old shards do nothing.

Fix: hash the date with the user_id (e.g., `(user_id, created_at)` so writes are spread by user) — but then range scans by time alone need to fan out.

## 2. Hash partitioning

Hash the key, mod by number of partitions.

```
shard_index = hash(key) % N
```

### Pros

- **Uniform distribution.** No hot shard from monotonically-increasing keys.
- Trivial to implement.
- Works for any key type.

### Cons

- **Range queries fan out across all shards.** `SELECT * WHERE created_at > X` must hit every shard.
- **Resharding is painful** — changing N changes `% N` for nearly every key. Mitigated by **consistent hashing** (next example).
- Locality is gone — adjacent keys land in random shards.

### Used by

- MongoDB sharded collections (hashed shard key).
- DynamoDB partition key.
- Most caches (Redis Cluster uses CRC16 hash slots, conceptually `hash % 16384`).

### When this wins

- You **query by key** (user_id lookups, get-by-id).
- You have **monotonic key sequences** (timestamps, IDs) that would hotspot under range partitioning.

## 3. Directory (or "lookup-table") partitioning

A separate service or table tells you which partition holds a given key.

```
key "alice" → directory: "alice → shard B"
                                    ↓
                              fetch from B
```

### Pros

- **Maximum flexibility** — any rule you can express in the directory.
- Easy to **rebalance** — move data, update the directory entry.
- Allows mixing strategies (some keys range-partitioned, others hash-partitioned).

### Cons

- **Extra hop** for every lookup.
- **Directory becomes a SPOF / bottleneck** — must be HA, sharded, cached aggressively.
- More moving parts; harder to reason about.

### Used by

- YouTube's Vitess (logical → physical mapping for MySQL shards).
- Many internal systems at large companies (Foursquare, early Instagram).

### When this wins

- You need **non-uniform partitioning rules** (e.g., specific large customers on dedicated shards).
- You're consolidating multiple legacy DBs behind a single API.

## Choosing a partition key

The hardest part of sharding is **picking the right key**. Get it wrong → expensive to fix.

### Good partition keys

- **Have high cardinality** (many possible values).
- **Are accessed together with the data they partition** (used in the WHERE clause of most queries).
- **Distribute load evenly** (no "celebrity" values with disproportionate traffic).

### Bad partition keys

- **Low cardinality** (e.g., country code with 95% US — Americans all on one shard).
- **Monotonic** (timestamp, auto-increment ID) → hotspots under range partitioning.
- **Not in your common queries** → every query becomes a scatter-gather.

### Examples

| App                     | Reasonable shard key                   |
|-------------------------|----------------------------------------|
| Twitter (tweets)        | `user_id` (a user's tweets co-located) |
| Uber (rides)            | `city_id` or `H3 geo cell`             |
| Slack (messages)        | `channel_id`                           |
| GitHub (repos)          | `repo_id`                              |
| E-commerce (orders)     | `user_id` for OLTP; `shop_id` for analytics |

## Multi-key queries: the scatter-gather

A query that can't be served by one shard becomes:

```
1. Query all N shards in parallel.
2. Each returns a partial result.
3. Coordinator merges.
```

Cost: N× the load, latency bounded by the slowest shard.

Mitigate: aggressive use of indexes (sometimes "global secondary indexes"), denormalization, async views (CQRS).

## Rebalancing

What happens when you add/remove a shard?

- **Range**: split a range across two new shards. Migrate data for half the range.
- **Hash with modulo**: every key potentially changes shard. Disaster.
- **Hash with consistent hashing** (next example): only ~1/N of keys move.
- **Directory**: update directory entries to point at the new shard. Migrate data.

A **virtual-node / range-based** approach with auto-splitting (CockroachDB, Spanner) is what state-of-the-art systems use.

## Architect's takeaway

- **Pick the partition strategy based on dominant query pattern**: by-key → hash, by-range → range, complex → directory.
- **The partition key is more important than the strategy.** A good key on a mediocre strategy beats the reverse.
- **Avoid monotonic keys under range partitioning.** Hotspot guaranteed.
- **Cross-shard queries are expensive.** Design schemas so 95% of queries hit one shard.
- **Plan for rebalancing on day one.** Tools that auto-split or use consistent hashing dramatically reduce future pain.
