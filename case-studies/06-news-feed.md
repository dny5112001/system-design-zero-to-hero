# Case study 06 · News feed (Twitter/X timeline, Instagram, Facebook)

**Difficulty:** ⭐⭐⭐ medium–hard. The famous one. Tests: fan-out, caching, the celebrity problem, read-heavy optimisation.

## 1. Requirements

**Functional**
- Users **post** (text + optional media).
- Users **follow** others.
- **Home feed**: recent posts from people you follow, newest first (ranking optional).

**Non-functional**
- 200M DAU; feed loads in **< 200 ms**.
- Highly available; **eventual consistency OK** (a post may take a few seconds to appear).
- Very read-heavy.

## 2. Estimates

```
Posts: 200M × 0.5/day = 100M/day      → ~1,000 writes/s (peak ~3k)
Feed reads: 200M × 20/day = 4B/day     → ~40,000 reads/s (peak ~100k)
Avg follows: 200; some accounts have 100M+ followers
Post metadata 1 KB → 100 GB/day; media via object storage + CDN
```

Read:write ≈ 40:1 → **optimise reads**.

## 3. API

```
POST /v1/posts                 { text, media_ids[] } → { post_id }
POST /v1/users/{id}/follow
GET  /v1/feed?cursor=&limit=20 → { posts[], next_cursor }
```

## 4. Data model

```
users(id, name, ...)                          SQL
follows(follower_id, followee_id, created_at) SQL/graph; indexed both ways
posts(post_id (Snowflake), author_id, text, media, created_at)  Cassandra/sharded SQL by author_id
feed cache: user_id → [post_id, post_id, ...] (latest ~800)       Redis
media: S3 + CDN
```

## 5. The core decision: how do feeds get built?

### Option A — Fan-out on **read** (pull)
When Asha opens her feed: get everyone she follows → fetch their recent posts →
merge → sort.
- ✅ Writes are cheap; no wasted work for inactive users.
- ❌ Slow reads: 200 followees × queries, at 100k feeds/s. Too slow at scale.

### Option B — Fan-out on **write** (push)
When Ravi posts: look up his followers → **append the post ID to each follower's
feed list** in Redis. Reading a feed = read one precomputed list.
- ✅ Very fast reads.
- ❌ Celebrity posts → 100M feed writes per post (**hot write storm**); wasted work for inactive users.

### Option C — **Hybrid** ⭐ (the expected answer)
- Normal users (< ~10k followers): **push** to followers' feeds.
- **Celebrities**: **don't** fan out. At read time, **merge** the precomputed feed with recent posts from the few celebrities the user follows (pull).
- Skip pushing to long-inactive users; rebuild their feed on next login.

## 6. High-level design

```
POST /posts ─► Post service ─► posts DB  ─► event "PostCreated" ─► Kafka
                                                          │
                                                 Fan-out workers
                                (get followers from Follow service, skip celebrities/inactive)
                                                          │
                                                 Redis feed lists (per user, capped)

GET /feed ─► Feed service ─► Redis feed list (post IDs)
                          + recent posts of followed celebrities (pull, cached)
                          ─► merge, (rank) ─► hydrate: post cache / user cache ─► response
                                                       media URLs → CDN
```

## 7. Deep dives

### Feed storage
Store only **post IDs** (8 bytes) per entry, cap ~800 per user: 200M users ×
800 × 8 B ≈ **1.3 TB** of Redis → a sharded cluster (by user_id). **Hydrate**
(fill in text, author, counts) from a post cache / user cache with batched lookups.

### Pagination
Cursor = last seen post ID (Snowflake IDs are time-ordered) → stable as new posts arrive.

### Ranking (if asked)
Candidate generation (the feed list + celebrity posts + maybe recommendations) →
ranking model scores (relationship strength, engagement prediction, recency) →
rerank for diversity. Precompute features; time-box ranking (fallback to chronological).

### Counters (likes, views)
Don't update a DB row per like at viral scale. Increment in Redis / aggregate
events from Kafka and flush periodically; approximate is fine.

### Media
Upload to S3 via pre-signed URL; thumbnails async; served by CDN.

### Deleting / editing a post
Mark deleted in the posts DB; feeds still hold the ID → filter on hydration
(and lazily clean feeds).

### New follow / unfollow
Follow: backfill a few recent posts into the feed. Unfollow: filter at read time; clean up lazily.

## 8. Wrap-up

- Hot spots: celebrities (hybrid), viral posts (cache post objects, replicate hot keys).
- Failures: if fan-out lags, feeds are a little stale — acceptable per requirements; Kafka buffers.
- Monitor: fan-out lag, feed p99 latency, cache hit ratio.

## Follow-ups

- "What's the threshold for celebrity?" (Tune it — e.g. > 10k–100k followers — based on fan-out cost.)
- "User follows 5,000 accounts?" (Cap merges; precomputed feed handles it.)
- "How to show 'new posts available'?" (Poll a lightweight endpoint or push via WebSocket/SSE.)
