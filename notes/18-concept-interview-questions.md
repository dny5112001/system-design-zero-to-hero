# 18 · Concept questions — short answers

Interviewers often fire quick "what's the difference / when would you use"
questions. Practise answering each in 2–4 sentences.

### Fundamentals

**1. Vertical vs horizontal scaling?** Bigger machine vs more machines. Vertical is simple but has a ceiling and is a single point of failure; horizontal scales further and survives failures but needs stateless services and distributed data.

**2. Why make services stateless?** Any instance can serve any request, so you can add/remove/replace instances freely behind a load balancer. State goes to shared stores (DB, Redis, object storage).

**3. Latency vs throughput?** Time for one request vs requests per second. Raising utilisation increases throughput until queues form, then latency explodes.

**4. Why p99 rather than average latency?** Averages hide the slow tail; with many service calls per page, most users hit some service's p99.

**5. What's a single point of failure? How do you remove one?** A component whose failure takes the system down. Add redundancy (multiple instances/AZs), failover, and remove hard dependencies.

### Networking

**6. TCP vs UDP?** TCP: reliable, ordered, connection-based (web, DBs). UDP: fast, no guarantees (video calls, games, DNS, QUIC).

**7. L4 vs L7 load balancer?** L4 routes by IP/port, fast and protocol-agnostic. L7 understands HTTP, can route by path/header, terminate TLS, do auth.

**8. WebSockets vs SSE vs long polling?** WebSockets: two-way persistent; SSE: server-to-client stream over HTTP; long polling: held-open requests as a fallback. Chat → WebSockets; notifications/streams → SSE.

**9. What does a CDN do?** Caches content at edges near users: lower latency, less origin load and bandwidth, DDoS absorption. Use versioned URLs to avoid invalidation pain.

**10. REST vs gRPC vs GraphQL?** REST for public, cacheable APIs; gRPC for fast typed internal calls and streaming; GraphQL when clients need to choose fields from a complex graph.

### Data

**11. SQL vs NoSQL — how do you choose?** SQL for relational data, transactions and flexible queries (the default). NoSQL when the access pattern is simple (by key) and scale/write volume or schema flexibility demands it, accepting weaker consistency or fewer query options.

**12. What is ACID?** Atomicity, Consistency, Isolation, Durability — transactions are all-or-nothing, keep rules, don't interfere, and survive crashes.

**13. What does an index cost?** Faster reads for matching queries; slower writes, extra storage. Composite index column order matters.

**14. Replication vs sharding?** Replication copies all data to several nodes (availability, read scaling); sharding splits data across nodes (capacity, write scaling). Usually both.

**15. How do you pick a shard key?** High cardinality, even traffic distribution, and data queried together stays together; avoid monotonically increasing keys for range sharding (hot spot).

**16. What is consistent hashing?** Map nodes and keys onto a ring; a key belongs to the next node clockwise. Adding/removing a node moves only ~1/N keys; virtual nodes balance load.

**17. Explain CAP.** During a network partition you must choose between consistency (reject/err) and availability (answer, maybe stale). Choose per feature: CP for money, AP for social features.

**18. What is eventual consistency? Give a problem it causes.** Replicas converge if writes stop. You might not see your own update immediately (fix with read-your-writes: read from the leader after writing).

**19. Quorum reads/writes?** With N replicas, require W acks on write and R on read; W + R > N guarantees overlap with the latest write.

**20. Leader-follower failover risks?** Losing un-replicated writes (async) and split brain; mitigate with consensus-based election and fencing.

### Caching

**21. Cache-aside vs write-through vs write-behind?** Cache-aside: app loads on miss, deletes on write (most common). Write-through: write cache and DB together (fresh, slower writes). Write-behind: write cache, flush later (fast, risk of loss).

**22. What is a cache stampede and how do you prevent it?** A hot key expires and many requests hit the DB at once. Use locking/single-flight, early refresh, jittered TTLs, serve-stale.

**23. How do you handle a hot key?** Local in-process caching, key replication with random suffixes, read replicas, rate limiting.

### Messaging and architecture

**24. Why use a message queue?** Decoupling, async processing, absorbing spikes, retries, independent scaling.

**25. Kafka vs RabbitMQ/SQS?** Kafka: durable, replayable log, ordered per partition, many consumer groups — event streaming. RabbitMQ/SQS: task queues where each message is consumed once and removed.

**26. Exactly-once delivery?** Practically: at-least-once delivery plus idempotent processing (dedupe by message ID / idempotent operations).

**27. What is the outbox pattern?** Write business data and an outbox event in one DB transaction; a relay/CDC publishes the event. Avoids the dual-write problem.

**28. Saga vs 2PC?** 2PC: atomic but blocking and fragile across services. Saga: chain of local transactions with compensations; eventually consistent; the standard for microservices.

**29. Circuit breaker?** Stops calling a failing dependency for a cool-down period, failing fast or using a fallback, then tests recovery. Prevents cascading failures.

**30. Monolith or microservices for a new product?** Usually a modular monolith first; split when team size, deploy independence or scaling needs justify the operational cost.

### Advanced

**31. How do you implement a distributed lock safely?** Use a lease with expiry (Redis SET NX PX or etcd), unique owner token, and fencing tokens checked by the resource. Prefer avoiding locks via conditional writes.

**32. Why not use timestamps to order events across machines?** Clocks drift. Use logical clocks (Lamport/vector), a single sequencer per key, or systems with bounded clock uncertainty (Spanner TrueTime).

**33. What is a Bloom filter?** A compact bit array answering "maybe present / definitely not". Saves expensive lookups for missing items; tolerates small false positives.

**34. How would you generate unique IDs across many servers?** Snowflake-style IDs (timestamp + machine ID + sequence), or UUIDv7/ULID; avoid a central counter bottleneck.

**35. How do you design for a region outage?** Data replicated to another region, defined RPO/RTO, DNS/global LB failover, stateless services deployable anywhere, regular failover drills.

**36. Idempotency key — how does it work?** Client sends a unique key per operation; server stores the key with the result and returns the stored result on retries; unique constraint handles concurrent duplicates.

**37. Rate limiting algorithms?** Token bucket (allows bursts), leaky bucket (smooth outflow), fixed window (simple, edge bursts), sliding window log/counter (accurate). Distributed: Redis with atomic Lua.

**38. How do you find nearby drivers/restaurants quickly?** Geospatial index: geohash or H3 cells (search cell + neighbours) or a quadtree; keep live locations in memory (Redis GEO).

**39. Push vs pull for a news feed?** Push (fan-out on write) precomputes feeds — fast reads, expensive for celebrities. Pull (fan-out on read) — cheap writes, slow reads. Hybrid: push for normal users, pull for celebrities.

**40. Blue-green vs canary?** Blue-green switches all traffic between two full environments (instant rollback). Canary shifts a small percentage first and watches metrics before going wider.
