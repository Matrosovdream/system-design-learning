# Example 01 — The lifecycle of an HTTP request

You type `https://twitter.com` and press Enter. What happens before you see the homepage?

## The 8 steps

```
[browser] → [DNS] → [TCP handshake] → [TLS handshake] → [HTTP request]
                                                                ↓
                                                         [server processing]
                                                                ↓
[render] ← [browser parse] ← [HTTP response] ← [DB / cache / business logic]
```

### 1. DNS resolution (~20-120 ms cold, ~0 ms cached)

Browser asks the OS: "what's the IP of `twitter.com`?". OS asks the local DNS resolver. Resolver may cascade to root → TLD → authoritative nameservers. Result is cached for **TTL** seconds.

> Cold DNS lookups can dominate latency on the first visit. Once cached, it's free.

### 2. TCP handshake (~1 RTT ≈ 10-150 ms)

Three packets: `SYN` → `SYN-ACK` → `ACK`. After this, both sides agree on sequence numbers and can exchange data.

> One round-trip. If the server is in another continent (~150 ms RTT), this step alone is 150 ms.

### 3. TLS handshake (~1-2 RTTs)

Browser and server agree on cipher, exchange certificates, derive session keys. TLS 1.3 = 1 RTT, TLS 1.2 = 2 RTTs.

> This is why HTTPS feels slower than HTTP on the first connect. Once the connection is open, both are equally fast.

### 4. HTTP request sent (~0 — it's just bytes)

The browser sends:
```
GET / HTTP/1.1
Host: twitter.com
Cookie: session=abc123
...
```

### 5. Server processing (~1-500 ms)

This is the part *you* design as an architect:
- App server picks up the request.
- Authenticates the cookie (possibly hitting Redis).
- Fetches the user's feed (DB query, cache, fan-out service...).
- Renders HTML or returns JSON.

> 90% of your career as an architect is spent making step 5 fast and reliable.

### 6. HTTP response sent

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 42137
Cache-Control: private, max-age=60
...
<html>...</html>
```

### 7. Browser parses & renders (~50-500 ms)

HTML → DOM, CSS → CSSOM, JS evaluated, layout, paint. Browser fetches sub-resources (JS, CSS, images, fonts), each potentially repeating steps 1-6.

### 8. The page is interactive

## Where time actually goes

For a typical request from a US user to a US server:

| Step                | Cold cache       | Warm cache       |
|---------------------|------------------|------------------|
| DNS                 | ~50 ms           | 0 ms             |
| TCP                 | ~30 ms           | 0 ms (kept alive)|
| TLS                 | ~30 ms           | 0 ms (resumed)   |
| Request travel      | ~15 ms           | ~15 ms           |
| Server processing   | **~100 ms**      | **~100 ms**      |
| Response travel     | ~15 ms           | ~15 ms           |
| Browser render      | ~200 ms          | ~50 ms           |
| **Total**           | **~440 ms**      | **~180 ms**      |

## Architect takeaways

- **DNS caching, keep-alive, TLS session resumption are free latency wins** — you usually configure them, not build them.
- The single biggest controllable variable is **server processing time** (step 5). Make this fast → user perceives fast site.
- Adding a CDN moves steps 1-7 from "browser ↔ origin" to "browser ↔ nearest edge", cutting RTTs dramatically. This is why CDNs are the #1 cheap latency win.
- Every extra round-trip = at least one RTT. Designs that need 5 round-trips to load a page are doomed on bad networks.
