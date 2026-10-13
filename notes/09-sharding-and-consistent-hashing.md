# 09 · Sharding (partitioning) and consistent hashing

Replication copies **all** data to each node. When the data or the write load is
**too big for one machine**, you **split** it: each node holds a **part**.
That's **sharding** (also called partitioning).

Like splitting a phone book into volumes A–F, G–M, N–Z, each on its own shelf.

Usually combined with replication: each shard has its own leader + followers.

## How to split: the shard key

Choosing the **shard key** (partition key) is the most important decision.
A good key:
- spreads data **and traffic** evenly,
- keeps data that's queried together **on the same shard**,
- matches the main access pattern.

### Strategies

| Strategy | How | ✅ | ❌ |
|---|---|---|---|
| **Range** | key ranges: users A–F → shard 1... or dates | range queries are easy | **hot spots** (everyone writes "today") |
| **Hash** | `hash(key) % N` | even spread | range queries hit all shards; **changing N moves almost everything** |
| **Consistent hashing** | ring (below) | adding/removing a node moves little data | a bit more complex |
| **Directory / lookup** | a table maps key → shard | flexible, can move single tenants | the directory is a dependency (cache it, replicate it) |
| **Geo** | by region/country | data near users, data-residency laws | uneven regions |

### Examples

- Chat messages: shard by `chat_id` — a conversation lives together.
- Social posts: shard by `user_id` — a user's posts together.
- Multi-tenant SaaS: shard by `tenant_id` (directory-based so a giant tenant can get its own shard).
- Time-series: hash(sensor_id) + time buckets, to avoid "everyone writes to the newest shard".

## The problems sharding brings

1. **Hot shards / celebrity problem** — one key gets huge traffic (Justin Bieber's
   followers). Fixes: split hot keys further (add a suffix: `user123#0..#9`),
   special handling for known-hot entities, caching.
2. **Cross-shard queries** — "top posts across all users" must ask every shard
   (scatter-gather) → slow. Precompute, or keep a separate index/aggregate.
3. **Cross-shard transactions** — hard; avoid by design, or use sagas (chapter 14) / distributed SQL.
4. **Joins** — across shards are expensive; denormalise.
5. **Rebalancing** — moving data when adding nodes, without downtime.
6. **Unique IDs** — auto-increment per database no longer works → use UUIDs or Snowflake IDs (case study).

**Don't shard early.** It adds lots of complexity. First: indexes, caching,
read replicas, vertical scaling, archiving old data.

## Consistent hashing

### The problem with `hash(key) % N`

With 4 servers, key → `hash % 4`. Add a 5th server → `hash % 5` → **~80% of keys
move**. For a cache, that's a near-total miss storm; for a database, a massive migration.

### The ring

Imagine the hash space as a **circle** (0 to 2³²−1). Place each **server** on the
ring by hashing its name. Place each **key** by hashing it, then walk
**clockwise** to the first server — that server owns the key.

```
              0
        S1 ●     ● key A  → owned by S2
      ●                ●
  key C                  S2 ●
  → S1 ●                    ●
          ●   S3 ●     ● key B → owned by S3
```

- **Add a server**: it takes over only the keys between it and the previous
  server — about **1/N** of the keys move.
- **Remove a server**: only its keys move to the next one.

### Virtual nodes

With few servers, the ring is uneven (one server owns a big arc). Fix: each
physical server appears at **many points** (e.g. 100–200 "virtual nodes").
Load evens out, and when a server dies its load spreads across **many** others
instead of dumping onto one neighbour. Bigger machines get more virtual nodes.

### Replication on the ring

Store each key on the owner **and the next N−1 distinct servers clockwise**
(Dynamo, Cassandra).

### Where it's used

Distributed caches (Memcached clients), Cassandra, DynamoDB, Riak, load
balancers that need key affinity, CDNs, chat-server routing.

> Alternative you may hear: **rendezvous (highest random weight) hashing** —
> for each key, score every server with `hash(key, server)` and pick the highest.
> Same "minimal movement" property, simpler when the server list is small.

## Remember

- Shard when one machine can't hold the data or writes; pick the shard key from the access pattern.
- Watch for hot shards, cross-shard queries/transactions, and rebalancing.
- Consistent hashing + virtual nodes = adding/removing nodes moves only ~1/N of keys.
