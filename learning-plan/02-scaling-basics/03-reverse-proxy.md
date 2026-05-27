# Example 03 — Reverse proxy: the swiss-army knife of the edge

A **forward proxy** sits in front of clients ("the office firewall"). A **reverse proxy** sits in front of *servers* — it represents your servers to the outside world.

Every production system has one. Usually Nginx, Envoy, HAProxy, Traefik, or Caddy.

## What it does (and why you can't skip it)

```
[client] ── HTTPS ──► [reverse proxy] ── HTTP ──► [your app servers]
                       (Nginx/Envoy)
```

The single box (or cluster) at the edge does **a lot**:

### 1. TLS termination
Decrypts HTTPS once at the proxy. Backend talks plain HTTP, much faster, no per-server cert management.

### 2. Load balancing
Spread requests across N backends using one of the algorithms from example 02.

### 3. Caching
Cache static responses (`/image.jpg`, `/main.css`) right at the proxy. Backends never see the second request.

### 4. Compression
gzip/brotli on the way out. Backends ship uncompressed; proxy adds compression once.

### 5. Rate limiting
"Max 100 req/sec per IP". Defends backends from accidental and malicious bursts.

### 6. Request routing
`/api/*` → API pool. `/static/*` → S3. `/v2/*` → new microservice. Backends don't need to know they're sharing a hostname.

### 7. Header rewriting / canonicalization
Normalize `X-Forwarded-For`, strip internal headers, add tracing IDs.

### 8. Security
Block known-bad IPs, enforce HSTS, drop oversized requests, filter weird HTTP smuggling, basic WAF rules.

### 9. Authentication offload
For simple cases (Basic Auth, JWT validation, mTLS) the proxy can authenticate before the request ever reaches the app.

### 10. Observability
Centralized access logs. One place to see "what's going in and out".

## Minimal Nginx config sketch

```nginx
upstream app_servers {
    least_conn;
    server 10.0.1.10:8080;
    server 10.0.1.11:8080;
    server 10.0.1.12:8080;
}

server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate     /etc/ssl/example.com.pem;
    ssl_certificate_key /etc/ssl/example.com.key;

    gzip on;
    gzip_types text/css application/javascript application/json;

    # Cache static assets
    location /static/ {
        proxy_cache static_cache;
        proxy_cache_valid 200 1d;
        proxy_pass http://app_servers;
    }

    # Rate limit API
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/s;

    location /api/ {
        limit_req zone=api burst=200 nodelay;
        proxy_pass http://app_servers;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        proxy_pass http://app_servers;
    }
}
```

## Reverse proxy vs load balancer — are they the same?

Mostly **yes** in modern stacks. A reverse proxy *includes* load balancing, plus the other 9 features above. A pure load balancer (e.g., AWS NLB at L4) is dumber but faster.

In practice:
- **Cloud-native**: AWS ALB / GCP HTTPS LB = reverse proxy (L7) ≈ what Nginx does.
- **High-throughput edges**: AWS NLB (L4) is just a load balancer; an Nginx/Envoy layer behind it adds the reverse-proxy features.
- **Service mesh** (Istio, Linkerd): Envoy as a sidecar proxy — same idea, applied per-service.

## Why this matters

Without a reverse proxy:
- Every app server runs TLS (cert management nightmare).
- Every app server does compression (CPU waste).
- Static caching lives inside the app (cache invalidation harder).
- Rate limiting is per-server, easy to bypass.
- You can't update LB rules without a deploy.

With a reverse proxy:
- Cert management = one place.
- Compression and caching = free.
- New service rollout = add a `location` block, reload Nginx (no app restart).
- Rate limits, IP blocks, A/B routing = config change, zero code.

## Architect's takeaway

- **Reverse proxy = single most useful piece of infrastructure in front of your app.** Cheap, mature, well-understood.
- **Defaults: Nginx or AWS ALB** for most stacks; **Envoy** for service-mesh and gRPC; **Caddy** for "automatic HTTPS, zero config" simplicity.
- It's the place to put cross-cutting concerns: **TLS, caching, rate limits, routing** — *before* you bake them into application code.
- Don't write authentication, rate-limiting, or routing in your app if the proxy can do it. Less code, more reliable.
