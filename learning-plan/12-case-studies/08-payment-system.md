# Case Study 08 — Design a payment system (Stripe-like)

The question that tests your understanding of **strong consistency**, **idempotency**, **financial correctness**, **regulatory awareness**, and **failure handling**. Get this wrong in real life and people lose real money. Interviewers love this one.

## Problem statement

Build a payment platform that:
- Accepts payments via cards (and other methods).
- Processes them through payment processors / banks.
- Records every transaction immutably.
- Supports refunds, disputes, chargebacks.
- Handles failures gracefully (no double-charges, no lost charges).
- Generates reports / payouts to merchants.

## Clarifying questions

1. **Scale**: transactions/day, peak rate, average amount.
2. **Methods**: cards only, or bank transfer, wallets, crypto, etc.?
3. **Currencies**: single or multi?
4. **Geography**: regulatory implications.
5. **Latency requirement**: how fast must the user see "payment succeeded"?
6. **PCI / compliance**: do we handle card data directly, or tokenize via a third party?
7. **Recurring billing**: subscriptions?

**Assumed answers:**

- 50M transactions/day, peak 5000/sec, average $50.
- Cards (Visa/Mastercard) + bank transfer (ACH/SEPA).
- Multi-currency.
- US + EU + APAC.
- p95 sub-2-second user-facing.
- PCI-DSS Level 1 compliant; handle tokenization in-house.
- Subscriptions out of scope (separate billing module).

## Functional requirements

- Charge a card.
- Refund a charge (full or partial).
- Cancel/void an authorized but uncaptured charge.
- List a customer's transactions.
- Payout to merchant bank accounts.
- Webhook events for status changes.

## Non-functional requirements

- **Correctness above all** — never double-charge, never lose a charge.
- p95 charge latency < 2 sec (limited by external processor).
- 99.99% uptime for the API.
- All transactions are immutable, fully auditable.
- Regulatory compliance: PCI-DSS, GDPR, regional finance laws.

## Capacity estimation

```
50M tx/day = ~580/sec average; peak ~5000/sec.
Each transaction record: ~2 KB metadata.
Storage: 50M × 2 KB × 365 = ~36 TB/year.

This is small. The architecture is dominated by correctness, not throughput.
```

## API design

```
POST /v1/charges
Idempotency-Key: <key>
{
  "amount": 5000,           // cents
  "currency": "usd",
  "source": "tok_visa",     // tokenized card
  "customer": "cus_...",
  "description": "Order #123",
  "capture": true,           // false = authorize only
  "metadata": {...}
}
→ 200 {
  "id": "ch_...",
  "status": "succeeded" | "failed",
  "amount": 5000, ...
}

POST /v1/charges/{id}/refunds
Idempotency-Key: <key>
{ "amount": 2500 }            // optional partial
→ 200 { "id": "re_...", "status": "succeeded" }

POST /v1/charges/{id}/capture       (for previously authorized)
GET  /v1/charges/{id}
GET  /v1/charges?customer={id}
```

## High-level architecture

```
[merchant client]
   │
   ▼
[API gateway]
   │
   ▼
[Payment API service]
   │
   ├─► [Idempotency service: Redis]
   │       (24h dedup of Idempotency-Key)
   │
   ├─► [Ledger: append-only journal in Postgres]
   │       (every state change is a journal entry)
   │
   ├─► [Card vault]
   │       (PCI-isolated service; only this service sees plaintext PANs)
   │
   ▼
[Payment processor service]
   │
   ├─► [Visa/Mastercard via acquirer (Stripe, Adyen, Worldpay, etc.)]
   │
   ├─► [ACH / SEPA via banks]
   │
   └─► [Webhook events service]
           ↓
       [Merchant webhook endpoints]
   
[Async workers]
   - Settlement / payout scheduling
   - Dispute / chargeback handling
   - Reconciliation with processor reports
   - Fraud scoring
```

## Deep dive: the ledger (double-entry accounting)

The core of any payment system. **Every money movement is an immutable journal entry.**

```sql
CREATE TABLE ledger_entries (
    id UUID PRIMARY KEY,
    transaction_id UUID NOT NULL,    -- groups debits/credits of one logical tx
    account_id UUID NOT NULL,
    direction TEXT NOT NULL,         -- 'debit' or 'credit'
    amount_minor BIGINT NOT NULL,    -- in smallest unit (cents)
    currency TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT now(),
    metadata JSONB
);

CREATE INDEX ON ledger_entries (transaction_id);
CREATE INDEX ON ledger_entries (account_id, created_at);
```

For one charge of $50 to customer C, sending $48.50 to merchant M after fees:

```
transaction_id: tx-123
  - debit: customer_funding_account, $50.00
  - credit: merchant_holdings_account_M, $48.50
  - credit: fees_account, $1.50
```

Invariant: **for each transaction, debits = credits.** This is a database constraint (or enforced in code).

Account balances are derived: `SUM(credits) - SUM(debits)` per account. (Cache with periodic snapshots for performance.)

This is **double-entry accounting**, the same model banks have used for centuries. It makes errors detectable and auditable.

## Deep dive: idempotency

The most critical correctness property. A POST to `/charges` must never produce two charges, no matter how many times the client retries.

```
POST /v1/charges
Idempotency-Key: ord_xyz_attempt_1

server:
  1. Look up key in Redis. If exists with response: return cached response (with original status code).
  2. If not exists: insert (key, "in_progress", started_at) atomically.
     a. If lost race: poll the key until status changes, return cached response.
  3. Process the charge.
  4. Store (key, response, status, completed_at) in Redis with 24h TTL.
  5. Return response.
```

The key is also propagated to the **downstream processor** call:

```
acquirer.charge(amount, card_token, idempotency_key=our_internal_tx_id)
```

So if **we** retry to the processor, they also dedupe.

End result: from API call to processor, the entire chain is idempotent. The user can retry; we retry; the processor sees one charge.

## Deep dive: the state machine of a charge

```
created → authorized → captured → succeeded
   │           │            │            │
   │           ↓            ↓            ↓
   ↓        failed       failed      refunded (partial or full)
failed                                  │
                                         ↓
                                      disputed → chargeback
                                                       ↓
                                                  resolved (won or lost)
```

Each transition is **logged in the ledger**. The current state is a derived view.

This makes recovery easy: if you crash mid-charge, you can reconstruct the exact state from the ledger.

## Deep dive: the processor call

This is where you're at the mercy of external systems.

```
charge_with_processor(card_token, amount, idempotency_key):
  try:
    response = processor.charge(...)   ← network call, might take 1-10 seconds
    if response.success: 
        return SUCCESS(response.processor_id)
    else:
        return FAILED(response.decline_reason)
  except Timeout:
    return UNKNOWN
  except NetworkError:
    return UNKNOWN
```

The crucial case: **timeout** or **network error**. You don't know if the processor charged the card or not.

Possibilities:
1. The request never reached the processor → no charge.
2. The request reached, processor charged, response lost → charge happened.
3. The request reached, processor errored → no charge.

You **cannot resolve this from one call**. Approach:

### Reconciliation

```
1. Mark the charge as "UNKNOWN" / "pending_verification".
2. Periodically (or on demand) query the processor: "what's the status of charge with my idempotency_key X?"
3. Processor returns "succeeded", "failed", or "not found".
4. Update the charge accordingly.
```

If the processor doesn't support querying by your idempotency_key, you might:
- Wait for the processor's daily settlement file.
- Reconcile then.
- Show "processing" to the user in the meantime.

This is **the most painful part of payments**. Most outages and bugs live here.

## Deep dive: webhook events

Merchants subscribe to events (`charge.succeeded`, `charge.failed`, `refund.created`, etc.).

```
event store: append-only
  on state change → insert event
  webhook delivery worker: send to merchant URL with signature
  retry with exponential backoff on failure (24h+ window)
  if still failing: DLQ + manual alert
```

Sign events (HMAC-SHA256) so merchants can verify authenticity.

Make webhooks **idempotent at the merchant side**: include an `event_id` so the merchant can dedupe. Stripe does this.

## Deep dive: PCI-DSS isolation

Card numbers (PANs) are highly regulated. Mishandling them = massive fines + breach reporting.

Architecture: a small "vault" service is the **only** service that touches plaintext PANs.

```
Client → vault.tokenize(card_number) → "tok_visa_xyz" (random opaque token)
Client uses "tok_visa_xyz" for all subsequent operations.
Vault is the only place that maps token → encrypted PAN.
The vault is heavily isolated:
  - Separate VPC, separate audit, separate access controls.
  - Encryption at rest (HSM-backed).
  - Strict access logging.
```

This way, your main payment service (and DB, logs, metrics) never sees a real PAN. PCI scope is reduced to the vault only.

Stripe and others provide tokenization as a service — most companies just use that and never store cards themselves. **You should too** unless you're building Stripe.

## Deep dive: settlement and payouts

The processor doesn't pay merchants instantly. Funds are held, then settled (often T+1 or T+2 days), minus fees and chargebacks.

```
- Each completed charge increases merchant's "holding" balance in your ledger.
- Daily / weekly, run a payout worker:
    - Calculate available balance.
    - Trigger bank transfer to merchant's bank.
    - Record payout entry in ledger.
- Reconcile with processor reports daily.
```

Disputes / chargebacks can hit days/weeks later. They're tracked in the ledger as well:

```
chargeback:
  - debit merchant's holding account by disputed amount + fees
  - credit dispute reserve account
  - if won: reverse
  - if lost: credit customer's reversal account; cash to issuing bank
```

## Trade-offs discussion

### Why Postgres for the ledger?

ACID guarantees, mature, well-understood. The ledger is the most-correctness-critical part. Use a proven RDBMS.

For massive scale: shard by `account_id` or `transaction_id`. But payments don't have YouTube-scale traffic; one Postgres cluster goes a long way.

### Strong consistency throughout

No "eventual" anywhere near the money path. A charge must be in a definite state. Money is not a place for CAP trade-offs.

### Async path for non-critical work

Webhooks, fraud scoring, payouts → async. The user-facing charge path is synchronous up to "processor responded"; everything else is decoupled.

### Two-phase commit avoided

Cross-service ACID is painful (step 07 example 03). Instead: use **sagas** with state machines. Every transition is durable in the ledger. Recovery is "where am I in the state machine?".

### Outbox for webhooks

When recording a state change, also write an outbox row. A relay reads the outbox and dispatches webhooks. Ensures webhooks are never lost (no race between DB commit and webhook send).

## Failure modes

### Processor times out (the classic)

UNKNOWN state. Reconciliation queries clarify. User sees "Processing..." until resolved.

### Two charges to the same user, same amount, same second

Possible legitimately (oops, double click). Your idempotency key (passed by the merchant client or generated on form submit) prevents server-side duplication. If merchant doesn't send one, you can't fully protect.

### Card declines for "do not honor"

User-facing error. No money moved. Charge state = failed. User retries with another card.

### Processor outage

Brief: queue charges, retry. Long (minutes): show user "we're having trouble; please try again later". Optionally route to a backup processor (multi-acquirer routing).

### Internal DB outage

If the ledger is down, **don't proceed**. Refuse new charges. Show error. Better than processing a charge you can't record.

## Common follow-up questions

1. **"How do you prevent fraud?"**
   Fraud detection service consumes the same event stream (Kafka). Scores each transaction with rules + ML. Can hold or reject high-risk transactions, often before processor call.

2. **"How do you handle currency conversion?"**
   Convert at the time of the transaction at the published rate. Record both the original and converted amounts in the ledger.

3. **"How do you do refunds for partial failures?"**
   Refund only the captured portion. Use the same idempotency machinery — each refund is its own state-machine event with its own idempotency key.

4. **"Recurring billing / subscriptions?"**
   A separate scheduling system. Periodically, the scheduler picks up due subscriptions and triggers a charge through the same payment API.

5. **"How do you scale this?"**
   Vertical scaling first (Postgres can handle a lot). Then shard by `merchant_id` (most queries are scoped to a merchant). Most payment companies don't need YouTube-scale.

6. **"What about PSD2 / SCA / 3D Secure?"**
   Adds a step: "verify with bank" before completing. Adds latency and conditional UI (challenge prompts). State machine has additional state `authentication_required`. Doesn't change the core architecture.

7. **"How do you handle the 'magic disappearing money'?"**
   That's reconciliation. Daily, compare your ledger to processor reports. Investigate every discrepancy. This is where many payment teams spend most of their time.

## Key takeaways

- **Correctness > performance > scale.** This is the priority.
- **Idempotency keys at every layer.** End-to-end, from client to processor.
- **Double-entry ledger** is the immutable truth.
- **State machines for charges** track the journey explicitly.
- **Reconciliation** handles the unknown-state cases. Plan for them.
- **Webhooks via outbox** prevent silent loss.
- **PCI isolation** via tokenization vault. Don't store PANs unless you must.
- **Async everything not on the synchronous user-facing path.** Payouts, fraud, webhooks — all async.
- Stripe-grade payment systems are conceptually understandable. The engineering quality difference is in the reconciliation, edge cases, regulatory compliance, and reliability — not algorithmic complexity.
