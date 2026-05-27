# Step 08 — Microservices & APIs

By this point you can design the data and runtime substrate. Now: how to carve a system into services, how those services talk, and how to expose them to clients without making them brittle.

## Goals

- Decide whether to do microservices at all (most teams shouldn't, day one).
- Compare REST, gRPC, GraphQL on real criteria, not blog hype.
- Explain what an **API gateway** does and when you need one.
- Describe **service discovery**: client-side vs server-side, with examples.
- Describe **BFF** (Backend for Frontend) and when it earns its keep.
- Draw a clean monolith → microservices migration.

## Key concepts

1. **Monolith vs microservices** — the trade-off, the inflection point.
2. **API styles**: REST (HTTP+JSON), gRPC (HTTP/2+protobuf), GraphQL (query language), webhooks.
3. **API gateway** — the front door: TLS, auth, rate limit, routing, aggregation.
4. **Service discovery** — DNS, Consul, etcd, Kubernetes services, service mesh.
5. **Service mesh** — Istio, Linkerd, sidecar proxies, mTLS between services.
6. **Backend for Frontend (BFF)** — a tailored API per client.
7. **API versioning** — URL versioning, header versioning, content negotiation.
8. **Strangler fig pattern** — incremental monolith decomposition.

## Reading

- **Primer**: "Communication" section (RPC, REST, etc.).
- **Sam Newman**: *Building Microservices* (book) — the canonical reference.
- **Martin Fowler**: blog posts on microservices, strangler fig, BFF.
- **gRPC docs**: design and best practices.

## Examples in this folder

- `01-monolith-vs-microservices.md` — the trade-off and when to switch.
- `02-rest-vs-grpc-vs-graphql.md` — three styles, real differences.
- `03-api-gateway.md` — the front door pattern.
- `04-service-discovery.md` — finding services in a dynamic world.
- `05-bff-pattern.md` — per-client APIs.
- `06-strangler-fig.md` — incrementally killing a monolith.

## Self-check

1. You have a 5-engineer startup. Should you start with microservices? Why or why not?
2. Why is gRPC popular for service-to-service traffic but rare for public APIs?
3. What does an API gateway do that a reverse proxy doesn't (or vice versa)?
4. Why do mobile apps love GraphQL but backend teams have mixed feelings?
5. You inherit a 10-year-old PHP monolith. What's the lowest-risk way to extract a piece into a service?
