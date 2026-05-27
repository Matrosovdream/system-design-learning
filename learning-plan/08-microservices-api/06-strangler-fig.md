# Example 06 — Strangler fig: incrementally killing a monolith

You inherit a 10-year-old PHP monolith. Everyone says "we need microservices". You can't rewrite it all (last team that tried failed). What do you do?

The **strangler fig pattern**: incrementally replace pieces of the monolith with new services, until the monolith is small enough to retire (or stays as a small legacy core).

Named after the strangler fig tree, which grows around a host tree until the host eventually dies inside it.

## The high-level approach

```
Year 0:                     Year 2:                       Year 4:
[ monolith ]                [ monolith ]   [svc A]        [monolith]  [svc A]
                                                          ↘          ↘
                                                            [svc B]    [svc C]
                                                                       [svc D]
```

The monolith shrinks; the services grow. The host (monolith) is "strangled" over time.

## The pattern in detail

### Step 1: Add a routing layer in front of the monolith

Put a **proxy / gateway** between clients and the monolith. Initially it just forwards everything.

```
[client] → [proxy] → [monolith]
```

Why: gives you a place to redirect traffic later, without changing clients.

### Step 2: Pick a candidate piece

Look for a piece of the monolith that:
- **Has clear boundaries** — its data and logic don't tangle with everything else.
- **Has a real reason to be extracted**: scale, deploy frequency, team ownership.
- **Is high-value but bounded** — succeeding here proves the pattern.

Common first candidates: **notification, search, image processing, authentication**.

### Step 3: Build the new service alongside

Build the replacement service. It implements the same API as the corresponding monolith endpoints.

```
[client] → [proxy] → [monolith]   (existing)
                  → [new service] (built but no traffic yet)
```

### Step 4: Shadow traffic (optional but recommended)

The proxy sends each request to **both** the monolith and the new service. Compare responses. The monolith's response is what the client sees; the new service is just observed.

Discrepancies = bugs in the new service. Fix until shadow traffic matches the monolith for ~all traffic.

### Step 5: Cut over a small percentage

Route 1% of traffic to the new service. Monitor errors, latency, correctness.

```
[client] → [proxy] → 99% [monolith]
                  →  1% [new service]
```

Increase gradually: 1% → 10% → 50% → 100%. At each step, validate.

### Step 6: Decommission the old code path

Once 100% on the new service, **delete the old code from the monolith**. (Tempting to leave it "just in case" — resist. Dead code rots.)

### Step 7: Repeat for the next piece

Now the monolith is slightly smaller. Pick the next candidate. Repeat.

## Why incremental beats "big-bang rewrite"

Big-bang rewrites famously fail. Reasons:
- **You discover undocumented behavior** that the old system did, that you didn't replicate.
- **The business moves while you're rewriting** — by the time you ship, the old system has new features.
- **No business value** until launch day → leadership loses patience.
- **Risk concentration** — if launch goes wrong, you have a catastrophe.

Incremental strangling:
- Each extracted piece ships independently.
- Each piece delivers value (and proves the approach).
- Risk is distributed across many small launches.
- The business keeps moving on the monolith; you don't fall behind.

## Picking the right first piece

Best candidates have these properties:

### High value of independence

A piece that the team **needs to deploy faster** (e.g., notification — A/B test new email templates) is worth extracting. A piece nobody touches (legacy export feature) isn't worth the effort.

### Clear data ownership

Easy: notifications, authentication, search. These have data that doesn't tangle with everything.

Hard: orders, users, products. These data are referenced by half the monolith. Extract these later, with more care.

### Bounded blast radius

If the new notification service breaks, the worst case is "users don't get an email". Annoying, recoverable.

If the new payments service breaks, you might double-charge customers. Don't extract payments first.

## Common patterns during strangling

### Database stays shared (temporarily)

The new service might initially read/write the same DB as the monolith. Yes, this is "the distributed monolith antipattern" — but **as a transition state** it's pragmatic.

Goal: incrementally migrate data ownership to the new service over time (next pattern).

### Data migration: extract + sync

When you're ready to give the new service its own DB:

1. **Add CDC** (change data capture) from the monolith's DB → the new service's DB. The new service has its own copy, kept in sync.
2. **Direct writes** from the new service initially go to **both** the new DB and the old DB (or just the old, via the monolith API).
3. **Cut over reads** to the new DB first.
4. **Cut over writes** once you trust the new DB to be the source of truth.
5. **Stop syncing.** Old DB is read-only or eventually removed.

This is risky, takes weeks. Don't underestimate.

### Anti-corruption layer

The new service shouldn't inherit the monolith's bad legacy concepts. Build an **anti-corruption layer** at the boundary that translates between the monolith's data model and the new service's clean model.

```
monolith concept (legacy):    User has both billing_addr_1 and billing_addr_2 strings
new service concept (clean):  User has a list of structured Addresses

translation lives in the boundary, not in the new service.
```

## A real example: extracting notifications

PHP monolith. The codebase has `notify_user($user_id, $event)` called from 47 places, doing email sending inline with SMTP.

Goal: extract a notification service in Go that handles email, SMS, push, with templating and async delivery.

Plan:

1. **Add a proxy.** Add Envoy in front of the monolith.
2. **Build notification service.** Go service with REST API: `POST /notifications`. Internally: queue → workers → SES/SendGrid.
3. **Add a feature flag in the monolith.** When set, `notify_user()` calls the new service instead of inline SMTP.
4. **Shadow traffic.** Flag set to "shadow mode" for 1 week — calls both, compares.
5. **Cut over 10% → 50% → 100%** of `notify_user()` calls.
6. **Delete the inline SMTP code** from the monolith. ~2,000 lines gone.
7. **Repeat for SMS and push.**

Result: notification logic lives in a small service that the team can deploy independently. The monolith is slightly smaller.

## What you do NOT do

- **Don't try to extract the most central piece first** (users, orders). Wait until you have practice.
- **Don't allow new features to live in both worlds.** New features go in the new service or in the monolith, not both — drift kills you.
- **Don't skip shadow traffic / canary.** Big-bang cutover is how outages happen.
- **Don't leave dead code in the monolith.** Delete it the moment the cutover is complete.
- **Don't extract for the sake of extracting.** Each extraction must justify its complexity.

## When the strangler is "done"

Reality: it's often not "done", and that's fine.

Many companies end up at:
- 70% of functionality in new services.
- 30% in a smaller monolith that holds auth, billing, and "stuff that's stable and works".

That's a perfectly good steady state. The monolith was the enemy because of its size and tangling; a smaller, well-bounded monolith is just another service.

## Architect's takeaway

- **The strangler fig pattern is the right way to migrate monoliths.** Big-bang rewrites fail.
- **Pick the first piece carefully**: bounded, high-value, low blast radius.
- **Use a proxy / gateway** to enable gradual cutover without client changes.
- **Shadow traffic** before percentage-based cutover. Compare responses for correctness.
- **Anti-corruption layers** keep new services from inheriting legacy mess.
- **You don't have to fully strangle the monolith.** A small, focused legacy core is fine.
