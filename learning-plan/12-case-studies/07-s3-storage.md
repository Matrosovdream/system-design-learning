# Case Study 07 — Design an S3-like object storage system

A massively distributed, eventually-consistent, durable blob store. Probes your ability to design for scale measured in **exabytes** and reliability measured in **eleven nines (99.999999999%)** of durability.

## Problem statement

Build object storage that supports:
- Upload / download arbitrary-sized objects (typically blobs).
- Buckets for organization.
- Multipart upload for large files.
- High durability (no data loss).
- Low cost per GB.
- Global access.

## Clarifying questions

1. **Scale**: total objects, storage volume, ops/sec.
2. **Object sizes**: small files (KBs)? Huge files (TBs)?
3. **Read vs write**: read-heavy or write-heavy?
4. **Consistency model**: strong or eventual?
5. **Geographic**: single region or multi-region?
6. **Versioning**: keep old versions of an object?
7. **Lifecycle policies**: tiered storage, expiration?

**Assumed answers:**

- 100B objects, 1 EB (exabyte) total storage, 1M ops/sec peak.
- Object sizes from KB to multiple TB.
- Read-heavy (5:1).
- Read-after-write strong consistency (matches modern S3).
- Multi-region.
- Versioning supported.
- Lifecycle policies for archive/expire.

## Functional requirements

- PUT, GET, DELETE objects (with metadata).
- Multipart upload (for large objects).
- Bucket management.
- Versioning.
- Pre-signed URLs (time-limited access).
- Lifecycle / archive tiers.

## Non-functional requirements

- **Durability: 99.999999999%** ("11 nines") — losing 0.000000001% of objects per year.
- Availability: 99.99%.
- Read latency p95 < 100 ms (excluding network).
- Throughput: arbitrary (GB/s on multipart).
- Cost: low $/GB.

## Capacity estimation

```
1 EB = 1,000,000 TB.

Per node: 200 TB raw storage → need 5,000 storage nodes.
Replication factor 3: 15,000 storage units worth → need 15,000 storage nodes minimum.
   Or with erasure coding (more efficient): less, see below.

Metadata:
  100B objects × ~1 KB metadata each = 100 TB metadata. Significant!
  Stored in a sharded metadata DB.

Ops/sec:
  1M ops/sec average; peak 5M+.
```

## High-level architecture

```
[client]
   │
   ▼
[API gateway / LB]
   │
   ├─► [Authentication / authorization service]
   │
   ▼
[Frontend service]
   │
   ├─► [Metadata service]
   │       ↓
   │   [Metadata DB: sharded, e.g., Spanner-class or sharded Postgres]
   │   bucket_name → bucket info
   │   bucket_name + key → object info (versions, locations, ...)
   │
   ├─► [Storage placement]
   │   decides which storage nodes hold the chunks
   │
   └─► [Storage nodes — 15,000+]
       Each holds many "chunks" (a chunk is a fixed-size piece of an object,
       typically 4-64 MB).
       Replicated or erasure-coded across nodes / racks / DCs.

[Background services]
  - Garbage collector (deletes unreferenced chunks)
  - Repair service (replaces lost replicas)
  - Lifecycle worker (moves cold data to archive tier)
  - Anti-entropy / checksums (verify durability)
```

## Deep dive: chunking and storage

Large objects aren't stored as one blob. They're split into **chunks** of (say) 64 MB.

```
Upload "video.mp4" (5 GB):
  1. Frontend assigns object ID, version ID.
  2. Splits into ~80 chunks of 64 MB.
  3. For each chunk:
     - Compute storage locations (3 replicas across racks/DCs).
     - Stream the chunk to those nodes.
     - Each node returns checksum on store.
  4. After all chunks stored: write metadata listing (chunk_id, location) tuples.
  5. Return 200 OK.
```

Benefits:
- **Parallel upload/download**: 80 chunks in parallel, each at the network limit.
- **Resumable**: if upload fails mid-way, resume per-chunk.
- **Deduplication possible**: chunk by content hash; same chunk across multiple objects stored once.

## Deep dive: durability via erasure coding

Replication factor 3 = 3× storage cost for 11-nines durability with 2 simultaneous node failures tolerated.

**Erasure coding** (Reed-Solomon): split each chunk into K data shards + M parity shards. Any K out of K+M shards can reconstruct the chunk.

Common: (10, 4) coding → 10 data + 4 parity = 14 shards. Tolerate 4 simultaneous failures. Storage overhead: 14/10 = 1.4× (vs 3× for triplication).

Cost saving: 3× → 1.4× = 53% cheaper storage. At exabyte scale, **huge** dollars.

Trade-off:
- Reads are more expensive: usually fetch K shards in parallel; if some fail, need to fetch parity and reconstruct.
- Writes are more expensive: more shards to compute and place.

Most real systems use replication for hot/recent data, erasure coding for cold data (lifecycle migration).

## Deep dive: metadata service

The most-complex part. The metadata DB stores:
- Bucket info (owner, region, ACLs, lifecycle policies).
- Object info per (bucket, key, version): chunk list, content hash, size, ACLs, created_at, etc.

This is heavily accessed: every GET requires a metadata lookup.

### Sharding the metadata

By bucket: hash on `bucket_name`. Each shard handles its buckets' metadata.
- Pros: a single bucket's keys stay together → range scans cheap.
- Cons: a popular bucket overloads its shard.

By (bucket, prefix): finer partitioning.

S3's actual story: very sophisticated, including auto-splitting of hot partitions, multi-region replication of metadata. The metadata service is itself a sub-architecture.

### Strongly consistent metadata

S3 (since 2020) is strongly consistent. To achieve this, every PUT first commits to the metadata service (a quorum write); the metadata is the source of truth. Storage nodes are eventually consistent with it.

GET resolves: metadata says "object X has chunks A, B, C at locations L1, L2, L3". Even if a chunk's replica is briefly stale, the metadata is canonical.

## Deep dive: multipart upload

For files > 100 MB, multipart upload:

```
1. POST /multipart      → returns upload_id
2. PUT  /multipart/upload_id/part/1   (5+ MB)
   PUT  /multipart/upload_id/part/2
   ...
3. POST /multipart/upload_id/complete  with manifest of part ETags
   → metadata service assembles, writes object with all parts as chunks.
```

Benefits:
- Parallel.
- Resumable.
- Each part is a chunk.

The "complete" step is the only place where an object becomes visible (atomic publish).

## Deep dive: read path

```
GET /bucket/object:

1. Frontend authenticates.
2. Frontend asks metadata service: GET (bucket, key, version=current)
3. Metadata returns chunk list with locations.
4. Frontend streams chunks from storage nodes, in order, to the client.
   - Parallelism for large objects.
   - Failover to replicas if one storage node is slow.
   - Reconstruct from parity if erasure coded and shards missing.
```

Latency for small object: ~1-2 metadata RTTs + 1 storage RTT ≈ 5-20 ms.
Latency for huge object: starts streaming after first chunk fetched; bandwidth-bound after that.

## Deep dive: durability mechanisms

To achieve 11 nines:

### 1. Replication / erasure coding

Survive N simultaneous failures.

### 2. Background scrubbing

Periodically read all chunks, recompute checksums, compare to stored hash. Detect silent corruption (bit rot).

### 3. Repair on detected failures

When a node dies: a "repair service" identifies all chunks that lost a replica. It schedules re-replication from other replicas onto other nodes.

The faster repair is, the lower the multi-failure exposure window. AWS, Google heavily optimize repair speed.

### 4. Geographic redundancy

Cross-region replication (optional, per-bucket). Multiple replicas in multiple regions.

### 5. Diversity

Replicas in different racks, different power supplies, different failure domains. So one rack failure doesn't take out all replicas.

## Deep dive: versioning and deletes

With versioning on:
- Each PUT creates a new version (doesn't overwrite).
- DELETE creates a "delete marker" (a special version).
- GET on current version returns the most recent (or delete marker → 404).
- GET with explicit version returns that version.

Behind the scenes:
- Each version has its own object metadata row.
- Chunks aren't immediately deleted; they're referenced by the version.
- Garbage collector deletes chunks once no version refers to them (after lifecycle policy expires the version).

## Deep dive: lifecycle policies

```yaml
lifecycle:
  - filter: prefix=logs/
    transition:
      - days: 30
        storage_class: standard-ia    # infrequent access, cheaper
      - days: 90
        storage_class: glacier         # archive, much cheaper, slower retrieval
    expiration:
      days: 365
```

A lifecycle worker (cron-style or stream-based) periodically scans, applies policies, moves chunks between storage tiers (tier choice changes the underlying erasure code / placement).

This is the source of S3's revenue model: customers store cold data cheap, retrieve rarely.

## Trade-offs discussion

### Why not just a global filesystem?

POSIX semantics are too strong: directory locks, ordered writes, etc. Object storage is **immutable** (overwriting a key creates a new version; you can't append). That immutability is what enables the massive parallelism and erasure coding.

### Replication vs erasure coding

Replication: simple, fast reads, expensive storage.
Erasure: cheaper storage, more compute on reads.

Real systems: replication for hot tier, erasure for cold tier. Lifecycle migrates.

### Strong vs eventual metadata consistency

S3 used to be eventually consistent (LIST after PUT might be stale). In 2020 it became strongly consistent — a years-long engineering effort. Modern designs default to strong.

### Path / key naming and partition hotspots

If many clients write to the same prefix (e.g., `logs/2026/05/27/...`), all writes hit one metadata shard. Recommended: randomize prefixes. AWS S3 historically had this constraint (now mostly fixed by auto-splitting).

## Common follow-up questions

1. **"How do you guarantee 11-nines durability?"**
   - Replication or erasure coding.
   - Diverse placement (rack, AZ).
   - Fast detection and repair of lost replicas.
   - Background scrubbing.
   - Geographic replication for multi-region durability.

2. **"How do you handle a corrupt storage node?"**
   Background checksum scrub detects. Repair service kicks off. The node may be quarantined and ultimately retired.

3. **"How do you scale metadata?"**
   Shard by (bucket, key-prefix). Auto-split hot partitions. Replicate metadata DB across regions for global consistency (Spanner-class or similar).

4. **"How do you handle hot objects (a viral video)?"**
   Cache the object at the edge (CDN — CloudFront in AWS terms). Most reads never hit the storage layer.

5. **"How do you bill?"**
   Storage = average GB-month per tier × $/GB.
   Requests = number × $/request.
   Bandwidth = GB out × $/GB.

   Billing is its own complex system that consumes audit logs from the data path.

6. **"What about read-after-write consistency?"**
   The metadata write is the strong-consistency point. Once the metadata commits, any reader sees the new object — because metadata says so, even if a storage replica is briefly behind.

7. **"How do you handle the 1 TB file upload over a flaky connection?"**
   Multipart upload. Client tracks which parts succeeded. Retries failed parts. The "complete" step assembles. Resumable.

8. **"How would you handle pre-signed URLs?"**
   The frontend signs `(bucket, key, expires_at, method)` with an HMAC key. The URL contains the signature. On request, verify the signature. No DB lookup. The storage service then proceeds normally.

## Key takeaways

- **Object storage = metadata service + chunked, replicated/erasure-coded storage.**
- **Chunking enables parallelism and resumability.**
- **Erasure coding cuts storage costs at scale.**
- **Metadata is the strongly-consistent core; storage is eventually consistent.**
- **Durability comes from replication + diversity + fast repair + scrubbing.**
- **Lifecycle policies migrate data to cheaper tiers automatically.**
- **The architecture is conceptually simple but the engineering is staggering** at exabyte scale.
