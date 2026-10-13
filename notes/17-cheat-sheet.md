# 17 · Night-before cheat sheet

## The method (45 min)

1. **Requirements** (5) — functional (3–4 core features, say what's out of scope) + non-functional (scale, latency, availability, consistency, durability).
2. **Estimates** (3) — QPS read/write, storage, bandwidth. 1 day ≈ 10⁵ s; 1M/day ≈ 12/s.
3. **API** (3) — 3–6 endpoints; cursor pagination; idempotency keys.
4. **Data model** (3) — entities, DB choice + why, partition key.
5. **High-level design** (10) — walk one write + one read end to end.
6. **Deep dives** (15) — 2–3 hardest parts; options → trade-offs → choice.
7. **Wrap-up** (5) — bottlenecks, SPOFs, failures, monitoring, next steps.

## Problem → tool

| If you need to... | Reach for |
|---|---|
| Handle more requests | stateless servers + load balancer + autoscaling |
| Make reads fast | cache (Redis, cache-aside + TTL), CDN, read replicas, denormalise/precompute |
| Handle huge writes | sharding, LSM stores (Cassandra/DynamoDB), batching, queues |
| Serve files/media | object storage (S3) + CDN + pre-signed URLs |
| Do slow work later | queue + workers (SQS/RabbitMQ); Kafka for streams/fan-out |
| Push updates to clients | WebSockets (two-way), SSE (one-way), long polling (fallback) |
| Search text | Elasticsearch/OpenSearch (inverted index) synced by CDC |
| Find things nearby | geohash / quadtree / H3, Redis GEO, PostGIS |
| Rank / leaderboard | Redis sorted sets |
| Count uniques / top-K approx. | HyperLogLog / Count-Min Sketch |
| Avoid duplicates / safe retries | idempotency keys, unique constraints, idempotent consumers |
| Keep DB and events in sync | transactional outbox / CDC |
| Multi-service transaction | saga (orchestrated) + compensations |
| Prevent double booking | row lock / conditional update / optimistic version / short-lived holds with expiry |
| Elect a leader / store config / locks | etcd / ZooKeeper (consensus) + fencing tokens |
| Generate unique IDs | Snowflake / UUIDv7 / ULID |
| Protect from abuse | rate limiting (token bucket) at the gateway, WAF |
| Survive failures | redundancy across AZs, timeouts, retries+backoff+jitter, circuit breakers, bulkheads |
| Global low latency | CDN, geo-DNS, regional deployments, data near users |

## Key trade-offs to say out loud

- **Consistency vs availability** (CAP) — and latency (PACELC). Decide per feature.
- **SQL vs NoSQL** — transactions/joins vs scale/flexible access by key.
- **Push vs pull** (feeds, notifications) — write cost vs read cost; hybrid for celebrities.
- **Cache freshness vs speed** — TTL length, invalidation strategy.
- **Sync vs async** — immediate answer vs resilience/decoupling.
- **Normalise vs denormalise** — clean writes vs fast reads.
- **Monolith vs microservices** — simplicity vs team/scale independence.
- **Exactly-once is expensive** — at-least-once + idempotency.
- **Accuracy vs cost** — approximate counts, sampling.

## Numbers

| | |
|---|---|
| RAM read | 100 ns |
| SSD random read | ~0.1 ms |
| Same-DC round trip | ~0.5 ms |
| Cross-continent | ~50–150 ms |
| 99.9% / 99.99% | 8.8 h / 53 min down per year |
| Redis | ~100k ops/s per node |
| SQL DB | ~10k simple QPS per node |
| Snowflake ID | 64 bits = 41 time + 10 machine + 12 sequence |

## Red flags to avoid

- Designing before asking requirements.
- No numbers; or 15 minutes of numbers.
- "Use microservices / Kafka / Cassandra" with no reason.
- A single database for everything at 1M QPS.
- Ignoring failure: no replicas, no timeouts, no retries.
- Strong consistency everywhere (slow) or nowhere (wrong for money).
- Not reacting to the interviewer's hints.
- Going silent.

## Strong phrases

- "Let me clarify the scope and the scale first."
- "This is read-heavy at about 40:1, so I'll optimise the read path."
- "The trade-off is..., and given our requirement X, I'd accept it."
- "This is a single point of failure; I'd add..."
- "For money I want strong consistency; for likes, eventual is fine."
- "To make retries safe, the client sends an idempotency key."
- "If I had more time, I'd look at..."
