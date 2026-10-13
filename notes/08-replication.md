# 08 · Replication — keeping copies of data

**Replication** = keeping the same data on several machines.

Why:
1. **Availability** — one machine dies, another has the data.
2. **Read scaling** — spread reads across copies.
3. **Latency** — a copy near users in each region.

The hard part: keeping copies **in sync** when data changes.

## 1. Single-leader (primary–replica) — the most common

```
writes ──► Leader ──(replication log)──► Follower 1 ──► reads
                                   └───► Follower 2 ──► reads
```

- All writes go to the **leader**; followers copy its changes; reads can go anywhere.
- Used by: Postgres, MySQL, MongoDB (replica sets), Redis.

### Sync vs async replication

| | Synchronous | Asynchronous |
|---|---|---|
| Leader waits for follower? | yes | no |
| Data loss if leader dies | none | the last few writes may be lost |
| Write latency | higher; a slow follower slows everything | low |
| Common | "semi-sync": wait for **one** follower | the default in many setups |

### Replication lag and its surprises

Async followers are a little **behind**. That causes classic bugs:

| Problem | Example | Fix |
|---|---|---|
| **Read-your-own-writes** | you update your profile, refresh, see the old one (read hit a lagging follower) | read your own data from the leader for a short time after writing; or track the write position |
| **Monotonic reads** | refresh twice, data goes *backwards* (two different followers) | pin a user to one follower |
| **Consistent prefix** | see an answer before the question | keep related writes in the same partition / order |

### Failover

The leader dies → promote a follower:
1. Detect failure (missed heartbeats / timeout).
2. Choose the most up-to-date follower (often via consensus — chapter 15).
3. Point writes at it.

Risks: losing un-replicated writes (async), and **split brain** (two nodes think
they're leader → conflicting writes). Prevent with consensus/quorum and **fencing**.

## 2. Multi-leader

Several leaders accept writes (e.g. one per region) and sync with each other.

- ✅ Write locally in each region (low latency); survives a region outage.
- ❌ **Write conflicts**: two regions change the same record at once.

Conflict resolution:
- **Last-write-wins (LWW)** by timestamp — simple, but silently drops a write (and clocks drift).
- Merge values, or ask the user.
- **CRDTs** — data types that merge automatically (counters, sets, collaborative text).

Used for: multi-region apps, offline-first apps (each phone is a "leader"), collaborative editing.

## 3. Leaderless (Dynamo-style)

Any node accepts reads and writes. Client (or a coordinator) sends each write to
**several** replicas and reads from several.

Used by: Cassandra, ScyllaDB, Riak, DynamoDB's original design.

### Quorums: N, W, R

- **N** = number of replicas for each item (e.g. 3)
- **W** = replicas that must confirm a write
- **R** = replicas asked on a read

If **W + R > N**, every read overlaps at least one replica with the latest write.

| Setting (N=3) | Effect |
|---|---|
| W=2, R=2 | balanced, the classic "quorum" |
| W=3, R=1 | fast reads, writes fail if any replica is down |
| W=1, R=1 | fastest, weakest — may read stale data |

Repair mechanisms bring stale replicas up to date: **read repair** (fix on read),
**anti-entropy** with **Merkle trees** (background comparison), **hinted handoff**
(a neighbour holds writes for a down node).

## Choosing

| Need | Use |
|---|---|
| Most apps, strong consistency for writes | single-leader |
| Writes in many regions / offline clients | multi-leader (+ conflict strategy) |
| Huge write volume, always writable, tunable consistency | leaderless with quorums |

## Remember

- Single-leader is the default; know sync vs async and replication lag bugs.
- Failover risks: lost writes and split brain.
- Leaderless uses quorums: W + R > N for overlapping reads/writes.
