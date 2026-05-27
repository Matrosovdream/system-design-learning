# Example 03 — OAuth 2.0 with PKCE: the standard delegated auth flow

OAuth 2.0 is **delegated authorization**: a user grants a third-party application access to their data on another service, without sharing their password.

OIDC (OpenID Connect) sits on top of OAuth 2.0 and adds **authentication** — telling the third-party app who the user actually is.

## The cast

- **Resource Owner**: the user.
- **Client**: the app requesting access (your app).
- **Authorization Server**: issues tokens (e.g., Google, GitHub, Auth0).
- **Resource Server**: the API holding the user's data (e.g., Google's APIs).

## The "Login with Google" you've seen 1000 times

```
1. You click "Sign in with Google" on app.example.com.
2. Browser redirects to accounts.google.com with parameters.
3. You log into Google (if not already), grant permissions to app.example.com.
4. Google redirects back to app.example.com with a "code".
5. app.example.com's server exchanges the code for an access token (and ID token).
6. app.example.com is now logged in as you.
```

This is the **Authorization Code flow** — the most common OAuth flow, and the one you should use.

## Authorization Code with PKCE: the modern default

PKCE (pronounced "pixie", "Proof Key for Code Exchange") adds a defense against authorization code interception. It's now recommended for **all** clients — even server-side ones.

### Step-by-step

#### Step 1: Client generates a code verifier and challenge

```python
# Random 43-128 character string
code_verifier = "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"

# Hash + base64url encode it
code_challenge = base64url(sha256(code_verifier))
# = "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM"
```

The verifier is kept secret by the client. The challenge will be sent to the authorization server.

#### Step 2: Client sends user to the authorization server

```http
GET https://accounts.google.com/o/oauth2/v2/auth?
  response_type=code
  &client_id=client-app-id
  &redirect_uri=https://app.example.com/callback
  &scope=openid+email+profile
  &state=random_anti_csrf_token
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
  &code_challenge_method=S256
```

User sees Google's login page (if not already authenticated) and a consent screen ("app.example.com wants access to your profile and email").

#### Step 3: User approves; auth server redirects back with code

```http
HTTP/1.1 302 Found
Location: https://app.example.com/callback?
  code=4/0AfJohXl...
  &state=random_anti_csrf_token
```

The browser hits app.example.com's `/callback` with the authorization code in the URL.

The `state` parameter must match what the client sent — defends against CSRF where an attacker tricks the user into hitting the callback with someone else's code.

#### Step 4: Client exchanges code + verifier for tokens (back channel)

The client's server (not browser) makes a POST to Google:

```http
POST https://oauth2.googleapis.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=4/0AfJohXl...
&client_id=client-app-id
&client_secret=very-secret    ← if server-side; SPAs omit this
&redirect_uri=https://app.example.com/callback
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

Google checks that `sha256(code_verifier) == code_challenge` from step 2. If yes, it returns tokens:

```json
{
  "access_token": "ya29.a0AfH6S...",
  "expires_in": 3600,
  "refresh_token": "1//0g_OBLeY8u-...",
  "id_token": "eyJhbGciOi...",   ← OIDC, identifies the user
  "scope": "openid email profile",
  "token_type": "Bearer"
}
```

#### Step 5: Client uses the access token to call APIs

```http
GET https://www.googleapis.com/oauth2/v3/userinfo
Authorization: Bearer ya29.a0AfH6S...
```

Response: the user's profile data.

The `id_token` is a JWT containing identity claims about the user (sub = user_id, email, name, picture). The client decodes it to know who the user is.

## Why PKCE matters

Without PKCE:
1. Attacker intercepts the auth code (e.g., from a leaky mobile app's URL handler, network tap).
2. Attacker exchanges the code for tokens.
3. Attacker is now the user.

With PKCE: even if the code is intercepted, the attacker doesn't have the `code_verifier`. The token exchange fails.

Originally PKCE was for public clients (mobile, SPA — clients that can't keep a secret). **Now it's recommended for all clients**, including server-side ones, as defense in depth.

## Other OAuth flows (and when to use them)

| Flow                          | Use when                                    | Notes                              |
|-------------------------------|---------------------------------------------|------------------------------------|
| **Authorization Code + PKCE** | Web apps, SPAs, mobile (everything modern)  | The default. Use this.             |
| **Client Credentials**        | Service-to-service (no user)                | Backend systems calling each other |
| **Device Authorization**      | Devices without a browser (TV, CLI)         | "Visit this URL, enter this code"  |
| Implicit (deprecated)          | Old SPAs                                    | Don't use — replaced by auth code + PKCE |
| Password (deprecated)          | First-party trusted clients only            | Discouraged — defeats OAuth's purpose |

## OIDC: adding authentication on top

OAuth alone says "the bearer of this token can access these scopes" — it doesn't say who the user is.

OIDC adds:
- **The `id_token`**: a JWT with the user's identity claims (`sub`, `email`, `name`, etc.).
- **A standardized `/userinfo` endpoint**.
- **Discovery**: `/.well-known/openid-configuration` describes the provider's endpoints.

If you're "logging in with Google", you're using OIDC, not pure OAuth.

## Tokens: access, refresh, ID

- **Access token**: short-lived (5-60 min), used to call APIs. If leaked, damage is bounded by the TTL.
- **Refresh token**: long-lived (days-months), used to get new access tokens without re-prompting the user. Must be stored securely.
- **ID token**: identity of the user. Used once (after auth), not for API calls.

## Common mistakes

### Storing access tokens in localStorage

XSS can steal them. Prefer:
- **HttpOnly cookies** for SPAs that share a domain with the backend.
- **Memory-only storage** during the user's session; re-acquire on refresh.

### Long-lived access tokens

A leaked access token with 30-day TTL is a disaster. Short TTLs (15-60 min) + refresh token is the right shape.

### Confusing OAuth scopes with app permissions

Scopes are what the OAuth provider allows. **Your app still needs its own authorization model** — "this Google user can do X in MY app" is your problem, not Google's.

### Using OAuth where you don't need it

OAuth is for **third-party delegated** access. For your own users on your own app, password-based or magic-link login is fine. Don't add OAuth complexity if you're not delegating.

## Architect's takeaway

- **Authorization Code with PKCE is the modern default.** Use it everywhere.
- **OIDC = OAuth + identity layer.** Use OIDC when you want "Sign in with Google".
- **Short-lived access + long-lived refresh tokens** is the standard token pattern.
- **Don't roll your own.** Use Auth0, Okta, Cognito, Keycloak, or trusted libraries. OAuth has many subtle pitfalls.
- **For service-to-service**, use Client Credentials flow.
- **Validate `state` parameter** to prevent CSRF.
- **PKCE is no longer just for mobile** — recommended for all OAuth clients.
