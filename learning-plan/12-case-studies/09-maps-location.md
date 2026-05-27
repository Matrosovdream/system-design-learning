# Case Study 09 — Design a maps / location service (Google Maps, Uber)

A geography problem: find things nearby, fast. Probes your knowledge of **spatial indexing**, real-time updates, and serving global users with low latency.

## Problem statement

Build a service that:
- Stores points of interest (POIs): restaurants, shops, addresses, places.
- Tracks moving entities (driver locations, sharing-app users) in real time.
- Answers "what's near me?" / "show all X within Y km of point P".
- Routes from A to B (out of scope; focus on storage and search).

## Clarifying questions

1. **What entities?** Static POIs only? Dynamic (e.g., drivers)? Both?
2. **Scale**: number of POIs, number of moving entities, queries/sec.
3. **Update frequency**: drivers update every 5s?
4. **Query types**: nearest-N, within-radius, within-bounding-box?
5. **Routing or just search?**
6. **Map rendering** (tiles, vectors) or just data API?

**Assumed answers:**

- Static POIs (100M) + dynamic drivers (1M live).
- 1M queries/sec peak.
- Drivers send updates every 5 seconds.
- Both nearest-N and within-radius queries supported.
- Routing out of scope.
- Map tiles out of scope (covered by CDN already).

## Functional requirements

- Store POIs with lat/long + attributes.
- Real-time location updates from drivers (or any moving entity).
- Geospatial queries:
  - Nearest N entities to a point.
  - All entities within a radius / bounding box.
  - Filter by attributes (e.g., "active drivers" only).

## Non-functional requirements

- Query latency p95 < 100 ms.
- Driver location updates: write QPS ~200k/sec (1M drivers / 5s).
- Read QPS up to 1M/sec.
- Global users, low latency.
- High availability.

## Capacity estimation

```
Driver updates: 1M / 5s = 200k writes/sec. Steady.
Reads: 1M/sec peak (rider apps).

Storage:
- POIs: 100M × ~500 bytes = ~50 GB. Easy.
- Driver locations: 1M × 200 bytes = 200 MB. Trivial.
- Historical driver trails: 1M × 17,280 updates/day × 50 bytes = ~860 GB/day. Significant if kept long.
```

## High-level architecture

```
[rider app]                    [driver app]
   │                                │
   │  search query                  │  location update (every 5s)
   ▼                                ▼
[API gateway]                  [Location ingestion service]
   │                                │
   │                                ▼
   │                          [Kafka: locations.updated]
   │                                │
   │                                ▼
   │                          [Location service]
   │                                │
   │                          ┌─────┴─────────────┐
   │                          ▼                   ▼
   │                  [Hot store: Redis]   [Persistent: time-series DB]
   │                  (current loc per     (historical trails)
   │                   driver, expires
   │                   in 30s)
   ▼
[Search service]
   │
   ├─► [Spatial index: H3 or Geohash]
   │   in-memory or Redis
   │
   └─► [POI database: PostGIS / Elasticsearch]
       (static POIs, with attribute filters)
```

## Deep dive: spatial indexing — the core problem

How do you efficiently answer "what's within radius R of point P"?

A naive scan: check distance to every entity. With 1M drivers and 1M queries/sec, that's 10^12 distance calculations/sec. Impossible.

You need an index that supports **proximity** queries efficiently.

### Geohash

Encode (lat, long) as a string. Nearby points share prefixes.

```
Berlin:   u33d8        # u33d nearby
Paris:    u09tu        # u09 nearby
SF:       9q8yy
```

Lookup "within ~5 km": search by prefix length. Truncate to first N characters, find all matches.

Pros: simple, sortable strings, works on any KV store.
Cons: rectangular cells (not great for circular queries near corners). Edge cases at boundaries.

Used by: Redis geo commands (under the hood), older Foursquare, MongoDB geospatial.

### H3 (Uber's hexagonal grid)

Tile the globe into hexagons at multiple resolutions. Each hex has an ID.

```
H3 index "8928308280fffff" identifies a specific hex.
Resolution 0 = continent-sized. Resolution 15 = ~1 m².
```

Hexagons have uniform edge distance to neighbors (better for spatial reasoning than squares).

Pros: clean cell geometry, native to Uber. Powerful built-in operations (compact, uncompact, neighbors).
Cons: less ubiquitous than geohash; H3 library required.

Used by: Uber, modern geo-indexed systems.

### R-tree / quadtree / k-d tree

Tree-based spatial indexes. Each node holds a bounding box; children divide it.

Pros: very efficient for various query shapes.
Cons: complex to distribute and update.

Used by: PostGIS, MongoDB, ES (under the hood).

### Recommendation for this design

For driver locations (highly dynamic): **Redis with H3 (or geohash)** for in-memory speed.

For POIs (static): **PostGIS** or **Elasticsearch with geo_point** — mature, supports filtering by attributes.

## Deep dive: driver location updates

```
Driver app → POST /locations every 5 seconds:
  { driver_id, lat, lng, heading, status }

Ingestion service:
  - validates
  - publishes to Kafka "locations.updated"

Location service consumes Kafka:
  - update Redis: key="driver:42", value=(lat, lng, ts), TTL=30s
  - also compute H3 cell for this position, update spatial index:
      key="h3:8928308280fffff", set of drivers in this cell
      add driver to set, remove from old cell
  - optionally write to time-series DB for historical trail
```

Why TTL on the driver location? If the driver app crashes or loses connection, after 30s the driver disappears from search results automatically.

## Deep dive: nearest-N query

```
Rider asks: "show me 10 nearest drivers to (lat, lng)"

Search service:
  1. Compute H3 cell for (lat, lng) at chosen resolution (say resolution 8, ~0.7 km hex edge).
  2. Get the cell + ring of neighbors (e.g., k=2 → cell + 18 neighbors).
  3. For each cell:
     read Redis "h3:{cell}" → set of driver_ids.
  4. Combine into candidate list (~50-500 drivers in a city).
  5. For each candidate, look up Redis "driver:{id}" → exact position.
  6. Compute Haversine distance to (lat, lng).
  7. Sort, take top 10. Return.
```

Steps 1-3 are O(1) per cell. Step 4-7 is bounded by number of candidates in nearby cells — usually small.

Result: a nearest-N query is ~30 Redis ops, ~5 ms.

If a city has 50 active drivers, you might need to expand the search ring; if it has 5000, you can shrink it.

## Deep dive: scaling the spatial index

A city like NYC has 100k drivers; each in a hex. The set for one hex might have a few hundred drivers.

To scale: shard Redis by H3 prefix.

```
shard A: H3 cells starting with 88
shard B: H3 cells starting with 89
...
```

Each shard handles its region. Reads and writes go to the right shard.

For global scale (billions of POIs), shard by larger H3 region.

## Deep dive: POI search with filters

"Show me Italian restaurants within 2 km of me."

Geohash/H3 alone gives you proximity. Filtering by `cuisine = 'italian'` needs another structure.

Two approaches:

### Approach 1: Elasticsearch with geo_point

```
ES doc: { name, cuisine, location: {lat, lng} }
ES query:
  query: { match: { cuisine: "italian" } }
  filter: { geo_distance: { distance: "2km", location: { lat: ..., lng: ... } } }
```

ES handles indexing and querying both attributes and geo. Mature, well-tested.

### Approach 2: PostGIS

```
SELECT * FROM pois
WHERE cuisine = 'italian'
  AND ST_DWithin(location, ST_MakePoint(lng, lat)::geography, 2000);
```

GiST index on `location`, B-tree on `cuisine`, PostGIS optimizes.

For 100M POIs, Elasticsearch tends to be more flexible for diverse filters; PostGIS for transactional updates and joins.

## Trade-offs discussion

### Why Redis for live drivers, not Postgres?

Drivers update every 5 seconds → ~200k writes/sec sustained. Postgres can do it but at higher latency and operational cost. Redis with TTLs is perfect for the "current state" use case.

### Why separate POI store from driver store?

POIs are static, queried with attribute filters, ~100M entries. Drivers are dynamic, queried by proximity only, ~1M entries. Different access patterns → different stores.

### Geohash vs H3

Geohash: simple, available in Redis natively (`GEOADD`, `GEORADIUS`). Choose for simplicity.

H3: better proximity semantics, popular in newer systems. Choose for sophistication.

For an interview answer: either is correct; explain why you picked it.

### Historical trails

Every driver's location every 5s = ~17k records per driver per day. Compressed and stored in a time-series DB (TimescaleDB, InfluxDB) or Parquet on S3 for ad-hoc analysis. Not needed for live queries.

## Common follow-up questions

1. **"How do you handle a driver who teleports (GPS jump)?"**
   Server-side smoothing: discard updates that imply > X km/h. Validation, not just acceptance.

2. **"How do you support 'shareable' tracking (rider sees driver moving)?"**
   WebSocket / SSE channel per ride. Driver's update → publish to a per-ride topic. Rider's app subscribes. Same Kafka backbone, different routing.

3. **"What if a city has 100k drivers and a rider queries with no filters?"**
   You'd return tens of thousands of drivers. Cap at N (e.g., 50). Use the resolution-aware search: shrink the cell size to limit candidate count.

4. **"What if Redis is down?"**
   Drivers' positions in the queue stack up. Reads start failing. Need HA Redis (cluster with failover). For graceful degradation: maybe serve cached "last known good" snapshot for a few minutes.

5. **"How do you handle map tiles?"**
   Separate problem. Pre-rendered raster or vector tiles, served via CDN. Updates are rare and batched. Not part of the live location system.

6. **"How do you match a rider to a driver (the assignment problem)?"**
   That's the dispatch service. Reads nearest drivers, runs an assignment algorithm (Hungarian / greedy / ML-based), assigns. Different module; uses the location service as a dependency.

7. **"How do you scale globally?"**
   - Redis sharded by H3 region.
   - Multi-region deployment: each region serves its own data.
   - Cross-region rare (your rider in NYC is unlikely to be matched with a driver in Berlin).

## Key takeaways

- **Spatial indexing (geohash / H3) is the crux.** Naive scanning doesn't work.
- **Two separate stores**: hot in-memory for live entities (Redis), persistent for POIs (Postgres/ES).
- **Driver updates via streaming pipeline**: Kafka decouples ingestion from indexing.
- **TTLs on driver locations** auto-handle disconnected drivers.
- **Filter + geo queries**: ES or PostGIS.
- **Don't keep historical trails in the hot store** — separate time-series DB.
- The geography problem reduces to: **bucket by space, query buckets, refine**.
