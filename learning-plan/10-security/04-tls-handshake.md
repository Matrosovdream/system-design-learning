# Example 04 — TLS handshake: how the lock icon works

TLS (Transport Layer Security) is what puts the "S" in HTTPS. Every architect should know the rough shape of the handshake — both because it explains performance (handshakes are expensive) and because it shapes deployment (certificates, TLS termination, mTLS).

## What TLS provides

1. **Encryption** — packets are unreadable to network eavesdroppers.
2. **Integrity** — packets can't be modified in transit without detection.
3. **Authentication** — the server proves it really is who it claims (via certificate).

It does **not** provide authentication of the **client** — that's optional (mTLS).

## TLS 1.3 handshake (the current standard)

Before TLS 1.3, handshakes took 2 round-trips. TLS 1.3 cuts it to 1 RTT (and 0-RTT for resumption). That's why migrating to 1.3 is a performance win.

### Step 1: Client hello

```
Client → Server:
  - TLS version (1.3)
  - Cipher suites I support: TLS_AES_256_GCM_SHA384, ...
  - Random number (32 bytes)
  - Key share (Diffie-Hellman public value for several curves)
  - Server name (SNI: example.com)  ← tells the server which cert to use
  - Other extensions
```

In TLS 1.3, the client **already includes** a Diffie-Hellman key share, saving a round-trip.

### Step 2: Server hello + certificate + finished

```
Server → Client:
  - Chosen cipher suite
  - Server's random number
  - Server's key share (DH public value)
  - Server's certificate (X.509)
  - Certificate verification (signed)
  - Finished message
```

The server picks a cipher, contributes its DH key share, sends its cert, and signs a hash of the handshake so the client knows the response is authentic.

### Step 3: Client validates and replies

The client:
1. **Validates the certificate**:
   - Cert is signed by a trusted CA (or chains to one in the trust store).
   - Cert is for the requested domain (SNI matches `subject` / `SAN`).
   - Cert is not expired and not revoked.
2. **Computes shared secret** using its private DH value and the server's DH public.
3. **Derives session keys** from the shared secret.
4. Sends a "Finished" of its own, encrypted with the new session keys.

### Step 4: Encrypted application data flows

Both sides now have a shared secret. All further data is encrypted with AEAD ciphers (AES-GCM, ChaCha20-Poly1305).

## Why "one RTT"

TLS 1.3:
- RTT 1: ClientHello + key share → ServerHello + cert + finished → ClientFinished + encrypted data.
- Application data can start flowing immediately after the handshake.

TLS 1.2 (legacy):
- RTT 1: ClientHello → ServerHello + cert.
- RTT 2: ClientKeyExchange + ChangeCipherSpec → finished.
- THEN application data.

That's the visible perf win.

## Certificates: the trust anchor

A certificate is a signed document containing:
- The domain (or wildcard).
- The public key.
- The issuer (CA).
- Validity dates.
- Various extensions (key usage, etc.).

The CA signs the cert with its private key. The client's "trust store" contains CA public keys. So:

```
Browser trust store: { Let's Encrypt root, DigiCert root, GlobalSign root, ... }
Cert says: "I am example.com, signed by Let's Encrypt"
Browser: I trust Let's Encrypt → I trust this cert.
```

Modern certs are issued via **Let's Encrypt** (free, automated). Use `certbot`, `acme.sh`, or your platform's built-in automation.

### Wildcard vs SAN certs

- **Wildcard** (`*.example.com`): covers any subdomain. Larger blast radius if compromised.
- **SAN** (Subject Alternative Name): cert lists specific domains (`api.example.com`, `www.example.com`, ...). More granular.

For automated provisioning (Let's Encrypt + DNS challenge), wildcards are easy. For static deployments, SAN is fine.

## TLS termination

Decrypting TLS costs CPU. Best practice: terminate at the edge.

```
[client] ── HTTPS ──► [LB / reverse proxy: TLS terminates here] ── HTTP ──► [app servers]
```

The LB / reverse proxy:
- Holds the cert and private key.
- Decrypts incoming TLS.
- Re-encrypts (or passes plaintext) to backends.

Backends serve plain HTTP, which is fine **inside your network** (depending on trust model).

## mTLS: mutual authentication

Normal TLS: server authenticates to client. Client is anonymous (until it logs in with credentials).

mTLS: client ALSO presents a cert. Server verifies it. Now both parties are authenticated.

Used for:
- **Service-to-service** communication in zero-trust networks.
- **API access** for partner integrations (more secure than API keys).
- **VPN / Wireguard-style tunneling**.

In a service mesh (Istio, Linkerd), mTLS is automatic for every service-to-service call. Sidecars handle the cert lifecycle.

## Cipher suite

A TLS cipher suite specifies the algorithms used:

```
TLS_AES_256_GCM_SHA384
  ↑
  protocol
        ↑
       symmetric cipher + mode
                       ↑
                       hash function (for HMAC and key derivation)
```

In TLS 1.3, only AEAD ciphers (authenticated encryption) are allowed:
- AES-256-GCM
- AES-128-GCM
- ChaCha20-Poly1305 (good for CPUs without AES-NI)

Older ciphers (RC4, 3DES, CBC mode, RSA key exchange) are removed. **Don't enable them** even if your TLS library supports it.

## Forward secrecy

A property: if the server's private key leaks in the future, **past sessions cannot be decrypted**.

Achieved by using ephemeral DH key exchange — each session has its own one-time DH keys, separate from the cert's RSA key.

TLS 1.3 mandates forward-secret cipher suites. TLS 1.2 supports them via `ECDHE_*` cipher suites — make sure you only enable those.

## Common pitfalls

### Cert expired

Cert expires; browsers show big red error; site is down. Famous outages from this (LinkedIn, Equifax, GitHub Pages).

**Fix:** automated renewal (cron + certbot), monitoring expiry well in advance (alert at 30 days remaining).

### Trusting too many CAs

Default trust stores have hundreds of CAs. Some have been compromised in the past. For sensitive operations, use **certificate pinning** (mobile apps) or a restricted CA list.

### Mixed content

HTTPS page loads HTTP resources. Modern browsers block; older ones warn. **Everything should be HTTPS**, including third-party assets.

### Weak protocols / ciphers enabled

A server that accepts TLS 1.0 / 1.1 fails compliance and is vulnerable to known attacks. Disable TLS < 1.2; ideally enforce TLS 1.3 only.

Tools to test: SSL Labs' SSL Test (ssllabs.com/ssltest/), `testssl.sh`.

## Architect's takeaway

- **TLS 1.3 is the standard.** Don't deploy 1.2-only stacks if you can avoid it.
- **Terminate TLS at the edge** (LB / reverse proxy) — backends serve HTTP, less CPU and easier cert management.
- **mTLS for service-to-service** if you're in zero-trust mode. Service mesh makes this easy.
- **Use Let's Encrypt** with automatic renewal. Manual cert management is how outages happen.
- **Enforce forward-secret cipher suites** (ECDHE) — never RSA key exchange.
- **Monitor cert expiry** with multiple weeks of warning. Don't be the next "expired cert outage" headline.
