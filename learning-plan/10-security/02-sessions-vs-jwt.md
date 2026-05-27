# Example 02 — Sessions vs JWT: the trade-off everyone gets wrong

Two ways to maintain "I'm logged in" between requests. Each has real strengths and weaknesses; "JWT is modern, sessions are old" is wrong (and dangerous).

## Sessions: server-side state

User logs in. Server stores a session record:

```
session_id: random_secure_string
user_id: 42
expires_at: 2026-05-28T12:00:00Z
role: admin
```

Server returns the `session_id` to the client (typically in a cookie). On every subsequent request, the cookie is sent; server looks up the session record.

### Storage backends

- **Postgres / MySQL** — durable, fine for small scale.
- **Redis** — fast, common at scale, TTL handles expiry.
- **PHP `$_SESSION`** — defaults to filesystem, swap to Redis for prod.

### Properties

- **Stateful**: requires a lookup per request (cheap with Redis, but a hop).
- **Easy to revoke**: just delete the session record.
- **Easy to rotate**: change the session_id; nothing on the client breaks.
- **Cookie carries no business data** — just a meaningless opaque ID.

## JWT: signed, stateless tokens

User logs in. Server creates a token containing user claims and a signature:

```
header:    { "alg": "HS256", "typ": "JWT" }
payload:   { "user_id": 42, "role": "admin", "exp": 1716123456 }
signature: HMAC(header + payload, secret_key)
```

The whole thing is base64-encoded and dot-joined:

```
eyJhbGc...header...
.eyJzdWI...payload...
.SflKxw...signature
```

The server returns this JWT. On every request, the client sends it (usually as `Authorization: Bearer ...`). The server **validates the signature** and reads the claims — **no DB lookup needed**.

### Properties

- **Stateless**: no DB or cache lookup needed (signature math only).
- **Self-contained**: claims travel with the token (user_id, role, scopes).
- **Hard to revoke**: a JWT is valid until it expires. (More below.)
- **Sized payload travels every request**: bigger than a session cookie.

## When sessions win

- **You need to revoke logins immediately** (password change, account suspend, security incident).
- **You want to track all active sessions** ("log me out everywhere").
- **You need to update permissions in real-time** (role change should affect the next request).
- **You have a single web app** (no need for cross-service token sharing).
- **You don't need to share auth across domains**.

**For typical web apps with a single backend: sessions are the right answer.** Server-rendered HTML, classic backend frameworks, single-domain apps — use sessions. They're battle-tested and free of JWT footguns.

## When JWT wins

- **Multiple services share auth without a central session store** — each can verify the JWT independently.
- **Cross-domain SSO**: token issued by one service is consumed by another.
- **Stateless services** — you don't want every service touching Redis on every request.
- **Mobile / native apps** that can't easily handle cookies (though they can).
- **Short-lived tokens** for API-to-API where revocation isn't critical.

Note: JWTs are also used as the **ID token** in OIDC, where they're consumed once to identify a user and then translated to a server-side session — that's a fine use.

## The JWT revocation problem

The classic gotcha. JWTs are valid until they expire. If you sign a 24-hour JWT and a user is later banned, **the JWT works for up to 24 more hours**.

### "Just check a blacklist"

Track revoked JWTs in Redis; check on every request. **You've just rebuilt sessions** with extra cryptographic overhead. If you must look up state per request anyway, sessions are simpler.

### "Use short-lived tokens"

Make JWTs valid for 5 minutes. Refresh via a long-lived refresh token. On revocation, the JWT is invalid within 5 minutes (acceptable for most apps).

This is the standard pattern. But it requires:
- A separate refresh-token mechanism (which is typically session-like and revocable).
- The 5-minute revocation window has to be acceptable.

### "Token versioning"

Include a `version` claim in the JWT. Store the user's current valid version in the DB. On each request, check `jwt.version >= db.user.current_version`. On revoke, bump the version.

**This is sessions in disguise.** You're doing a DB read per request.

## The JWT secret-management problem

The signing key validates every token. If it leaks, an attacker forges valid tokens for **anyone**. Devastating.

- Store the key in a secret manager (Vault, AWS KMS, GCP Secret Manager) — **not in code, not in env vars in plain text**.
- **Rotate keys** regularly. Support multiple active keys for verification during rotation.
- For asymmetric algorithms (RS256, ES256), the public key can be shared widely; only the private signing key must be guarded.

## JWT in `localStorage` vs cookies

A common debate. Both have risks:

- **localStorage** is accessible via JavaScript → susceptible to XSS (any compromised script reads it).
- **HttpOnly cookies** are not accessible via JS → safer from XSS, but susceptible to CSRF (an attacker's site can trigger requests with the cookie attached).

For web apps:
- **HttpOnly + Secure + SameSite=Lax (or Strict) cookies** is the safer default.
- For mobile / non-browser clients, the question is moot — those use `Authorization` headers from a secure store.

## A real architecture: hybrid

A pragmatic, common setup:

```
[user] ── login ──► [auth service]
                      ↓
                   creates session in Redis (server-side state)
                   issues short-lived (15-min) JWT signed with private key

[user] ── API call with JWT ──► [any service]
                                  validates JWT signature locally (no Redis hit)
                                  reads user_id, role, scopes from claims
                                  → handle the request

JWT expires every 15 min → client uses refresh token (cookie, HttpOnly)
to call /auth/refresh, which checks the Redis session and issues a new JWT.
```

Benefits:
- Services don't touch Redis per request (fast).
- Revoke a user → delete the session in Redis → within 15 min, all their access dies.
- Compromised JWT signing key impacts only the 15-min refresh window.

## Architect's takeaway

- **Sessions for typical web apps with one backend.** Battle-tested, simple, easy to revoke.
- **JWTs for stateless, multi-service architectures**, but plan for revocation explicitly.
- **Hybrid (short JWT + refresh session)** combines benefits — common in production.
- **JWT secret management is critical.** Use a secret store and rotate.
- **HttpOnly + Secure + SameSite cookies** are the safe default for browsers.
- **"JWT is modern" is not a reason** to choose it. Choose based on architecture needs.
