# 15 · Advanced distributed systems toolkit

The topics senior/staff interviews dig into. You don't need the maths — you need
to know **what each tool solves** and **when to reach for it**.

## Consensus: Raft and Paxos

**Problem:** several machines must **agree** on one value (who's leader? what's
the next entry in the log?) even when some crash or messages get lost.

**Raft** (the understandable one):
- Nodes are **followers**, a **candidate**, or the **leader**.
- If followers stop hearing the leader's heartbeat, one becomes a candidate and
  asks for votes. **Majority wins** → new leader for a new **term**.
- The leader appends entries to its log and replicates them; an entry is
  **committed** once a **majority** has it.
- With 5 nodes, survives 2 failures (needs 3 = majority). Always use an **odd** number.

Used inside: **etcd** (Kubernetes' brain), Consul, CockroachDB, TiKV, Kafka
(KRaft). **ZooKeeper** uses ZAB (similar). Spanner uses Paxos.

**In designs**: you rarely implement it — you **use** etcd/ZooKeeper for
leader election, configuration, service discovery and locks.

## Leader election

Many systems want **exactly one** node doing something (the scheduler, the
compaction job). Use a lease in etcd/ZooKeeper/Consul (or a DB row with an
expiry): whoever holds the lease is leader; it must keep renewing it.

## Distributed locks — and why they're tricky

"Only one worker may process invoice 42."

- **Redis**: `SET lock:42 <random-token> NX PX 30000` (set if not exists, auto-expire), release only if the token matches (Lua script).
- **etcd/ZooKeeper**: stronger guarantees via consensus.

The trap: a process pauses (GC, network) longer than the lock's expiry, another
process takes the lock, then the first wakes up and writes anyway.
**Fix: fencing tokens** — every lock grant includes an increasing number; the
storage rejects writes with an older number.

Better still: **avoid needing a lock** — use database constraints, conditional
updates (`UPDATE ... WHERE version = 7`), or a single partition owner.

## Time and ordering

Clocks on different machines **drift**; you can't trust timestamps to order
events across machines.

| Tool | Gives |
|---|---|
| **Lamport clock** | a counter: if A happened-before B, then L(A) < L(B). Total order, but can't detect concurrency |
| **Vector clock** | one counter per node; can tell "A before B", "B before A", or **concurrent** (conflict!) — used for conflict detection in Dynamo-style stores |
| **Hybrid logical clocks** | physical time + logical counter (CockroachDB) |
| **TrueTime** (Google Spanner) | GPS/atomic clocks give a time *interval*; Spanner waits out the uncertainty → global strong consistency |

## Unique IDs at scale

Auto-increment doesn't work across shards. Options:
- **UUID v4**: random 128-bit; no coordination; not sortable; poor index locality.
- **UUID v7 / ULID**: time-ordered prefix + randomness → sortable, good for indexes.
- **Snowflake** (Twitter): 64-bit = 41 bits timestamp (ms) + 10 bits machine ID + 12 bits sequence → ~4,096 IDs per ms per machine, roughly time-sorted, fits in a `BIGINT`. (Case study.)

## Probabilistic data structures (huge scale, tiny memory)

| Structure | Answers | Trade-off | Used for |
|---|---|---|---|
| **Bloom filter** | "is X **possibly** in the set?" — no = definitely not, yes = probably | small false-positive rate, no false negatives | skip disk lookups for missing keys (Cassandra), "username taken?", crawler "seen this URL?", cache penetration |
| **HyperLogLog** | "how many **unique** items?" | ~1–2% error in ~12 KB | unique visitors, distinct searches |
| **Count-Min Sketch** | "how often did X occur?" | may overestimate | trending topics, heavy hitters, top-K |

## Data integrity and syncing

- **Merkle tree**: a tree of hashes; compare roots, then only the differing
  branches → find which data differs between replicas cheaply (Cassandra
  anti-entropy, Git, blockchains, file sync).
- **Gossip protocol**: each node regularly shares state with a few random peers;
  information spreads like a rumour. Used for membership and failure detection
  (Cassandra, Consul).
- **Checksums** to detect corruption.

## Geospatial indexing (for "nearby" queries)

- **Geohash**: encode lat/long into a string; nearby places share a prefix
  (`tdr1y...`). Search the cell + its 8 neighbours. Simple, works with any DB / Redis GEO.
- **Quadtree**: split the map into 4 squares recursively until each holds ≤ N
  places; dense cities get small cells. Good for uneven density.
- **S2 (Google) / H3 (Uber)**: sphere-aware cell systems; H3 uses hexagons
  (equal-distance neighbours) — great for ride-hailing and delivery.

## Storage engine internals (one level deeper)

| | B-tree | LSM-tree |
|---|---|---|
| Writes | update pages in place (random I/O) | append to memory table → flush sorted files → **compact** in background (sequential I/O) |
| Reads | fast, predictable | may check several files (Bloom filters help) |
| Used by | Postgres, MySQL/InnoDB | Cassandra, RocksDB, LevelDB, HBase |
| Best for | read-heavy, transactional | write-heavy |

**Write-ahead log (WAL)**: before changing data, append the change to a log on
disk. After a crash, replay the log. It's also what replication and CDC read.

## Multi-region designs

| Pattern | How | Trade-off |
|---|---|---|
| Active-passive | one region serves; another on standby with replicated data | simple; failover takes time; standby is idle cost |
| Active-active (read local, write home) | each user has a "home" region for writes | low read latency; writes may cross regions |
| Active-active (multi-leader) | write anywhere | needs conflict resolution |
| Globally consistent DB | Spanner / CockroachDB | strong consistency everywhere; higher write latency |

## Remember

- Use etcd/ZooKeeper (Raft/ZAB, majority quorum) for leaders, locks and config.
- Locks need expiry + fencing tokens — or avoid them with conditional writes.
- Bloom filter, HyperLogLog, Count-Min: approximate answers with tiny memory.
- Geohash/quadtree/H3 for location; LSM for write-heavy, B-tree for read-heavy.
