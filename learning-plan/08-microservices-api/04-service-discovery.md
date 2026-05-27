# Example 04 — Service discovery: finding services in a dynamic world

In a static world: service A calls service B at `10.0.0.5:8080`. Done.

In a dynamic world (containers, autoscaling, rolling deploys): service B's instances come and go every minute. Pinning an IP is wrong. Service discovery answers the question: **"What's the current set of healthy instances for service B?"**

## Two main patterns

### Client-side discovery

The client knows about the discovery service. It asks "where's service B?", gets back a list of instances, picks one (load-balancing client-side), calls directly.

```
[client] → [discovery (Consul, etcd)] → "service B at: 10.0.0.5, 10.0.0.6, 10.0.0.7"
[client] → picks one (round-robin / least-conn / weighted) → calls 10.0.0.5:8080
```

**Pros**
- No extra hop — direct connection.
- Client can implement smart LB (consistent hashing, circuit breaker).

**Cons**
- Every client (Go, PHP, etc.) needs a discovery client library.
- More logic in the application code.

**Used in**: Netflix Eureka + Ribbon, Finagle, Polaris.

### Server-side discovery

The client just calls a stable address (DNS name or load balancer). Discovery happens behind that address.

```
[client] → http://service-b.internal/  → [LB or DNS] → picks instance → 10.0.0.5:8080
```

**Pros**
- Clients are dumb — just use a name.
- Cross-language naturally.
- The LB is the discovery point.

**Cons**
- Extra hop (sometimes).
- Discovery and LB are coupled.

**Used in**: Kubernetes Services (DNS-based), AWS ALB, GCP internal LB.

## DNS-based discovery: the workhorse

The simplest server-side approach: a DNS name resolves to all healthy instances of a service.

```
service-b.internal → [10.0.0.5, 10.0.0.6, 10.0.0.7]
```

When an instance dies, DNS is updated. New instances are added on rollout.

### Pros

- **Universal.** Every language and platform speaks DNS.
- **Stable name** clients can hardcode.

### Cons

- **DNS caching is the enemy.** A client might cache an IP for the TTL. If the IP died yesterday, your client is calling a corpse.
- **No load balancing** beyond round-robin (and even that depends on client behavior).
- **Health checks** are at the DNS-record level, often slow to converge (seconds to minutes).

Kubernetes pushes DNS hard because it's universal, but real production setups add a service mesh on top to handle the smarter parts.

## Kubernetes services: how they actually work

In Kubernetes:

1. You deploy `pods` (containers).
2. You define a `Service`: a stable DNS name + virtual IP that selects pods by label.
3. The `kube-proxy` on each node sets up iptables/IPVS/eBPF rules to forward traffic from the virtual IP to a healthy pod.
4. Clients call `http://service-b.namespace.svc.cluster.local/`.

The mechanism is transparent. From the client's view, it's just DNS. Behind the DNS, kubernetes is dynamically rewriting routes as pods come and go.

## Service mesh: the modern discovery + reliability layer

A service mesh (Istio, Linkerd, Consul Connect) takes service discovery further. Each service gets a **sidecar proxy** (Envoy) that handles:

- **Discovery**: knows the current set of instances for every service.
- **Load balancing**: configurable algorithms.
- **mTLS**: encrypted, authenticated service-to-service communication.
- **Retries, timeouts, circuit breakers**: configured centrally, applied per call.
- **Observability**: traces, metrics, logs for every call.

```
[service A's sidecar] ──mTLS──► [service B's sidecar] → service B process
```

The application code just calls `http://service-b/`. The sidecar handles everything else.

### Pros

- **Cross-language, no library changes.** Works for Go, PHP, Java, anything.
- **mTLS by default** — every service-to-service call is encrypted and authenticated.
- **Centrally configurable** — change timeouts via config, no code redeploy.
- **Observability is automatic.**

### Cons

- **Operational complexity.** Running Istio is a real undertaking.
- **Resource overhead** — every service runs an extra container.
- **Failure modes are new** — the mesh control plane can have its own bugs.

**Use a service mesh** when: 10+ services, you need mTLS, the team can operate it.
**Skip a service mesh** when: small system, simple traffic patterns, you have a good API gateway already.

## Health checking

Discovery is only as good as **knowing what's healthy**.

Levels of health checks:

- **TCP open**: port accepts connection. Cheapest, often wrong (process up but app broken).
- **HTTP `/healthz`**: app responds. Good baseline.
- **HTTP `/readyz`**: app says "I'm ready to serve traffic" (distinct from "I'm alive"). Used in k8s.
- **Deep health**: app checks its DB, cache, downstream. Heavy but truthful.

Discovery system continuously polls these. Failing instances are removed from rotation.

Important: **don't let health checks cause cascading failures**. If service B's `/healthz` checks the DB, and the DB hiccups, every B instance is marked unhealthy, the LB has nothing to send to, more retries hit the DB. Use **shallow** health for in-rotation, **deep** for monitoring/alerting.

## Failure modes worth knowing

### Stale discovery

A dead instance is still in the registry. Clients retry it before it's removed. Mitigate with:
- Short health-check intervals (e.g., every 2 seconds).
- Aggressive removal on consecutive failures.
- Client-side circuit breakers — after N failures to a specific IP, skip it.

### Discovery system outage

If Consul/etcd is down, what happens? Best practice:
- **Cache the last-known list** locally and continue using it.
- **Don't refuse all calls** because discovery is down. Fail open with stale data — better than total outage.

### Partial network partition

Some clients can reach an instance; others can't. Discovery doesn't know that. **Gray failures** — covered in step 09 (observability).

## A choose-one menu by setup

- **Bare VMs / no Kubernetes**: Consul or etcd for registry; client-side LB or HAProxy.
- **Kubernetes**: K8s Services (DNS + kube-proxy) by default. Add Istio/Linkerd for mesh.
- **AWS**: ALB / NLB with target groups; ECS Service Discovery; AWS App Mesh for mesh.
- **GCP**: GCP internal LB or service mesh via Anthos.
- **Hybrid**: Consul (it spans clouds and on-prem nicely).

## Architect's takeaway

- **You will need service discovery** the moment instances become ephemeral.
- **DNS-based discovery is the universal baseline**, with Kubernetes Services as the most common implementation.
- **Service mesh** adds mTLS, retries, observability, and circuit breakers — at the cost of operational complexity. Adopt when org scale demands it.
- **Health checks must be carefully designed** to avoid cascading failures.
- **Discovery system outage shouldn't take down everything** — local caching + fail-open behavior are essential.
