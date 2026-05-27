# Step 02 — Scaling Basics

Now you can describe a single request's journey. Next: what to do when one server isn't enough. This step covers the **first** moves you make when traffic grows: scale up, scale out, put a load balancer in front, add a CDN, make the app stateless.

## Goals

- Explain **vertical vs horizontal scaling** and pick correctly given a constraint.
- List 4 load-balancing algorithms and when to use each.
- Explain what a **reverse proxy** is and why every production system has one.
- Explain what a **CDN** does and why it's usually the biggest latency win.
- Define **stateless service** and explain why statelessness enables horizontal scaling.
- Trace the evolution of a system from "1 box" to "geo-distributed at millions of QPS".

## Key concepts

1. **Vertical scaling (scale up)** — bigger box (more CPU/RAM/disk). Easy, cheap to start, has a ceiling.
2. **Horizontal scaling (scale out)** — more boxes. Cheaper at scale, but needs the app to be stateless and the data to be shardable.
3. **Load balancer (L4 vs L7)** — L4 routes by IP/port (fast, dumb). L7 routes by HTTP path/header/cookie (slower, smart).
4. **Load-balancing algorithms** — round-robin, least-connections, weighted, consistent-hash, IP-hash.
5. **Reverse proxy** — sits in front of your servers, terminates TLS, does caching, compression, rate limiting, request routing. Examples: Nginx, HAProxy, Envoy, Traefik.
6. **CDN** — geo-distributed cache. Static content (images, JS, CSS, video) served from the edge close to the user.
7. **Stateless services** — a service that stores no session state on disk/in memory between requests. Any instance can handle any request.
8. **Session affinity ("sticky sessions")** — the *opposite* of stateless. Sometimes necessary, often a smell.

## Reading

- **Primer**: sections on Load Balancing, Reverse Proxy, CDN, Horizontal scaling.
- **Xu V1**: Chapter 1 — *Scale from Zero to Millions of Users*. This chapter is basically this step in book form.

## Examples in this folder

- `01-vertical-vs-horizontal.md` — when to scale up vs out.
- `02-load-balancer-algorithms.md` — RR, least-conn, hash, weighted, with use cases.
- `03-reverse-proxy.md` — why every system has Nginx/Envoy and what it actually does.
- `04-cdn.md` — how CDNs cut latency and bandwidth at the same time.
- `05-stateless-services.md` — why state in app servers is the enemy of scale.
- `06-scaling-story.md` — a single product goes from 1 user to 100M, step by step.

## Self-check

1. You have a PHP monolith handling 5k RPS. CPU is at 80%. What do you scale first — vertical or horizontal? Why?
2. Why does a least-connections LB usually beat round-robin?
3. Your team adds a Redis-backed shopping cart. The cart drops items randomly. Why might "sticky sessions" be a worse fix than "fix the bug"?
4. What's the difference between a load balancer and a reverse proxy? (Trick question.)
5. Why are CDNs more useful for read-heavy than write-heavy workloads?
