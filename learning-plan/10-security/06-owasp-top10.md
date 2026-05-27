# Example 06 — OWASP Top 10: the canonical app vulnerabilities

OWASP (Open Worldwide Application Security Project) publishes a "top 10" list every few years summarizing the most prevalent web application security risks. Knowing these is the minimum security literacy for an architect.

The current list (2021 — the most recent major revision):

## A01: Broken Access Control

The most common category — failure to enforce authorization.

**Examples**
- IDOR (Insecure Direct Object Reference): `GET /api/orders/123` returns any order, no ownership check.
- Privilege escalation: a regular user calls `/admin/users` and gets results.
- CORS misconfiguration: `Access-Control-Allow-Origin: *` on auth'd endpoints.
- Tampering with JWTs to elevate role (works if `none` algorithm is accepted).

**Defense**
- Always check authorization at the API/server layer.
- Default-deny: explicitly authorize each action.
- Centralize authorization logic (don't sprinkle `if user.role == ...` checks).
- Audit-log access decisions.

## A02: Cryptographic Failures (was "Sensitive Data Exposure")

Failure to protect data in transit or at rest.

**Examples**
- HTTP-only auth endpoint.
- Storing passwords in plaintext (or md5/sha1).
- Old TLS / weak ciphers enabled.
- API keys, tokens in URLs (logged everywhere).
- Database column with social security numbers, plaintext.
- Hard-coded secrets in code or env vars without secret manager.

**Defense**
- HTTPS everywhere; HSTS; modern TLS only.
- Passwords: hash with bcrypt/argon2 (not MD5/SHA1).
- Encryption at rest for sensitive columns (KMS-backed).
- Secrets in a vault, not in code or plain env.
- Don't put secrets in URLs.

## A03: Injection

User input is interpreted as code by a downstream system.

**Examples**
- SQL injection: `"SELECT * FROM users WHERE name = '" + input + "'"` with input `' OR '1'='1`.
- Command injection: `system("convert " + filename + " ...")`.
- NoSQL injection: passing user input directly into Mongo queries.
- LDAP, XPath, SMTP header injection.

**Defense**
- **Parameterized queries** / prepared statements — always. Never string-concatenate SQL.
- ORMs help (mostly) — but check raw query usage.
- For shell: don't use `system()` with user input; use `exec()` with arg arrays.
- Input validation as defense-in-depth, not as primary defense.

## A04: Insecure Design

Architecture-level flaws: missing rate limiting, no MFA, no abuse modeling. Whole-system issues, not specific bugs.

**Examples**
- /forgot-password sends new password to any email entered.
- "View cart" loads any cart by ID.
- No rate limit on /login — credential stuffing succeeds.
- API doesn't authenticate; relies on knowing the URL.

**Defense**
- Threat-model designs before implementing.
- Build security requirements into design docs.
- Use secure design patterns (defense in depth, least privilege).
- Conduct architecture reviews.

## A05: Security Misconfiguration

System runs in an insecure state because of bad defaults or missing hardening.

**Examples**
- Default credentials still in use (`admin/admin`).
- Stack traces returned in production responses.
- Debug mode enabled in prod.
- Open S3 buckets, public databases.
- Unnecessary services running, ports open.
- Missing security headers (HSTS, CSP, X-Frame-Options).

**Defense**
- Hardening checklists for new deployments.
- IaC (infrastructure as code) reviewed; resources private by default.
- Disable default accounts, set strong passwords.
- Strip stack traces from prod responses (log them server-side).
- Use security header presets (CSP, HSTS, X-Frame-Options, etc.).

## A06: Vulnerable and Outdated Components

Using libraries with known CVEs.

**Examples**
- `log4j` famously, in 2021.
- jQuery 1.x with XSS issues.
- Old Spring versions with RCEs.
- npm packages with known vulnerabilities.

**Defense**
- Automated dependency scanning (Dependabot, Snyk, Trivy, GitHub security alerts).
- Patch promptly; have a process for emergency patches.
- Pin versions in lockfiles; review updates.
- Minimize dependency footprint — fewer libs = fewer CVEs.

## A07: Identification and Authentication Failures

Weak login mechanisms.

**Examples**
- No rate limit on login → credential stuffing.
- No MFA option for admins.
- Session IDs in URL (leak via referrer headers).
- "Remember me" cookies that last years and can't be revoked.
- Weak password rules (allowing "password123").

**Defense**
- Rate-limit login per account and per IP.
- MFA for sensitive accounts (admin, finance).
- Use secure session management (HttpOnly + Secure + SameSite cookies).
- Password hashing with bcrypt/argon2.
- Detect impossible-travel logins (geo anomalies).
- Implement account lockout on repeated failures (with care to avoid DoS).

## A08: Software and Data Integrity Failures

Trusting unverified code or data.

**Examples**
- CI/CD pipeline accepts unsigned packages from public registries.
- Auto-update mechanism downloads code without signature verification.
- Deserializing untrusted data (`pickle.loads(user_input)` — catastrophic).
- Webhooks accepted without signature verification.

**Defense**
- Verify package signatures.
- Don't auto-execute code from untrusted sources.
- Treat all webhooks as untrusted until validated.
- Avoid deserialization of untrusted data; if you must, sandbox.
- Sign your own artifacts and verify them at deploy time.

## A09: Security Logging and Monitoring Failures

You can't respond to what you don't see.

**Examples**
- No login attempt logging.
- Logs not centralized / not retained.
- No alerts on anomalies.
- 6 months between breach and detection.

**Defense**
- Log authentication events, authorization decisions, admin actions.
- Centralize logs (SIEM or log management).
- Alert on suspicious patterns (lots of failed logins, role changes, mass deletes).
- Test incident response.
- Retain logs for a sufficient period (compliance often requires 1-7 years).

## A10: Server-Side Request Forgery (SSRF)

User input causes the server to make an HTTP request to an internal resource.

**Examples**
- "Fetch URL" feature lets attacker fetch `http://169.254.169.254/` (AWS metadata service → IAM credentials).
- Open redirect that leads to internal scan.
- Webhook handler that fetches arbitrary URLs from payloads.

**Defense**
- Don't let user input dictate the URL of an outbound HTTP call without strict allowlisting.
- Block internal IPs (`10.0.0.0/8`, `169.254.0.0/16`, `localhost`, etc.) in outbound HTTP libraries.
- Use IMDSv2 on AWS (requires session token to access metadata).
- Network-level egress controls — deny default, allow specific outbound destinations.

## Cross-cutting principles

### Defense in depth

Don't rely on one layer. Multiple defenses:
- WAF blocks SQL injection patterns.
- App uses parameterized queries.
- DB user has limited permissions (can't drop tables even if SQLi works).
- Network rules limit DB access.

### Principle of least privilege

Every user, service, key has the minimum permissions to do its job.

### Fail securely

Errors should default to "deny", not "allow". On unexpected errors, **don't** return data or grant access.

### Validate input, encode output

- **Validate** at boundaries (types, lengths, formats).
- **Encode** when output (HTML-escape for HTML output, JSON-escape for JSON, SQL-parameterize for SQL).

### Threat-model

For any new feature: who could abuse this, and how?

## How to use the OWASP Top 10

- **Engineering**: bake into design reviews and code reviews.
- **Architecture**: each design doc should consider relevant top-10 risks.
- **Training**: required reading for new engineers.
- **Tooling**: SAST (static analysis), DAST (dynamic scanning), dependency scanning all map to top-10 categories.

## Architect's takeaway

- **Broken access control is #1** — and the most likely thing you'll personally ship if you're not careful. Centralize AuthZ.
- **Injection still works** — parameterized queries everywhere.
- **Modern crypto** — TLS 1.3, bcrypt/argon2, secrets in vaults.
- **Threat-model designs** — security can't be retrofitted.
- **Patch dependencies** with automation.
- **Log and monitor** — undetected breaches are the worst.
- **Read the actual OWASP Top 10 doc** — it's well-written and has good examples per category.
