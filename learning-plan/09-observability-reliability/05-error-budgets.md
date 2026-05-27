# Example 05 — Error budgets: the SRE management technique

A way to turn reliability from "infinite goal" into a measurable, tradeable resource.

You saw the basic SLA/SLO/SLI definitions in step 01 example 04. This example digs into how error budgets actually function as a tool.

## The setup

You have an SLO: "99.9% of requests succeed (return < 5xx)".

That means you're **allowed** 0.1% failure. Over a 30-day month with 100M requests:
```
0.1% × 100M = 100,000 errors permitted
                ⇡
            your error budget
```

The error budget is a finite, monthly resource.

## The deal

**With the team:**

- Spend the budget however you like:
  - Risky launches.
  - Deploys during business hours.
  - A/B test that briefly increases error rate.
- If you have budget remaining → keep shipping fast.
- If you burn through it → **stop launching, focus on reliability** until next month or the SLO is back in shape.

**With leadership:**

- Reliability is no longer "be as reliable as possible". It's "stay at the agreed SLO".
- You won't promise 100%, because 100% is infinitely expensive.
- You won't promise 99.9999% if the customer would be fine with 99.9%.

This converts the eternal reliability-vs-velocity argument into **a finite, negotiable resource**.

## What "spending the budget" looks like

| Event                                          | Budget impact            |
|------------------------------------------------|--------------------------|
| Routine traffic; some errors                   | Small steady spend       |
| Buggy deploy → 30 min of 0.5% extra errors     | Small chunk              |
| Outage → 2 hours of 100% errors                | Big chunk (maybe entire month) |
| Risky A/B test → 0.1% extra errors for a week  | Moderate spend           |
| Burndown DB migration with brief locks         | Small spend              |

## What "out of budget" means

Possible policies:

- **Soft**: only deploy bug fixes. No new features until budget rebuilds.
- **Medium**: rollback all changes shipped this month that caused failures. Engineer on-call doesn't take on new work.
- **Hard**: production freeze until end of month. All team focus on reliability.

The point is: **the policy is pre-agreed**. When the budget is gone, you don't argue about whether to slow down — you do.

## Why this works

### It aligns incentives

Without error budgets:
- Product wants speed.
- Ops wants reliability.
- They fight.

With error budgets:
- They share a number (the SLO).
- Speed is fine as long as the number is met.
- Reliability is fine if there's budget room for risky changes.

They cooperate around the budget instead of fighting against each other.

### It puts a price on "more 9s"

If the team is consistently coming in at 99.95% (much better than SLO 99.9%), maybe you over-engineered. You could relax: ship more, accept more failure.

If the team is consistently at 99.5% (missing SLO of 99.9%), something is broken — engineering investment is needed.

### It catches "death by a thousand cuts"

A single 1-hour outage = obvious. But 30 days of 30-minute hiccups can be equally bad while looking "fine". The budget aggregates everything; the math doesn't lie.

## How to compute it

Pick:

1. **Time window**: usually 30 days (rolling) or calendar month.
2. **SLI**: what counts as "good"? E.g., HTTP 2xx/3xx within 500 ms.
3. **SLO target**: e.g., 99.9% good.
4. **Error budget**: 100% - SLO = 0.1%.

Tracking:

```
good_requests_30d = sum of good in last 30 days
total_requests_30d = sum of all in last 30 days
slo_target = 0.999

budget_total = total_requests_30d × (1 - slo_target)
budget_consumed = total_requests_30d - good_requests_30d
budget_remaining = budget_total - budget_consumed
budget_remaining_percent = budget_remaining / budget_total
```

Dashboard this. Alert when remaining drops below thresholds (50%, 25%, 0%).

## Pitfalls

### Picking the wrong SLI

If your SLI is "5xx rate from the load balancer", you miss the bug where the app returns 200 with a JSON error body. **Pick SLIs the user actually cares about.**

For a payment API: success means "payment was charged correctly and customer was notified". Not just "we returned 200".

### Setting unrealistic SLO

If you set 99.99% but your current state is 99.9%, you'll be out of budget every month. Set the SLO based on **what users need + what you can sustain**. Adjust over time.

### No budget policy

You define the budget but don't define what to do when it's gone. Leadership says "ship anyway". Now the budget is theater.

The policy must be **agreed beforehand** and **enforced**.

### Burning the budget early in the month

Big outage on day 3 → budget gone for the month. Team can't ship anything for 27 days. Demoralizing.

Mitigations:
- **Burn-rate alerts**: alert not just when remaining < X%, but when the burn **rate** is too high (e.g., spending 1 week's budget per day).
- **Multi-window targets**: short windows (1h, 6h, 24h) trigger immediate response; long windows (30d) trigger policy changes.

### Different SLOs for different things

You have one SLO for "checkout" (must be 99.95%) and one for "profile picture upload" (acceptable at 99.5%). One uptime number can't be the whole story.

## A real example: a payment API

SLO definitions:

| SLI                                  | Target          | Budget (30 day) at 100M req      |
|--------------------------------------|------------------|----------------------------------|
| Successful charges                   | 99.95%           | 50,000 failures allowed          |
| Latency p95 < 500ms                  | 99%              | 1M slow requests allowed         |
| Webhook delivery within 1 min        | 99.9%            | 100,000 late deliveries allowed  |

Each has its own budget. Stripe-style: you don't ship riskier changes when payment SLO is burning, even if latency SLO is healthy.

Policy:

- **< 25% budget remaining**: production freeze for non-essential changes.
- **< 10%**: only fixes for issues that caused the burn.
- **0%**: all-hands reliability sprint, postmortem-driven action items.

## The "right" SLO

The SLO that:
- The customer's experience requires (not less).
- The team can sustain without burnout (not more).
- The business can afford engineering-wise.

For most non-critical SaaS: **99.9% is a good default**.
For latency-sensitive consumer apps: **99.95%+**.
For mission-critical (banking, healthcare): **99.99%+**, and budget for the cost.

## Architect's takeaway

- **Error budgets convert "reliability vs velocity" into a number both sides can read.**
- **Pick SLIs users actually care about.** Not "5xx rate"; user-perceived success.
- **Pre-agree the budget policy.** What happens when it's gone? Decide ahead of time.
- **Multi-window burn-rate alerts** prevent surprise budget exhaustion.
- **Different SLOs for different journeys** — one number doesn't cover everything.
- **Tracking budget consumption is a key reliability metric.** Dashboard it; review it weekly.
