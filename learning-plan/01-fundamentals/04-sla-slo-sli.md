# Example 04 — SLA, SLO, SLI: the three letters

Three terms that look similar but mean very different things. You need to use them correctly because they're the language of every reliability conversation at a real company.

## Definitions in one line

- **SLI** — *Service Level Indicator*: the **measurement**. ("99.93% of requests succeeded last month.")
- **SLO** — *Service Level Objective*: the **internal target**. ("We aim for 99.95%.")
- **SLA** — *Service Level Agreement*: the **contract**, with **financial consequences**. ("We promise customers 99.9% — refund if we miss.")

Mnemonic: **I-O-A** → **Indicator → Objective → Agreement**, going from internal measurement outward to customer contract.

## The relationship

```
SLI (real measurement)  →  SLO (what we target internally)  →  SLA (what we promise externally)

Stricter ────────────────────────────────────────────────► Looser
99.95% actual          99.9% target                       99.5% promised
```

You always promise less than you target, and target stricter than you measure to, so you have buffer.

## Real numbers: what each "9" buys you

For 1 year of uptime:

| Availability | Downtime per year | Per month  | Per week  | Per day  |
|--------------|-------------------|------------|-----------|----------|
| 99%          | 3.65 days         | 7.2 hours  | 1.7 hours | 14.4 min |
| 99.9%        | 8.77 hours        | 43.8 min   | 10.1 min  | 1.44 min |
| 99.95%       | 4.38 hours        | 21.9 min   | 5 min     | 43 sec   |
| 99.99%       | 52.6 min          | 4.4 min    | 1 min     | 8.6 sec  |
| 99.999%      | 5.26 min          | 26.3 sec   | 6.05 sec  | 0.86 sec |

> "Five nines" (99.999%) = ~5 minutes of downtime per year. To hit this you need multi-region, active-active, automated failover, no manual deploys to prod during business hours.

## Real-world SLAs

| Service                | SLA                | Penalty if missed              |
|------------------------|--------------------|--------------------------------|
| AWS S3 Standard        | 99.9%              | Service credits, tiered        |
| AWS EC2 (single region)| 99.99%             | Service credits, tiered        |
| GCP Cloud SQL HA       | 99.95%             | Service credits, tiered        |
| Cloudflare WAF         | 100% (no downtime) | Credits if any downtime        |
| Stripe API             | No public SLA      | (Enterprise contracts custom)  |

> Cloudflare promises 100% — they get away with it because the penalty is small credits, and they're so redundant that real outages are rare.

## SLI examples

Pick SLIs that **the user actually cares about**, not what's easy to measure.

| Bad SLI               | Better SLI                                         |
|-----------------------|----------------------------------------------------|
| Server CPU < 80%      | 99.9% of API requests return < 500 ms              |
| Disk usage < 90%      | 99.95% of writes are durable (not lost)            |
| 200 response rate     | 99.9% of *user-facing* requests succeed at the edge |

## Error budgets — the SRE killer feature

If your SLO is **99.9%** availability for the month, you're allowed **0.1% downtime** = ~**43.8 minutes**.

This 43.8 minutes is your **error budget**. SRE practice (popularized by Google):

- If you've consumed less than the budget → you can ship risky features, do experiments, deploy faster.
- If you've consumed more → freeze releases, focus on reliability work until you're back under.

This converts reliability from "infinite goal" into a **negotiable resource** the team can spend on velocity.

## Architect's takeaway

- Use **SLI** when you mean "what we observe".
- Use **SLO** when you mean "what we aim for".
- Use **SLA** only when there's a contract or it's customer-facing.
- **Always target ≥ 1 nine higher than you promise.** If your SLA is 99.9%, your SLO should be 99.95%+.
- More 9s = exponentially more expensive. Don't promise 99.99% if 99.9% is acceptable.
- An "error budget" is a powerful way to **reconcile feature velocity and reliability work**.

## Reading

- *Site Reliability Engineering* (Google SRE book) — Chapter 3 (Embracing Risk) and Chapter 4 (Service Level Objectives). Free online: https://sre.google/sre-book/
