# Case study 04 · Distributed key-value store (like Dynamo / Cassandra / Redis Cluster)

**Difficulty:** ⭐⭐⭐ medium–hard. Tests: partitioning, replication, consistency, failure handling — most of chapters 08–10 and 15 in one question.

## 1. Requirements

**Functional**: `put(key, value)`, `get(key)`, `delete(key)`. Values up to ~1 MB.

**Non-functional**
- Huge scale (billions of keys, 100k+ ops/s), add nodes without downtime.
- **Highly available** — always accept writes (AP), even during failures.
- **Tunable consistency** per request.
- Low latency (single-digit ms).

## 2. Building blocks (each answers a question)

| Question | Answer |
|---|---|
| Where does a key live? | **consistent hashing** with virtual nodes |
| How do we survive node failure? | **replicate** each key to N nodes (next N on the ring) |
| How do reads see writes? | **quorums**: W + R > N for strong-ish reads |
| What if replicas disagree? | **versioning** (vector clocks or timestamps) + conflict resolution |
| A replica was down during writes? | **hinted handoff**, **read repair**, **Merkle-tree anti-entropy** |
| How do nodes know who's alive? | **gossip** protocol + failure detection |
| How is data stored on disk fast? | **LSM tree**: commit log + memtable + SSTables + compaction, Bloom filters |

## 3. Design

```
client ─► any node (acts as coordinator for this request)
             │ hash(key) → position on ring → preference list [A, B, C]  (N = 3)
             ├──► A
             ├──► B     write succeeds when W of them ack
             └──► C     read returns when R of them answer (newest version wins / conflicts returned)
```

Every node is equal (**no leader**, no single point of failure). Clients can be
"partition-aware" and go straight to a replica.

## 4. Deep dives

### Write path on a node
1. Append to the **commit log** (durability).
2. Insert into the **memtable** (sorted, in memory).
3. When the memtable is full, flush to an immutable **SSTable** on disk.
4. **Compaction** merges SSTables, drops overwritten/deleted values (deletes are
   "tombstones" until compaction).

### Read path on a node
Check memtable → check SSTables newest-first, using a **Bloom filter** per
SSTable to skip files that can't contain the key → merge results.

### Consistency settings (N = 3)
- `W=1, R=1`: fastest, may read stale.
- `W=2, R=2`: quorum — overlap guarantees reading the latest acknowledged write (barring concurrent writes/sloppy quorum edge cases).
- `W=3`: writes fail if any replica is down.

### Conflicts
Two clients write the same key on different replicas during a partition.
- **Last-write-wins** by timestamp (Cassandra): simple; may silently lose a write.
- **Vector clocks** (Dynamo): detect concurrent versions and return both "siblings" to the client to merge (e.g. merge shopping carts).
- **CRDTs**: types that merge automatically (counters, sets).

### Failures
- **Temporary**: a node is down → writes go to the next healthy node with a
  **hint** ("this belongs to C"); delivered when C returns (**hinted handoff**,
  "sloppy quorum").
- **Permanent / drift**: replicas compare **Merkle trees** of key ranges and sync only differing ranges.
- **Detection**: gossip heartbeats; a node is suspected dead when several peers stop hearing from it.

### Adding a node
It takes over some virtual-node ranges; data streams to it from neighbours;
only ~1/N of data moves.

### Hot keys
Consistent hashing doesn't fix one extremely hot key: cache it, replicate reads
more widely, or split the key.

## 5. Trade-offs to state

- AP + tunable consistency, not strict ACID. No multi-key transactions (or limited "lightweight transactions" via Paxos, slow).
- Queries only by key (plus sorted clustering columns in wide-column variants).
- LSM: great writes, reads may touch several files; compaction costs I/O.

## 6. Wrap-up

Consistent hashing + N replicas + quorum R/W + versioning + hinted handoff +
Merkle repair + gossip + LSM storage. That sentence **is** Dynamo — be able to
explain each piece and why it's there.

## Follow-ups

- "Make it strongly consistent instead?" (Per-partition leader with Raft — like etcd/TiKV/CockroachDB — at the cost of availability during partitions and latency.)
- "How would you add TTLs?" (Store expiry per value; drop on read/compaction.)
- "What about a cache instead of a store?" (Redis Cluster: hash slots, primary/replica per slot, in-memory, async replication.)
