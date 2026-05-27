# Step 01 — Fundamentals

Before you can design distributed systems, you need a shared vocabulary about how a single request travels across the internet, how fast/slow each step is, and what words like "reliable" or "scalable" actually mean.

## Goals

By the end of this step you should be able to:
- Describe what happens when you type `https://twitter.com` into a browser, end to end.
- Recite (within an order of magnitude) the latency of memory access, SSD read, network round-trip cross-continent.
- Do a back-of-envelope capacity estimation for a product with N users.
- Explain the difference between **SLA**, **SLO**, **SLI** with an example.
- Pick between **TCP** and **UDP** for a given use case and justify it.
- Define **reliability**, **scalability**, **maintainability** in one sentence each.

## Key concepts

1. **The request lifecycle** — DNS → TCP → TLS → HTTP → server → response. Each step takes time and can fail.
2. **Latency vs throughput** — latency is "how long for one request"; throughput is "how many requests per second". Optimizing one often hurts the other.
3. **Latency numbers every programmer should know** — cache, RAM, SSD, disk, network. These set the budget for every design.
4. **Back-of-envelope (BoE) estimation** — given DAU, write/read ratio, average payload size → storage/year, QPS, bandwidth.
5. **SLA / SLO / SLI** — the contract, the target, the measurement. Different things.
6. **Reliability, scalability, maintainability** — DDIA Ch.1's three non-functional requirements.
7. **Transport-layer choices** — TCP (ordered, reliable, slow handshake) vs UDP (best-effort, fast, app-controlled).

## Reading

- **Primer**: "How do I prepare for a system design interview?" + "Performance vs scalability" + "Latency vs throughput" + "Availability vs consistency" sections.
- **DDIA**: Chapter 1 — *Reliable, Scalable, and Maintainable Applications*.
- **Xu V1**: Chapter 1 (*Scale from Zero to Millions of Users*) and Chapter 2 (*Back-of-the-envelope Estimation*).

## Examples in this folder

- `01-http-request-lifecycle.md` — what really happens when you open a webpage.
- `02-latency-numbers.md` — Jeff Dean's numbers, scaled to human time.
- `03-back-of-envelope-estimation.md` — Twitter capacity walk-through.
- `04-sla-slo-sli.md` — the three letters, demystified with real numbers.
- `05-tcp-vs-udp.md` — when each one is correct.

## Self-check

After reading, you should be able to answer in 2-3 sentences each:
1. Why is a TCP connection across the Atlantic "expensive"?
2. If a product has 100M DAU and each writes 2 KB/day, how much storage per year?
3. AWS S3 claims "99.99% availability SLA". How many minutes of downtime is that per year?
4. Why doesn't Zoom use TCP for the audio stream?
5. What's the difference between "the system is scalable" and "the system is fast"?
