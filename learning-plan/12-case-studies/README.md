# Step 12 — Case studies (real interview-style walkthroughs)

You've covered the building blocks. This step is where you put them together. Each file is a full system-design walkthrough in the **interview format**:

1. **Clarifying questions** — never start solving without these.
2. **Functional & non-functional requirements** — what the system must do.
3. **Capacity estimation** — back-of-envelope from step 01.
4. **API design** — the contract with clients.
5. **High-level architecture** — the boxes and arrows.
6. **Data model** — what's stored where.
7. **Deep dive** — 1-2 critical components in detail.
8. **Trade-offs** — what we picked and why.
9. **Common follow-up questions** — what the interviewer asks next.

These are the ten most common system-design interview problems. After you work through them all, **most other system-design problems are recombinations of these ten**.

## How to use this step

For each case study:

1. **Read the problem statement.** Stop. Spend 5-10 minutes designing it yourself — pen and paper.
2. **Then** read the walkthrough.
3. Compare your design to the walkthrough. Where did you go differently? Was the walkthrough's choice better?
4. Note the trade-offs section — these are the lines the interviewer is testing.
5. Read the follow-up questions and think how you'd answer.

This step is not a passive read; it's where you **practice**.

## The 10 case studies

| #  | System                          | Highlights                                              |
|----|----------------------------------|---------------------------------------------------------|
| 01 | URL shortener (bit.ly)          | Read-heavy, simple, foundational                        |
| 02 | Distributed rate limiter        | Algorithms, Redis, exactness vs scale                   |
| 03 | Real-time chat (WhatsApp)        | WebSockets, message delivery, presence                  |
| 04 | News feed (Twitter timeline)     | Fan-out write/read, celebrity problem                   |
| 05 | Notification system              | Multi-channel, retries, scheduling                      |
| 06 | Distributed key-value store      | Consistent hashing, replication, gossip                 |
| 07 | S3-like object storage           | Massive scale, durability, metadata                     |
| 08 | Payment system (Stripe-like)     | ACID, idempotency, ledger, PCI                          |
| 09 | Google Maps / location service   | Geo indexing, real-time data, scale                     |
| 10 | Stock exchange / order matching  | Low latency, fairness, audit                            |

## Files in this folder

- `01-url-shortener.md`
- `02-rate-limiter.md`
- `03-chat-system.md`
- `04-news-feed.md`
- `05-notification-system.md`
- `06-key-value-store.md`
- `07-s3-storage.md`
- `08-payment-system.md`
- `09-maps-location.md`
- `10-stock-exchange.md`

## A general framework for any system-design interview

1. **Repeat the question back. Ask 3-5 clarifying questions.** Scale? Users? Reads vs writes? Geography? Latency? Consistency requirements? What's in/out of scope?
2. **State assumptions explicitly.** "I'll assume 100M DAU, 50:1 reads, sub-200ms p95."
3. **Estimate scale.** QPS, storage, bandwidth.
4. **Sketch high-level architecture.** ~5 boxes. Client → LB → app → cache → DB. Mark where things scale.
5. **Define APIs.** ~5 endpoints. Methods, paths, payloads.
6. **Data model.** Tables / collections / event types. Partition keys.
7. **Deep dive 1-2 areas the interviewer hints at.** Could be the feed-fanout, the consensus on a write, the rate limiter, etc.
8. **Discuss trade-offs.** "We chose X over Y because of the read-heavy nature." Show you can hold both alternatives in mind.
9. **Address obvious follow-ups proactively.** Failure modes, scaling boundaries, monitoring, the celebrity case.

The interviewer is testing whether you can structure ambiguity, make defensible choices, and discuss trade-offs. They are not testing whether you remember the exact algorithm for consistent hashing — they want to see you *think*.
