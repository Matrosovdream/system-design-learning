# Example 03 — Back-of-the-envelope estimation: a Twitter-sized service

Goal: starting from product requirements, estimate **storage, QPS, bandwidth** in 5 minutes. This is what you do live in a system-design interview, and what you sketch on a napkin before any real architecture meeting.

## The product (assumed)

A Twitter-like service:
- **300 M DAU** (daily active users).
- Each user posts on average **2 tweets/day**.
- Each user reads on average **100 tweets/day** (timeline scrolls).
- Tweet body: **280 chars** ≈ 280 bytes text.
- 10% of tweets have an image, average size **200 KB**.

## Step 1: Writes per second (QPS)

```
writes/day = 300M users × 2 tweets = 600M tweets/day
writes/sec = 600M / 86,400 sec ≈ 7,000 tweets/sec
```

But traffic is not uniform — peak is usually ~**2-3× average**. So design for:

```
peak write QPS ≈ 20,000 tweets/sec
```

## Step 2: Reads per second (QPS)

```
reads/day = 300M users × 100 tweets read = 30B reads/day
reads/sec = 30B / 86,400 ≈ 350,000 reads/sec
peak read QPS ≈ 1,000,000 reads/sec
```

**Read:write ratio ≈ 50:1** → read-heavy. This drives the design: aggressive caching, read replicas, denormalized feeds.

## Step 3: Storage per day

**Text:**
```
600M tweets × 280 bytes ≈ 168 GB/day
```
Add metadata (id, author_id, timestamps, likes_count, retweets_count, etc.) — call it **~500 bytes/tweet** total:
```
600M × 500 bytes = 300 GB/day of structured data
```

**Images:**
```
10% of 600M = 60M images/day
60M × 200 KB = 12 TB/day of blob data
```

## Step 4: Storage per year

```
Tweets:  300 GB × 365 ≈ 110 TB/year
Images:  12 TB × 365  ≈ 4.4 PB/year
```

> One year of images = **4.4 petabytes**. You're not putting this in Postgres. Object storage (S3-class) required.

## Step 5: Bandwidth

**Outbound (read-heavy):**
```
reads/sec at peak = 1M
avg payload per tweet = 280 B text + (10% × 200 KB image) ≈ 20 KB
bandwidth = 1M × 20 KB = 20 GB/sec outbound at peak
```

That's **160 Gbps**. You don't serve this from one origin. You **need a CDN** to absorb image bandwidth at the edge.

## Step 6: Cache sizing

80/20 rule of thumb: 20% of tweets get 80% of reads. Cache the hot 20%.

```
20% of 600M new tweets/day = 120M tweets/day cached
× 500 bytes = 60 GB/day of cache footprint, just for that day's writes
```

If you cache the last 3 days:
```
~180 GB cache
```

That fits in a Redis cluster of 5-10 nodes with ~64 GB RAM each. Doable.

## What this estimate tells the architect

From 6 numbers (DAU, write rate, read rate, payload, image %, image size) you can already decide:

1. **Read-heavy by 50:1** → cache layer is mandatory, read replicas are mandatory.
2. **300 GB/day structured + 12 TB/day blob** → split storage: SQL/NoSQL for tweets, object store for media.
3. **20 GB/sec outbound at peak** → CDN is non-negotiable.
4. **~1M read QPS peak** → can't serve from one DB; sharding + cache + replicas all needed.
5. **180 GB hot cache** → fits in a small Redis cluster.

You haven't drawn a single box yet, and you already know the shape of the system. **That's what BoE buys you.**

## Useful constants to memorize

- 1 day = **86,400 seconds** (round to 100k for math).
- 1 year = **31.5M seconds** (round to ~30M).
- KB = 10³, MB = 10⁶, GB = 10⁹, TB = 10¹², PB = 10¹⁵.
- A modern server: ~32-64 GB RAM, ~1-10 TB SSD, ~10 Gbps NIC.
- A modern DB box can comfortably handle ~5-10k QPS reads, ~1-5k QPS writes (with caveats).
- Peak / average ratio: assume **~2-3×** unless you know otherwise.

## Reading

- Xu V1 Chapter 2 (*Back-of-the-envelope Estimation*) is exactly this exercise, in book form.
