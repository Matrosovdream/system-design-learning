# Example 04 — CDN: the cheapest latency win in distributed systems

A **CDN** (Content Delivery Network) is a globally distributed cache. Your content is replicated to **edge servers** in dozens or hundreds of cities. A user request hits the nearest edge, not your origin.

Examples: Cloudflare, Akamai, Fastly, AWS CloudFront, Google Cloud CDN.

## The two things a CDN does

### 1. Cut latency by getting physically closer to the user

A user in Sydney requesting an asset stored in Virginia: ~200 ms RTT.
Same user requesting it from a Sydney edge node: ~5 ms RTT.

**40× lower latency, with no app changes.** The bytes are identical; they just live closer.

### 2. Absorb bandwidth at the edge

If 10M users download a 5 MB JS bundle, that's 50 TB of bandwidth. Without a CDN, **all of it** hits your origin servers and your transit bills.

With a CDN, 99%+ of that traffic is served from edge caches. Your origin sees a few hundred requests, the rest are cache hits at the edge.

## How it actually works

```
user → DNS → CDN-resolves-to-nearest-edge → edge server

edge server: do I have this URL cached?
   yes  → serve from cache (≈5 ms)
   no   → fetch from origin, cache it, serve it (first request only)
```

**Cache key** is usually the full URL (plus optionally headers like `Accept-Language`). Cache TTL is set by your `Cache-Control` headers.

## What to put on a CDN — and what not to

### Good candidates (high cache hit rate)

- **Static assets**: images, CSS, JS bundles, fonts, videos.
- **Public API responses with TTL**: e.g., `/api/products?category=shoes` cached for 60 seconds.
- **Software downloads, binaries, installers**.
- **Map tiles, ML model files, generic ML responses**.

### Bad candidates (low or zero cache hit rate)

- **Per-user content** (your dashboard, your inbox). The cache key would be unique per user.
- **Real-time data** (live scores, stock prices) — except for very short TTLs.
- **Write requests** (POST, PUT, DELETE). CDNs forward these to the origin; caching doesn't apply.

You **can** still use a CDN as a TLS-terminating reverse proxy for dynamic content (Cloudflare does this) — the latency win there comes from cheaper TLS handshake closer to the user, not from caching.

## Cache control: the headers that matter

```
Cache-Control: public, max-age=86400         ← cache anywhere for 1 day
Cache-Control: private, max-age=60           ← only browser cache (not CDN), 60s
Cache-Control: no-store                      ← never cache
Cache-Control: max-age=3600, s-maxage=86400  ← browser 1 hr, CDN 1 day
ETag: "abc123"                               ← validator; CDN can revalidate
```

`s-maxage` is "shared cache age" — applies to CDNs/proxies. This lets you cache longer in the CDN than in the browser.

## Cache invalidation: the hard part

Once content is at 200 edge nodes globally, **changing it** is painful. Two strategies:

### Versioned URLs (best)

Include a hash in the filename:
```
/static/app.5f3a91.js
/static/styles.92b7c4.css
```

When the file changes, the URL changes → fresh fetch. Old URLs naturally age out.

This is how every modern frontend build system works (Webpack, Vite, Rollup → hashed filenames).

### Explicit purge

Tell the CDN "evict this URL" via API. Cloudflare, CloudFront, Fastly all support this. Slower (seconds to minutes for global propagation), but necessary for content that can't have a versioned URL (e.g., `/index.html`).

## Worked example: a Twitter-like service uses a CDN for…

| Asset                      | CDN?  | TTL              | Notes                              |
|----------------------------|-------|------------------|------------------------------------|
| `/static/app.<hash>.js`    | Yes   | 1 year (immutable)| Versioned filename                |
| `/static/logo.png`         | Yes   | 7 days           | Versioned via query string         |
| User avatar `/avatars/123.jpg` | Yes | 1 hour         | Re-uploads are rare                |
| Tweet images `/media/<id>.jpg` | Yes | Forever       | Immutable post-upload              |
| `/api/feed?user=alice`     | No    | —                | Per-user, can't cache              |
| `/api/trending`            | Yes   | 60 seconds       | Public, fresh-enough, huge fanout  |
| `POST /api/tweet`          | No    | —                | Write — passes through to origin   |

The CDN absorbs **most bytes** on this list (avatars + media), even though only a few endpoints are cacheable.

## Architect's takeaway

- **A CDN is the single highest-ROI infrastructure addition** for any user-facing product. Latency drops for global users, origin bandwidth drops dramatically.
- **Treat cacheability as a design property** — design URLs and headers from day one so a CDN can actually cache them.
- **Versioned URLs** are vastly better than purge-based invalidation. Make your builds emit hashed filenames.
- **Cache misses go to origin** — your origin must still be able to handle the long tail. CDN amplifies origin capacity, doesn't replace it.
- **Combine CDN with image resizing / format negotiation** (Cloudflare Images, ImageKit, imgproxy) — one upload, many edge variants (AVIF, WebP, sizes).
