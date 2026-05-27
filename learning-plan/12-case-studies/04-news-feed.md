# Case Study 04 — Design a news feed (Twitter timeline, Facebook feed)

A classic asymmetric system. Reading is the dominant operation; the design decisions revolve around what to pre-compute, what to compute on read, and how to handle "celebrities".

## Problem statement

Build a system where:
- Users post content.
- Users follow other users.
- Each user has a "timeline" of recent posts from people they follow, sorted by recency (or some ranking).
- Timeline updates near-real-time as new posts appear.

## Clarifying questions

1. **Scale**: total users, DAU, posts/day, average followers/following.
2. **Ranking**: chronological or algorithmic?
3. **Media**: text only, or images/videos?
4. **Privacy**: public, private accounts, blocked users?
5. **Edit/delete**: can posts be edited/deleted?
6. **Pagination depth**: how far back can users scroll?
7. **Notifications**: separate from feed?

**Assumed answers:**

- 1B total users, 300M DAU.
- Average 200 followers; some users (celebrities) have 100M+ followers.
- 500M posts/day; each user reads ~100 posts/day (heavy read skew).
- Reverse-chronological feed (no algorithmic ranking for now).
- Text + media (images, videos in S3-like).
- Public accounts only.
- Edits and deletes within 24h.
- Pagination up to ~1000 posts back.
- Notifications are a separate system (see case 05).

## Functional requirements

- Post content.
- Follow / unfollow users.
- Read your timeline (most recent posts from people you follow).
- View any user's profile (their own posts).
- Post deletion and basic editing.

## Non-functional requirements

- Feed read latency p95 < 200 ms.
- Post latency p95 < 1 second.
- 99.99% uptime.
- New posts appear in followers' feeds within seconds.

## Capacity estimation

```
500M posts/day = ~6000 posts/sec average; peak ~20k/sec.

Reads:
  300M DAU × 100 reads/day = 30B reads/day ≈ ~350k/sec average; peak ~1M/sec.
  read:write ratio = 50:1. Read-heavy. ← drives the design.

Storage:
  500M × ~500 bytes/post = 250 GB/day text. ~90 TB/year text.
  Media: separate (S3-like).

Followers:
  Average 200; but a celebrity might have 100M.
  Total follower edges: ~1B × 200 = 200B edges. ~20 TB if naively stored.
```

## API design

```
POST /api/v1/posts          { body, media_ids? }
GET  /api/v1/feed           ?cursor=&limit=20   → list of posts (most recent first)
GET  /api/v1/users/{id}     → profile
POST /api/v1/users/{id}/follow
POST /api/v1/users/{id}/unfollow
GET  /api/v1/users/{id}/posts ?cursor=&limit=20
DELETE /api/v1/posts/{id}
```

## High-level architecture

```
[client]
   │
   ▼
[CDN: cached static + some public profiles]
   │
   ▼
[API gateway]
   │
   ├─► [Post service]
   │     ↓
   │   [Posts DB: Cassandra, partition by user_id]
   │     ↓
   │   publishes "post.created" to Kafka
   │
   ├─► [Follow service]
   │     ↓
   │   [Follow graph: Cassandra or graph DB]
   │
   └─► [Feed service]
         ↓
       Reads from [Feed cache: Redis sorted sets per user]
       Falls back to [Posts DB]
       Combines

Kafka "post.created" event:
   ↓
[Feed fanout workers]
   ↓
   Reads followers from follow graph
   Writes post_id into each follower's Redis sorted set
   (this is "fan-out on write" — see below)
```

## The crux: fan-out on write vs fan-out on read

This is **the central trade-off** in feed design.

### Fan-out on write (push model)

When user A posts, **immediately push the post ID into every follower's feed cache**.

```
User A posts.
  → write post to Posts DB.
  → look up A's followers.
  → for each follower F: ZADD feed:F <timestamp> <post_id>
```

**Reads are fast**: feed = `ZREVRANGE feed:F 0 19` — one Redis op.

**Writes are expensive for celebrities**: A celebrity with 100M followers triggers 100M Redis writes per post.

### Fan-out on read (pull model)

When user F reads their feed, **fetch posts from each followed user and merge**.

```
User F reads feed.
  → look up F's followings (e.g., 200 users).
  → query each: SELECT * FROM posts WHERE user_id = ? ORDER BY ts DESC LIMIT 20
  → merge results, sort by timestamp, return top 20.
```

**Writes are cheap**: just write the post once.

**Reads are expensive**: 200 DB queries per feed read. With 1M reads/sec → 200M DB ops/sec. Impractical.

### Hybrid (the real answer)

Used by Twitter, Facebook, etc.

```
- Normal users (< 10k followers): fan-out on write. Push to all follower feeds at post time.
- Celebrities (> 10k followers): fan-out on read. When a follower reads their feed:
    - Fetch most recent posts from celebrities the user follows (cache-aside).
    - Merge with the user's "pre-fanned-out" feed (from normal users).
    - Return top N.
```

This bounds the work:
- Posting as a normal user: cheap (push to a few followers).
- Posting as a celebrity: very cheap (no fan-out; followers pull on read).
- Reading: mostly fast (read pre-computed feed); plus a few celebrity-post fetches.

Most reads benefit from pre-computation; most writes don't have the celebrity penalty.

## Data model

### Posts (Cassandra, partition by user_id)

```sql
CREATE TABLE posts (
    user_id     UUID,
    post_id     TIMEUUID,
    body        TEXT,
    media_ids   LIST<UUID>,
    created_at  TIMESTAMP,
    PRIMARY KEY (user_id, post_id)
) WITH CLUSTERING ORDER BY (post_id DESC);
```

Read "all posts by user X" → one partition scan. Fast.

### Follow graph (Cassandra, two tables for two access patterns)

```sql
-- who am I following?
CREATE TABLE following (
    user_id     UUID,
    followed_id UUID,
    since       TIMESTAMP,
    PRIMARY KEY (user_id, followed_id)
);

-- who follows me?
CREATE TABLE followers (
    user_id     UUID,
    follower_id UUID,
    since       TIMESTAMP,
    PRIMARY KEY (user_id, follower_id)
);
```

Both relations stored — denormalized for read efficiency.

### Per-user feed (Redis sorted set)

```
key: "feed:<user_id>"
value: sorted set of (timestamp, post_id), capped at e.g. 1000 entries.
TTL: a few days (inactive users get evicted).
```

Bounded size (capacity-limited Redis). On insert beyond cap, drop oldest. When user reads "first 20 posts", `ZREVRANGE feed:<user_id> 0 19`.

## Deep dive: handling celebrities

Celebrity user posts once → can't write to 100M follower feeds quickly. Strategies:

### Skip fan-out for celebrities

Maintain a "celebrities" set (anyone with > 10k followers). When they post, skip fan-out. Their posts are merged on read.

### Merging on read

```
def get_feed(user_id, cursor=None, limit=20):
    # 1. fan-out portion: pre-computed
    base = redis.zrevrange(f"feed:{user_id}", 0, limit*2)  # over-fetch for merging
    
    # 2. celebrity posts: cache-aside
    celebs_followed = following_celebs(user_id)
    celeb_posts = []
    for celeb_id in celebs_followed:
        posts = cache.get(f"recent_posts:{celeb_id}")
        if not posts:
            posts = db.query("SELECT * FROM posts WHERE user_id = ? ORDER BY post_id DESC LIMIT 20", celeb_id)
            cache.set(f"recent_posts:{celeb_id}", posts, ttl=60)
        celeb_posts.extend(posts)
    
    # 3. merge by timestamp
    merged = merge_sorted(base + celeb_posts, key=lambda p: p.created_at, desc=True)
    return merged[:limit]
```

Note: `recent_posts:{celeb_id}` is cached aggressively. A celebrity's recent-posts list is read by millions; cache it hard, refresh on each new post.

## Deep dive: scaling fan-out workers

Posts are pulled from Kafka by fan-out workers. Each post causes N writes to N follower feeds. Workers must scale.

```
Kafka topic: posts.created
Partitions: 100+
Workers: pool, each consumes one or more partitions
For each post:
  fetch followers (with pagination if huge)
  for each follower:
    ZADD their feed in Redis
    ZREMRANGEBYRANK to cap at 1000 entries
```

Backpressure: if Redis is slow, workers fall behind. Kafka retention covers this; you can replay.

## Deep dive: deletes and edits

User deletes a post. The post is in many follower feeds.

Options:

### Option 1: Tombstone

Mark the post as deleted in the Posts DB. On feed read, fetch each post's actual content; skip if deleted. This means **every feed read also fetches post details** (which you'd do anyway) — the deletion is naturally honored.

This is what most systems do. The feed stores post IDs only; the bodies live in posts table.

### Option 2: Active eviction

Look up the post's deleter; find all their followers; remove from each follower's feed. Expensive for celebrities. Usually skipped.

For edits: similar to delete — feed stores IDs; reads fetch current content (which is the edited version).

## Trade-offs discussion

### Why Cassandra for posts and follow graph?

- Append-heavy (posts).
- Partition by user → all queries by user are single-partition.
- Linear scale.

A graph DB (Neo4j) would be nice for traversal queries ("friends of friends"), but for the feed use case, simple denormalized adjacency in Cassandra is enough.

### Why Redis sorted sets for feeds?

- O(log N) inserts (ZADD) and reads (ZREVRANGE).
- Built-in capacity caps.
- Fast enough for millions of feed reads/sec.

Alternatives: Memcached (no sorted sets, harder); a TimescaleDB or specialized store — not standard.

### Stale feeds for inactive users

If user F hasn't logged in for 6 months, do we still fan out every post to F's feed? No — TTL on the Redis key, lazy rebuild when F next logs in.

### Ranking

This design uses chronological. For algorithmic ranking (like real Twitter / Facebook), a separate **scoring/ranking service** consumes the feed candidate set and re-orders. Adds significant complexity (ML models, A/B testing infrastructure).

## Common follow-up questions

1. **"How would you add a 'who liked this' feature?"**
   Likes table partitioned by post_id. On read, "show top 5 likers" = single-partition query. Plus an aggregate count denormalized into the post.

2. **"What if a user follows 10,000 people?"**
   On feed read, the pre-computed Redis feed already covers them (because each of those 10k pushed into this user's feed when posting). It's the follower side that pays — but they only pay once on post, not once on read.

3. **"What about new users with no posts in their feed yet?"**
   Cold-start. Show suggested posts: top posts from popular users, region-based, etc. (Recommendation problem; separate.)

4. **"How do you handle a sudden viral post?"**
   The post itself is just a row. Reads of the post body get cached aggressively. Likes/comments are sharded by post_id, so a hot post → one hot shard. Mitigate with replica reads + L1 in-process caches. Same pattern as the hot-key problem.

5. **"What about consistency? If I unfollow someone, do I still see their posts in my feed?"**
   The post IDs are already in the Redis sorted set. We could either:
   - Live with it briefly (eventual consistency; the post will age out).
   - On unfollow, scan the user's feed and remove the unfollowed user's post IDs. Expensive but doable for the recent timeline.

   Most products go with "live with it briefly" + filter at read time as a safety net.

6. **"How do you support 'this person's profile' (their own posts only)?"**
   `SELECT * FROM posts WHERE user_id = ? ORDER BY post_id DESC LIMIT 20`. Single-partition query in Cassandra. Cache the recent slice.

7. **"How would you do real-time updates (new post badge)?"**
   WebSocket or SSE pushing "new post available" notifications. Or polling `/api/v1/feed?since=last_seen_id` periodically.

## Key takeaways

- **Read-heavy, write-cheap = pre-compute feeds.** Fan-out on write.
- **Celebrities break naive fan-out.** Switch to fan-out on read for them.
- **Hybrid is the standard.** Most systems combine.
- **Cassandra for posts and follow graph; Redis sorted sets for feeds.** Standard stack.
- **The hard problem isn't storage; it's the fan-out asymmetry.** Get that right and the rest follows.
- **Tombstones for deletes, fetch-current-on-read** — keeps the feed structure simple.
