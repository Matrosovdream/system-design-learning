# Example 01 — Authentication vs Authorization: the pair people confuse

Two distinct steps, often blurred together in conversation. Mixing them up causes real security holes.

## Authentication (AuthN) — "who are you?"

The system verifies the user's identity. Done once at login (or on each request if stateless).

Mechanisms:

- **Password** — what you know.
- **Multi-factor** (TOTP, SMS, push) — what you have.
- **Biometric** (fingerprint, FaceID) — what you are.
- **Magic link, OAuth login** — what you can prove via a trusted party.
- **API key, mutual TLS certificate** — for service-to-service.

Outcome of authentication: **a verified identity**. e.g., "this request comes from user ID 42".

## Authorization (AuthZ) — "what can you do?"

Given a verified identity, decide whether the action is permitted.

Mechanisms:

- **RBAC** (role-based): users have roles; roles have permissions.
- **ABAC** (attribute-based): policy uses arbitrary attributes (department, time, resource).
- **ReBAC** (relationship-based): "Alice can view this doc because she's a member of the team that owns it".
- **Per-resource ACLs**: each resource lists who can do what.

Outcome of authorization: **allow or deny** this specific action.

## Where they happen in a request

```
HTTP request arrives
   ↓
[AuthN]  Validate JWT / session cookie.
         If invalid → 401 Unauthorized.
         Now we know: user_id = 42.
   ↓
[AuthZ]  Is user 42 allowed to GET /orders/789?
         Check ownership, role, etc.
         If no → 403 Forbidden.
   ↓
Handler executes
```

**Status codes:**
- `401 Unauthorized` = "I don't know who you are." (AuthN failed.)
- `403 Forbidden` = "I know who you are, and you can't do this." (AuthZ failed.)

The naming is unfortunate — `401` is really authentication failure, but the standard committed to "unauthorized".

## A concrete example

GitHub repo with users:

```
user "alice" — admin of repo "alice/my-project"
user "bob"   — read-only collaborator on the same repo
user "carol" — no relationship to the repo
```

AuthN: each is logged in, verified by their credentials. All three are authenticated.

AuthZ:
- `POST /repos/alice/my-project/issues` (create issue):
  - alice → allowed (admin)
  - bob → allowed (collaborators can create issues)
  - carol → allowed (if public repo) or denied (if private)
- `DELETE /repos/alice/my-project`:
  - alice → allowed
  - bob → forbidden (not admin)
  - carol → forbidden

Same identity check; different decisions per resource and action.

## Common security holes from mixing them up

### Mistake 1: AuthN without AuthZ

```python
@app.route('/api/orders/<id>')
@require_login   # checks AuthN
def get_order(id):
    return Order.find(id).to_json()
```

The function checks that the user is logged in. It does NOT check whether **this user** owns order `id`. An attacker logs in as user X, requests `/api/orders/Y` (someone else's order), gets it.

This is the **Insecure Direct Object Reference (IDOR)** vulnerability. It's #1 on OWASP for a reason.

**Fix:**
```python
@app.route('/api/orders/<id>')
@require_login
def get_order(id):
    order = Order.find(id)
    if order.user_id != current_user.id and not current_user.is_admin:
        return abort(403)
    return order.to_json()
```

### Mistake 2: AuthZ logic in the UI only

UI hides the "Delete" button for non-admin users → admin-only feature, right? Wrong. The button is hidden but the endpoint is unprotected. Anyone with curl can hit `DELETE /admin/users/1`.

**Rule:** AuthZ checks must be at the **API/server layer**, not at the UI. UI is the convenience layer; the server is the security layer.

### Mistake 3: Trusting the client to send identity

```python
@app.route('/api/orders')
def list_orders():
    user_id = request.headers.get('X-User-Id')  # 😱
    return Order.where(user_id=user_id).to_json()
```

Anyone sets `X-User-Id: 42` to see anyone's orders. The server must derive identity from a **verified** credential (validated JWT, session cookie), never from a free-form header.

### Mistake 4: Inconsistent checks across endpoints

`GET /orders/:id` checks ownership. `GET /orders` returns all of them (developer forgot). One endpoint leaks everything.

**Fix:** centralize authorization logic. Don't sprinkle `if user.id != ...` checks throughout the codebase.

## Modeling authorization

### RBAC — the default

```
roles: admin, editor, viewer
permissions: 
  admin: { create_post, edit_post, delete_post, manage_users }
  editor: { create_post, edit_post }
  viewer: { read_post }

User has one or more roles.
```

Simple, manageable for small apps. Fails when "user can edit posts in workspace X but only view in workspace Y".

### ABAC — for more complex policies

A policy engine evaluates rules:

```
policy "edit_post":
  ALLOW IF
    user.workspace_role[post.workspace_id] IN ('admin', 'editor')
    AND post.status != 'archived'
    AND time.now() < post.locked_until
```

Engines: AWS IAM policies, Open Policy Agent (OPA), Cedar, Casbin. Heavy but flexible.

### ReBAC — for graph-shaped permissions

Permissions defined by **relationships**:

```
- Alice is owner of document doc-1
- Document doc-1 is in folder folder-A
- Bob is editor of folder folder-A
- Therefore Bob can edit doc-1 (transitively)
```

Google's Zanzibar paper is the canonical reference. Implementations: SpiceDB, Permify, OpenFGA. Excellent for social-network and collaboration permissions.

## The principle of least privilege

Give every entity (user, service, API key) the **minimum** permissions to do its job.

- A service account that only needs to read from a bucket → grant only `s3:GetObject`, not `s3:*`.
- A user account for an integration → scoped API key with read-only permissions.
- Cron job that runs a daily report → permissions only for that report's tables.

When (not if) credentials leak, the blast radius is bounded.

## Audit logging

Every important authorization decision should be **logged**:

- Who: which authenticated identity.
- What: which action / which resource.
- When: timestamp.
- Result: allow / deny.

These logs:
- Detect anomalies ("admin accessed 10x more data than usual").
- Investigate incidents.
- Provide compliance evidence.

Sensitive operations (delete, role changes, financial actions) deserve **immutable** audit logs — write-only, retained for years.

## Architect's takeaway

- **AuthN and AuthZ are different.** Always do both, in that order.
- **`401` = no identity; `403` = identity but no permission.** Use them correctly.
- **Authorization at the API/server layer.** UI hiding is convenience, not security.
- **Centralize authorization logic** so you don't have inconsistent checks scattered.
- **Principle of least privilege** for every credential and service account.
- **Audit log authorization decisions.** You'll thank yourself during the next incident.
