# Example 02 — Latency numbers every architect should know

Jeff Dean (Google) published these numbers and they're the foundation of every capacity discussion. Memorize the orders of magnitude.

## The raw numbers (rounded)

| Operation                          | Latency        | Relative |
|------------------------------------|----------------|----------|
| L1 cache reference                 | 0.5 ns         | 1×       |
| Branch mispredict                  | 5 ns           | 10×      |
| L2 cache reference                 | 7 ns           | 14×      |
| Mutex lock/unlock                  | 25 ns          | 50×      |
| Main memory reference              | 100 ns         | 200×     |
| Compress 1 KB with Snappy          | 3,000 ns       | 6,000×   |
| Send 1 KB over 1 Gbps network      | 10,000 ns      | 20,000×  |
| Read 4 KB from SSD                 | 150,000 ns     | 300,000× |
| Read 1 MB sequentially from RAM    | 250,000 ns     | 500,000× |
| Round-trip in same datacenter      | 500,000 ns     | 1,000,000× |
| Read 1 MB sequentially from SSD    | 1,000,000 ns   | 2,000,000× |
| Disk seek (HDD)                    | 10,000,000 ns  | 20,000,000× |
| Read 1 MB sequentially from HDD    | 20,000,000 ns  | 40,000,000× |
| Send packet US → Europe → US       | 150,000,000 ns | 300,000,000× |

## Scaled to human time

If 1 ns = 1 second:

| Operation                          | Human time      |
|------------------------------------|-----------------|
| L1 cache                           | 0.5 seconds     |
| Main memory                        | 1.5 minutes     |
| SSD read                           | 1.7 days        |
| Same-DC network round-trip         | 5.8 days        |
| SSD sequential 1 MB                | 11.5 days       |
| HDD seek                           | 4 months        |
| US → Europe round-trip             | **4.75 years**  |

> A cross-Atlantic packet round-trip, in human time, takes nearly 5 years. Caching matters.

## What to memorize (the architect's cheat sheet)

You don't need exact numbers. You need orders of magnitude:

- **RAM is ~100× slower than L1 cache, but ~100× faster than SSD.**
- **SSD is ~100× faster than HDD seek.**
- **Same-DC round-trip ≈ 0.5 ms. Cross-continent ≈ 100-200 ms.**
- **1 GbE network: ~125 MB/s. 10 GbE: ~1.25 GB/s.**
- **A spinning HDD does ~100 random IOPS. A consumer SSD does ~10,000. An NVMe SSD does ~100,000+.**

## Why these numbers shape design

**"Just put it in cache"** is good advice because cache hits (RAM) are 100-1000× faster than DB reads (SSD + network).

**"Just shard the DB"** doesn't make any single query faster — it adds an extra hop. But it parallelizes throughput.

**"Just use a CDN"** is good for global apps because the round-trip from Sydney to a US server is ~150 ms; from Sydney to a Sydney edge node is ~5 ms.

**"Move computation to the data"** (e.g., Hadoop/Spark) exists because sending 1 TB over the network takes hours; running code on the node that already has the data takes seconds.

## Worked example: should I cache user profiles?

You have a user-profile lookup: `getUser(id)`. The DB query takes 5 ms. Calling Redis takes 0.5 ms.

- Without cache: 1M lookups/day × 5 ms = **5,000 seconds of DB time/day**.
- With cache (90% hit rate): 100K DB hits × 5 ms + 1M Redis hits × 0.5 ms = **500 + 500 = 1,000 seconds**.

5× reduction in DB load, plus each request is 10× faster on a hit. Almost always worth it.

## Reading

- The original Jeff Dean numbers: search "latency numbers every programmer should know" (Peter Norvig's site has them).
- *Systems Performance* by Brendan Gregg if you want to dive deep.
