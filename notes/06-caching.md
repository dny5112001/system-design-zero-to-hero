# 06 · Caching

A **cache** keeps a copy of frequently used data somewhere **faster** (usually
memory), so you don't redo slow work. Memory is ~1000× faster than a database
round trip on disk. Caching is often the **single biggest win** in a design.

Like keeping your most-used books on your desk instead of walking to the library.

## Where caches live

```
browser cache → CDN → API gateway/reverse proxy cache → app's in-memory cache → distributed cache (Redis) → database's own buffer cache
```

| Layer | Example | Note |
|---|---|---|
| Client | browser HTTP cache, mobile app storage | `Cache-Control`, `ETag` |
| CDN | static files, some API responses | near users |
| App memory | a small LRU map in each server | fastest; each server has its own copy (can disagree) |
| Distributed | **Redis**, Memcached | shared by all servers; the usual choice |
| Database | buffer pool, query cache | automatic |

## Caching strategies (know these by name)

### 1. Cache-aside (lazy loading) — the most common

```
read:  look in cache → hit? return it
                     → miss? read DB → put in cache (with TTL) → return
write: write DB → delete (invalidate) the cache key
```

- ✅ Only caches what's actually used; cache failure isn't fatal (fall back to DB).
- ❌ First read is slow (miss); small window where cache can be stale.
- Why **delete** on write instead of update? Two concurrent writers updating the
  cache can leave it with the older value; deleting is safer.

### 2. Read-through

The cache itself loads from the DB on a miss (the app only talks to the cache).
Same idea as cache-aside, logic moved into the cache layer/library.

### 3. Write-through

```
write: write to cache AND DB together (synchronously)
```
- ✅ Cache always fresh. ❌ Slower writes; caches data that may never be read.

### 4. Write-behind (write-back)

```
write: write to cache → return → cache writes to DB later in batches
```
- ✅ Very fast writes, absorbs bursts. ❌ **Data loss** if the cache dies before flushing. Use for counters/analytics, not money.

### 5. Write-around

Write only to the DB; cache fills on reads. Good when written data is rarely re-read soon.

## Eviction — what to throw out when full

| Policy | Throws out | Note |
|---|---|---|
| **LRU** (least recently used) | the item unused for longest | default choice |
| LFU (least frequently used) | the least popular | good for stable popularity |
| FIFO | the oldest | simple |
| TTL (time to live) | anything past its expiry | always set one, as a safety net |

## The hard parts (interviewers love these)

### Invalidation — "one of the two hard things"

How stale is acceptable? Options: short **TTLs**; **delete on write**; publish
change events (CDC from the database → invalidate keys); **versioned keys**
(`user:42:v7`).

### Cache stampede (thundering herd)

A hot key expires → thousands of requests miss at once → all hit the database.

Fixes:
- **Lock / single-flight**: only one request rebuilds; others wait or get the old value.
- **Early/probabilistic refresh**: refresh before expiry.
- **Jittered TTLs**: don't let many keys expire at the same second.
- **Serve stale while revalidating.**

### Hot keys

One key (a celebrity's profile) gets so much traffic that one Redis node melts.
Fixes: local in-memory copy on each app server for very hot keys; replicate the
key under several names (`key#1..#N`) and read a random one; read replicas.

### Cache penetration

Requests for keys that **don't exist** always miss and hit the DB (often an attack).
Fixes: cache the "not found" result briefly; a **Bloom filter** of valid keys (chapter 15).

### Consistency

A cache adds a second copy → it can be stale. Say clearly: "users may see data
up to N seconds old, which we agreed is acceptable for X". For data that must be
exact (balances, inventory at checkout), **read from the database**.

## Redis in one paragraph

An in-memory data store with rich types: strings, hashes, lists, **sets**,
**sorted sets** (leaderboards!), streams, HyperLogLog, geospatial indexes,
pub/sub, Lua scripts, TTLs. Optional persistence (snapshots/AOF). Scales with
replication + **Redis Cluster** (sharding over 16,384 hash slots). Used for
caching, sessions, rate limiting, leaderboards, queues, distributed locks.
Memcached: simpler, multi-threaded, strings only, pure cache.

## Sizing a cache

The **80/20 rule**: ~20% of data gets ~80% of traffic. Cache the hot 20%.

```
Daily reads touch 1 B items × 1 KB; cache 20% → 200 GB → a few Redis nodes (with replicas)
```

Track the **hit ratio** (aim for > 90% on hot paths).

## Remember

- Cache-aside + TTL + delete-on-write is the default answer.
- Name the risks: staleness, stampede, hot keys, penetration — and their fixes.
- Never cache-only data you can't afford to lose (unless write-behind is a deliberate choice).
