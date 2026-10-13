# 10 · CAP, PACELC and consistency models

## The CAP theorem

In a distributed system, when the network **splits** (a **P**artition — some
nodes can't talk to others), you must choose between:

- **C**onsistency — every read sees the latest write (or gets an error), and
- **A**vailability — every request gets a (non-error) answer, even if it might be stale.

You can't have both **during a partition**. Partitions *will* happen in real
networks, so the real choice is **CP or AP**:

| | During a partition | Examples | Use for |
|---|---|---|---|
| **CP** | refuse some requests rather than return stale/conflicting data | Spanner, etcd, ZooKeeper, HBase, MongoDB (default settings), single-leader SQL | money, inventory, bookings, locks, config |
| **AP** | keep answering, maybe stale; reconcile later | Cassandra, DynamoDB (default reads), Riak, DNS, CDNs | feeds, likes, carts, presence, analytics |

> Common mistakes: "CA" isn't a real option for distributed systems (you can't
> opt out of partitions). And CAP's "consistency" means *linearizability*, not the
> "C" in ACID.

**Interview move:** decide per feature. "For the payment ledger, CP. For the like
counter, AP — a slightly stale count is fine."

## PACELC — the more useful version

**P**artition → choose **A** or **C**; **E**lse (normal operation) → choose
**L**atency or **C**onsistency.

Even with no failures, staying consistent costs time (waiting for replicas,
cross-region round trips). So:

| System | P: A or C | E: L or C |
|---|---|---|
| Cassandra, DynamoDB | A | L (fast, eventually consistent by default) |
| Spanner, CockroachDB | C | C (consistent, pays latency) |
| MongoDB | C | C (with majority reads/writes) |

## Consistency models — from strongest to weakest

| Model | Plain meaning | Cost |
|---|---|---|
| **Linearizable (strong)** | behaves like one copy; once a write is done, everyone sees it | highest latency; needs consensus/quorum |
| **Sequential** | everyone sees operations in the same order (not necessarily real-time) | high |
| **Causal** | if A caused B (a reply to a comment), everyone sees A before B; unrelated things can differ | moderate — a sweet spot for social apps |
| **Read-your-writes** | you always see your own changes | cheap to add |
| **Monotonic reads** | you never see time go backwards | cheap |
| **Eventual** | if writes stop, all copies **eventually** agree | lowest latency, highest availability |

Real systems mix them: eventual for the feed, read-your-writes for your own
profile, strong for the account balance.

## Tunable consistency

Many databases let you choose **per request**:
- Cassandra: `ONE`, `QUORUM`, `ALL` (and `LOCAL_QUORUM` for one region).
- DynamoDB: eventually consistent reads (default, cheaper) or strongly consistent reads.
- MongoDB: read/write concerns (`majority`).

## Consistency inside one database (ACID) vs across services

- In one SQL database: transactions + isolation levels (chapter 07).
- Across services/databases: no global transaction. Use **sagas**, the
  **outbox pattern** and **idempotency** (chapters 11, 14) → eventually consistent,
  but correct.

## How to talk about it in an interview

1. Say what **must** be strongly consistent and **why** (no double-booking, no negative balance).
2. Say what can be **eventually** consistent and the **user impact** ("a new like may take a second to show").
3. Pick technology/settings that match each.

## Remember

- CAP: during a partition, pick consistency or availability. Decide per feature.
- PACELC adds: even without failures, consistency costs latency.
- Strong for money and scarce resources; eventual for social, counters and analytics.
