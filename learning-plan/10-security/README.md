# Step 10 — Security

Security is a cross-cutting concern, not a feature. As an architect, you need to know how authentication and authorization work, how data is protected in transit and at rest, how to harden APIs against abuse, and what the OWASP top 10 actually means.

This step focuses on **application-layer security**. Network security and compliance are vast topics with their own specialists — but every architect must understand this layer.

## Goals

- Explain the difference between **authentication** and **authorization**.
- Compare **sessions** and **JWT** and pick correctly.
- Walk through an **OAuth 2.0** flow (authorization code with PKCE).
- Apply **rate limiting** at multiple layers.
- Read and apply the **OWASP Top 10** in your designs.
- Reason about **encryption at rest, in transit, in use**.

## Key concepts

1. **AuthN vs AuthZ** — who are you vs what can you do.
2. **Sessions** — server-side stateful auth.
3. **JWT** — signed, stateless auth tokens.
4. **OAuth 2.0 / OIDC** — delegated auth, the standard for "login with X".
5. **RBAC / ABAC** — role- vs attribute-based access control.
6. **TLS** — transport encryption; mTLS for service-to-service.
7. **Rate limiting** — token bucket, leaky bucket, sliding window.
8. **Secrets management** — never in code; vault, KMS, env injection.
9. **OWASP Top 10** — the catalog of common app vulnerabilities.
10. **Defense in depth** — layered protections.

## Reading

- **OWASP Top 10** — https://owasp.org/Top10/
- **Auth0 blog** — excellent OAuth/OIDC explainers.
- **Cloud provider docs**: AWS IAM, GCP IAM, Azure AD.
- **Cryptographic Right Answers** — Latacora's regularly-updated post on crypto choices.

## Examples in this folder

- `01-authn-vs-authz.md` — the most-mixed-up pair in security.
- `02-sessions-vs-jwt.md` — the trade-off everyone gets wrong.
- `03-oauth2-flow.md` — the authorization code with PKCE flow, step by step.
- `04-tls-handshake.md` — how the lock icon actually works.
- `05-rate-limiting-strategies.md` — token bucket, leaky bucket, sliding window.
- `06-owasp-top10.md` — the canonical app vulnerabilities and how to defend.

## Self-check

1. Your API uses JWTs. A user's password is leaked. How do you log them out everywhere? (Trick — read this carefully.)
2. Why does OAuth not handle authentication directly? What does OIDC add?
3. You implement rate limiting in your app code. Why is that also/instead needed at the gateway?
4. What's the difference between "encrypted in transit" and "encrypted at rest"?
5. Your form has a SQL query: `SELECT * FROM users WHERE name = '" + input + "'`. Name three things wrong with this.
