# Case study 15 · Real-time leaderboard and top-K

**Difficulty:** ⭐⭐ medium. Tests: the right data structure (sorted sets), sharding a ranking, approximate counting.

## Part A — Game leaderboard

### 1. Requirements
- Players earn points; show the **top 10** and **my rank** (+ players around me).
- Leaderboards: global, per day/week/season, per region/friends.
- Updates visible in near real time.
- Scale: 50M players, 5M DAU, 50M score updates/day.

### 2. Estimates
```
50M updates/day ≈ 500 writes/s (peak ~2–5k/s, more at tournament end)
Reads (top 10 / my rank): maybe 10× writes → ~5k–20k/s
```

### 3. Why not SQL?
`SELECT COUNT(*) FROM scores WHERE score > my_score` for rank on 50M rows, thousands of
times per second → too slow (even with an index, ranking needs counting).

### 4. The right tool: Redis sorted set ⭐
A sorted set keeps members ordered by score (a skip list inside): updates and rank
lookups are **O(log N)**.

```
ZINCRBY lb:season:12 50 "player:42"         # add 50 points
ZREVRANGE lb:season:12 0 9 WITHSCORES        # top 10
ZREVRANK lb:season:12 "player:42"            # my rank (0-based)
ZREVRANGE lb:season:12 (rank-5) (rank+5)     # players around me
```

50M members × ~100 bytes ≈ 5 GB → fits in one Redis instance (with a replica).

### 5. Design
```
game servers ─► Score service ─► validate (anti-cheat) ─► DB (source of truth: score events/totals)
                                         └─► Redis sorted sets (global, daily, weekly, per region)
clients ─► Leaderboard API ─► Redis (top-N cached a few seconds; rank on demand) ─► profile cache for names/avatars
```
- **Time windows**: separate keys (`lb:daily:2026-10-13`) with TTL; or compute weekly from daily.
- **Ties**: encode a tiebreaker into the score (e.g. score × 10¹⁰ + (MAX_TS − timestamp) so earlier achievers rank higher).
- **Friends leaderboard**: fetch friends' scores (`ZMSCORE`) and sort in the app (friend lists are small).
- **Durability**: Redis can be rebuilt from the DB; the DB is the source of truth.

### 6. Scaling beyond one Redis
- **Shard by score range**: shard 1 holds 0–1,000, shard 2 holds 1,001–5,000... rank =
  (players in higher shards) + rank within your shard. Keep per-shard counts. Rebalance as
  scores grow.
- **Hash-sharding by player**: top-K = take the top-K from each shard and merge; exact global
  rank is harder → approximate rank (percentile buckets: "top 3%") is often enough for
  millions of players.

## Part B — Top-K heavy hitters (trending hashtags, most-viewed videos)

"Top 100 hashtags in the last hour" over a stream of millions of events/s.

### Exact (smaller scale)
Stream processing (Flink/Kafka Streams): count per key in **tumbling/sliding windows**,
keep a **min-heap of size K** per window → emit top-K.

### Huge scale → approximate
- **Count-Min Sketch**: fixed-size 2-D array of counters; estimates each key's count
  (may overestimate slightly) in tiny memory.
- Combine with a heap of size K to track the current heavy hitters.
- **Partition** the stream by key across workers (each computes a local top-K), then a
  final aggregator merges the local top-K lists. (Merging local top-Ks is approximate —
  good enough for "trending".)

### Serving
Write results to Redis/a DB every minute; API reads the precomputed list (cached, CDN-friendly).

## Wrap-up
- Leaderboards: Redis sorted sets; windows as separate keys; shard by score range or accept approximate ranks.
- Trending/top-K: windowed stream counting + heaps; Count-Min Sketch at huge scale.

## Follow-ups
- "Cheating?" (Server-authoritative scores, validation, anomaly detection.)
- "Leaderboard across 100M players with exact rank?" (Score-range sharding with cumulative counts.)
