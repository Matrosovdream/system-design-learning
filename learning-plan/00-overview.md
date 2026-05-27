# 00 — Overview & Study Method

## Why this plan exists

You want to become a great architect. That means three things:
1. **Knowing the building blocks** — caches, queues, replicas, indexes, balancers.
2. **Knowing the trade-offs** — when each block helps, when it hurts, what it costs.
3. **Designing under constraint** — given product, scale, money and team, pick the right blocks.

This plan walks you from blocks → trade-offs → designs.

## How the plan is structured

```
learning-plan/
  progress.md                     ← track here
  00-overview.md                  ← this file
  01-fundamentals/
    README.md                     ← curriculum + key concepts for the step
    01-...md                      ← worked example #1
    02-...md                      ← worked example #2
    ...
  02-scaling-basics/
    README.md
    01-...md
    ...
  ...
  12-case-studies/
    README.md
    01-url-shortener.md
    ...
```

Each step folder contains:
- A `README.md` — the theory: what to learn, the key concepts, recommended reading.
- Numbered `.md` files — concrete worked examples that illustrate the concepts (no exercises, just illustrated explanations).

## How to ask Claude to teach you a step

When you're ready to study, type one of:

- **"Show me step 1"** / **"show me step 01"** — Claude reads `01-fundamentals/README.md` + all example files and displays them in chat, in order.
- **"Show me step 4 example 2"** — Claude shows just one example.
- **"Quiz me on step 3"** — Claude reads the step, then asks you questions to check understanding.
- **"I finished step 2"** — Claude marks step 02 as `done` in `progress.md` with today's date.

## Study method (recommended)

For each step:

1. **Read the README first** — understand the scope of what you're about to learn.
2. **Read the examples in order** — they are sequenced from simple to complex.
3. **Re-implement at least one example in your head, in your own words** — if you can't explain it back, you haven't learned it.
4. **Write down questions** in `progress.md` "Notes" section.
5. **Cross-reference with one canonical source** when something is unclear (see Sources below).
6. **Mark done** only when you can answer: *"What problem does this solve and what is the trade-off?"* for every concept in the step.

Pace: ~1 step per week, ~5-7 hours/week. The whole plan = ~3 months focused work.

## Canonical sources (the same 3 reappear in every step)

- **Primer** = [donnemartin/system-design-primer](https://github.com/donnemartin/system-design-primer) — free, broad, with case studies
- **DDIA** = [Designing Data-Intensive Applications](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/) by Martin Kleppmann — the deep one
- **Xu V1/V2** = [System Design Interview Vol 1 & 2](https://blog.bytebytego.com/p/system-design-interview-books-volume) by Alex Xu — case study style

Plus, used occasionally:
- [roadmap.sh/system-design](https://roadmap.sh/system-design)
- [roadmap.sh/software-architect](https://roadmap.sh/software-architect)
- ByteByteGo YouTube channel
- AWS / GCP architecture center articles

## Your stack: PHP / Go

This is relevant because:
- **Go** examples will lean on Go's concurrency model (goroutines, channels) when discussing things like worker pools, fan-out, leader election.
- **PHP** examples will mostly appear in the API / web-tier / monolith-decomposition sections, because PHP dominates that space.
- For data-heavy or distributed primitives (consistent hashing, Raft, log-structured storage), the language barely matters — concepts come first.

You won't write code as part of this plan (you opted to skip practice). But when examples show code, expect Go for concurrency/distributed bits, PHP for web/API bits.

## What "done" looks like

By the end of step 12, you should be able to:
1. Estimate the capacity needs of any reasonable product, in your head, in 5 minutes.
2. Draw the architecture of any well-known system (Twitter, Uber, WhatsApp, Stripe) on a whiteboard.
3. Defend every choice in that architecture with at least one trade-off you considered.
4. Identify the **first** thing to scale when a given system hits 10× the load.
5. Read a system-design RFC at work and spot at least one issue or improvement.

That's the architect bar. Let's go.
