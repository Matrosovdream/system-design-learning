# Example 03 — API gateway: the front door pattern

The **API gateway** is a single entry point in front of many backend services. It handles cross-cutting concerns at the edge so each service doesn't reinvent them.

You can think of it as "a reverse proxy that knows about your services and your auth model".

## What the gateway does

```
[client] → [API gateway] → [service A]
                       ├──► [service B]
                       └──► [service C]
```

Typical responsibilities:

1. **TLS termination** — one cert at the edge.
2. **Authentication and authorization** — validate JWT, check OAuth scopes, attach user identity to backend calls.
3. **Rate limiting** — per IP, per user, per API key, per endpoint.
4. **Routing** — `/api/orders/*` → orders service; `/api/payments/*` → payments service.
5. **Request shaping** — header normalization, request/response transformation.
6. **Aggregation** — fan out one client request to multiple backends, combine responses.
7. **Caching** — short-TTL caches for hot endpoints.
8. **Logging, metrics, tracing** — one consistent place to observe the edge.
9. **Throttling and circuit breaking** — protect downstream services from overload.
10. **API versioning** — `/v1/*` vs `/v2/*` route to different backends or transform.

## Why have one

Without a gateway, every service has to:
- Validate JWTs.
- Implement rate limiting.
- Handle CORS.
- Terminate TLS (or each is behind its own proxy).
- Log uniformly.

Result: scattered, inconsistent, often subtly wrong. With a gateway, **cross-cutting concerns are centralized.** Services focus on business logic.

## Gateway vs reverse proxy

A reverse proxy (Nginx, Envoy) handles HTTP-level concerns: routing, TLS, basic load balancing.

An API gateway adds **API-aware concerns**: schema validation, auth via JWT/OAuth, per-route rate limits, per-endpoint quotas, transformations.

In practice, **modern gateways (Kong, Tyk, Apigee, AWS API Gateway, Envoy with config)** can do both. The distinction is fuzzy.

## Gateway as aggregator

A common pattern: gateway makes **one client request fan out** to multiple services and assembles the response.

```
Client: GET /api/dashboard

Gateway:
  parallel:
    user_data ← user-service.get(user_id)
    orders ← order-service.list_recent(user_id, 5)
    notifications ← notification-service.unread(user_id)
  return { user, orders, notifications }
```

This avoids the mobile client making 3 sequential calls. Latency drops from `sum` of 3 to `max` of 3.

But: it puts business logic in the gateway. Can be overdone. The **BFF pattern** (next example) is a more disciplined version of this.

## Where the gateway lives

```
[client] → [CDN] → [WAF] → [load balancer] → [API gateway] → [services]
```

Layers above the gateway (CDN, WAF, LB) handle network-level concerns. The gateway is the **HTTP/API**-level concern. Services are pure business logic.

In Kubernetes / cloud-native setups, the gateway is often **Envoy** (configured via Istio, Ambassador, Emissary, or directly) or **Kong** running as an in-cluster service.

## Popular gateways

| Tool                 | Style                                  | Notes                                   |
|----------------------|-----------------------------------------|-----------------------------------------|
| Kong                 | Open source + enterprise, plugin-based  | Powerful, Lua plugins                   |
| AWS API Gateway      | Managed service                         | Great with Lambda, AWS-native           |
| Apigee (Google)      | Enterprise managed                      | Heavy, expensive, very featureful       |
| Envoy                | Open source proxy + config              | Often the substrate for other gateways  |
| Istio Gateway        | k8s-native (built on Envoy)             | If you have a service mesh anyway       |
| Traefik              | Open source, dev-friendly                | Auto-discovery, great with Docker       |
| Tyk                  | Open source + enterprise                | Solid mid-tier                          |
| Cloudflare API Gateway | CDN-integrated                       | Edge-first model, global                |
| AWS ALB              | L7 load balancer                        | Cheaper than API Gateway for simple routing |

For most cloud-native PHP/Go shops: **Envoy or Kong** locally, **AWS API Gateway / GCP API Gateway** if you're cloud-managed.

## When you don't need a gateway

- **One service**, monolith — your reverse proxy (Nginx) is enough.
- **Service-to-service traffic** inside the same trust boundary — service mesh handles cross-cutting concerns at the **sidecar**, not at a gateway.
- **You want full control** at every service — usually not worth it; gateway saves you a lot.

## Anti-patterns

### The gateway as the new monolith

People keep adding logic to the gateway: auth, rate limit, transformations, response composition, schema validation, business rules. Eventually the gateway is itself a giant codebase with no team owning it.

**Rule**: the gateway is for **cross-cutting concerns**, not business logic. If a piece of logic is specific to one endpoint, it lives in the service, not the gateway.

### The gateway as SPOF

If the gateway is down, **all** of your APIs are down. Mitigations:
- Run multiple gateway instances (always).
- Have the LB above the gateway healthcheck and remove broken ones.
- Test failover.

### Different gateway behavior per environment

If staging and production have different gateway configs, you'll be debugging mysterious traffic shaping in production. Treat gateway config as code, with versioned changes.

## A concrete example: a multi-service app

A small SaaS company:

- 4 backend services: `users`, `orders`, `payments`, `notifications`.
- Mobile + web clients.

```
client
  │
  ▼
[ Cloudflare CDN: static, DDoS protection ]
  │
  ▼
[ AWS ALB: TLS, basic LB ]
  │
  ▼
[ Kong API Gateway:
    - Auth via JWT
    - Rate limit per user
    - Route /v1/users → users, /v1/orders → orders, etc.
    - CORS
    - Request ID injection ]
  │
  ▼
[ services ]
```

Each backend service runs **without** auth code, **without** rate limit code. They trust the gateway's headers (`X-User-Id`, `X-User-Role`) attached after validating the JWT.

This is the standard production layout.

## Architect's takeaway

- **An API gateway centralizes cross-cutting HTTP/API concerns** at the edge.
- **TLS, auth, rate limiting, routing, observability** belong at the gateway, not in each service.
- **Don't put business logic in the gateway.** Use a BFF (next example) for that.
- **The gateway is your perimeter** — secure it accordingly (rate limits, WAF, healthchecks).
- **Pick a gateway based on infra**: Kong/Envoy for self-hosted, AWS API Gateway / GCP for managed.
- **For monoliths, you don't need a gateway** — your reverse proxy already does its job. Don't add complexity until you have multiple services.
