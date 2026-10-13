# System Design: Zero to Hero (interview edition)

Plain-English notes to understand system design from the very basics to advanced
distributed systems — and to **perform well in system design interviews**.

- **`notes/`** — the concepts, each with trade-offs and "remember" points.
- **`case-studies/`** — 16 classic interview problems, each solved with the same 7-step method you'll use in the room.

## Start here

1. Read **[01 · The interview and the method](notes/01-the-interview-and-the-method.md)** — the 7 steps you'll use every time.
2. Learn the building blocks (notes 02–16).
3. Practise the case studies **out loud with a timer** — try each yourself for 30–40 minutes **before** reading the solution.
4. Night before: **[17 · Cheat sheet](notes/17-cheat-sheet.md)** + **[18 · Concept Q&A](notes/18-concept-interview-questions.md)**.

## Concepts (`notes/`)

| # | Chapter | Level |
|---|---|---|
| 01 | [The interview and the method](notes/01-the-interview-and-the-method.md) | 🟢 start here |
| 02 | [Numbers and back-of-the-envelope estimation](notes/02-numbers-and-estimation.md) | 🟢 |
| 03 | [Scalability, performance, availability](notes/03-scalability-basics.md) | 🟢 |
| 04 | [Networking and protocols](notes/04-networking-and-protocols.md) | 🟢 |
| 05 | [Load balancers, proxies, API gateways, CDNs](notes/05-load-balancers-proxies-cdn.md) | 🟢 |
| 06 | [Caching](notes/06-caching.md) | 🟡 |
| 07 | [Databases: SQL vs NoSQL, indexes](notes/07-databases.md) | 🟡 |
| 08 | [Replication](notes/08-replication.md) | 🟡 |
| 09 | [Sharding and consistent hashing](notes/09-sharding-and-consistent-hashing.md) | 🟡 |
| 10 | [CAP, PACELC, consistency models](notes/10-cap-and-consistency.md) | 🟡 |
| 11 | [Queues, pub/sub, Kafka, outbox](notes/11-queues-and-streams.md) | 🟡 |
| 12 | [Storage, search, data pipelines](notes/12-storage-search-and-data.md) | 🟡 |
| 13 | [API design](notes/13-api-design.md) | 🟡 |
| 14 | [Microservices and resilience patterns](notes/14-microservices-and-resilience.md) | 🔴 |
| 15 | [Advanced distributed systems toolkit](notes/15-advanced-distributed-systems.md) | 🔴 |
| 16 | [Reliability, observability, security](notes/16-reliability-observability-security.md) | 🔴 |
| 17 | [Night-before cheat sheet](notes/17-cheat-sheet.md) | ⭐ |
| 18 | [40 concept questions with short answers](notes/18-concept-interview-questions.md) | ⭐ |

🟢 basics · 🟡 core · 🔴 advanced (senior/staff)

## Case studies (`case-studies/`)

| # | Problem | Difficulty | Key ideas |
|---|---|---|---|
| 01 | [URL shortener](case-studies/01-url-shortener.md) | ⭐ | base62, ID ranges, caching, 301 vs 302 |
| 02 | [Unique ID generator](case-studies/02-unique-id-generator.md) | ⭐ | Snowflake, UUIDv7, clock issues |
| 03 | [Rate limiter](case-studies/03-rate-limiter.md) | ⭐⭐ | token bucket, sliding window, Redis + Lua |
| 04 | [Distributed key-value store](case-studies/04-key-value-store.md) | ⭐⭐⭐ | consistent hashing, quorums, vector clocks, LSM |
| 05 | [Notification system](case-studies/05-notification-system.md) | ⭐⭐ | queues, priorities, idempotency, providers |
| 06 | [News feed](case-studies/06-news-feed.md) | ⭐⭐⭐ | fan-out on write vs read, celebrities, hybrid |
| 07 | [Chat system](case-studies/07-chat-system.md) | ⭐⭐⭐ | WebSockets, session registry, ordering, receipts |
| 08 | [Typeahead / autocomplete](case-studies/08-typeahead-autocomplete.md) | ⭐⭐ | trie with top-K, offline build, CDN caching |
| 09 | [Web crawler](case-studies/09-web-crawler.md) | ⭐⭐ | frontier, politeness, Bloom filters, dedupe |
| 10 | [Video streaming](case-studies/10-video-streaming.md) | ⭐⭐⭐ | transcoding DAG, HLS/DASH, ABR, CDN |
| 11 | [File storage and sync](case-studies/11-file-storage-sync.md) | ⭐⭐⭐ | chunking, content hashes, delta sync, conflicts |
| 12 | [Ride-hailing](case-studies/12-ride-hailing.md) | ⭐⭐⭐ | geo index (H3/geohash), matching, state machine |
| 13 | [Ticket booking](case-studies/13-ticket-booking.md) | ⭐⭐⭐ | no double booking, holds, waiting room |
| 14 | [Payment system](case-studies/14-payment-system.md) | ⭐⭐⭐⭐ | idempotency, ledger, reconciliation, PSPs |
| 15 | [Leaderboard and top-K](case-studies/15-leaderboard-top-k.md) | ⭐⭐ | Redis sorted sets, Count-Min Sketch |
| 16 | [Distributed job scheduler](case-studies/16-distributed-job-scheduler.md) | ⭐⭐⭐⭐ | leases, at-least-once, fencing, time buckets |

## A 4-week study plan

| Week | Do |
|---|---|
| 1 | Notes 01–07. Case studies 01, 02, 03. Memorise the numbers in 02. |
| 2 | Notes 08–13. Case studies 04, 05, 06, 08. |
| 3 | Notes 14–16. Case studies 07, 09, 10, 11, 15. |
| 4 | Case studies 12, 13, 14, 16. Two timed mock interviews a week (a friend, or record yourself). Review 17 and 18. |

## How to practise a case study

1. Set a 45-minute timer. Talk out loud as if the interviewer is there.
2. Follow the 7 steps; draw on paper / Excalidraw.
3. Then read the solution and note: **what did I miss?** (requirements, a number, a failure case, a trade-off).
4. Redo the same problem a week later in 30 minutes.

## Related notes

- [terraform-zero-to-hero](https://github.com/dny5112001/terraform-zero-to-hero)
- [kubernetes-zero-to-hero](https://github.com/dny5112001/kubernetes-zero-to-hero)
- [rust-zero-to-hero](https://github.com/dny5112001/rust-zero-to-hero)
- [k8s-terraform-lab](https://github.com/dny5112001/k8s-terraform-lab)
