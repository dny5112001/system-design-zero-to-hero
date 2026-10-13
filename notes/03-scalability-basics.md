# 03 · Scalability, performance and availability — the basics

## From one server to millions of users

Almost every design grows the same way. Know this story — it's the backbone
of most answers.

```
Stage 1: one server          app + database on one machine. Simple. Dies = everything dies.
Stage 2: split the database  app server(s) + a separate database server.
Stage 3: many app servers    load balancer in front; app servers are "stateless".
Stage 4: cache               Redis in front of the database for hot reads.
Stage 5: read replicas       copies of the database take the read load.
Stage 6: CDN                 images/videos/static files served from near the user.
Stage 7: async work          queues + workers for slow tasks (emails, video processing).
Stage 8: shard the database  split data across many databases when one can't hold it.
Stage 9: split services      separate services for separate jobs (microservices).
Stage 10: many regions       copies of everything near users worldwide.
```

**Don't jump to stage 10.** In interviews, start simple and move up when the
numbers say you must.

## Vertical vs horizontal scaling

| | Vertical ("scale up") | Horizontal ("scale out") |
|---|---|---|
| How | a bigger machine (more CPU, RAM) | more machines |
| Like | a bigger truck | more trucks |
| Pros | simple; no code changes | almost no upper limit; survives one machine dying |
| Cons | hard ceiling; expensive at the top; one machine = single point of failure | needs load balancing, stateless design, distributed data |

Real systems do both: scale up until it's awkward, then scale out.

## Stateless servers — the key to scaling out

A **stateless** server keeps **no user data in its own memory between requests**.
Any server can handle any request. So you can add/remove servers freely, and a
crash loses nothing.

Where does the state go? Sessions → Redis or a signed token (JWT); files →
object storage; data → databases.

> "Sticky sessions" (always sending a user to the same server) work, but make
> scaling and failures messier. Prefer stateless.

## Performance vs scalability

- **Performance problem**: slow even for **one** user → fix the code, query, index.
- **Scalability problem**: fast for one user, slow under **load** → add capacity, remove bottlenecks.

## Latency vs throughput

- **Latency** = how long **one** request takes (ms). Like the time for one car to drive the road.
- **Throughput** = how many requests per second the system handles. Like cars per hour on the road.

You usually want high throughput **with acceptable latency**. Pushing a system
to 100% use makes latency explode (queues build up). Aim for ~60–70% use at peak.

### Measure latency with percentiles, not averages

| Metric | Meaning |
|---|---|
| p50 (median) | half the requests are faster than this |
| p95 / p99 | 95% / 99% are faster — the slow tail |
| p99.9 | the really unlucky users |

An average hides the 1% of users waiting 5 seconds. If one page calls 100
services, **most** page loads hit at least one service's p99. Tail latency matters.

## Availability and reliability

- **Availability** = % of time the system works (chapter 02's "nines").
- **Reliability** = it keeps doing the right thing (no wrong answers, no data loss).
- **Durability** = data, once saved, isn't lost.

### SLA, SLO, SLI

| Term | Meaning | Example |
|---|---|---|
| **SLI** (indicator) | what you measure | % of requests that succeeded in < 300 ms |
| **SLO** (objective) | your internal target | 99.9% per month |
| **SLA** (agreement) | the promise to customers, with penalties | 99.5%, or credits back |
| **Error budget** | 100% − SLO | 0.1% ≈ 43 min/month you can "spend" on failures and risky releases |

### Single point of failure (SPOF)

Any part that, if it dies, takes everything down. Find them and add
**redundancy**: 2+ load balancers, 2+ app servers, database replicas with
failover, multiple availability zones.

## Common bottlenecks to look for

1. **The database** (almost always first) → cache, indexes, replicas, sharding.
2. **A single hot key/user/item** (a celebrity, a viral post) → special handling.
3. **Synchronous chains** (A calls B calls C...) → async, timeouts, parallel calls.
4. **Large payloads** → CDN, compression, pagination.
5. **Locks / contention** on one row → partition the counter, queue the writes.

## Remember

- Know the "one server → global" growth story and climb it step by step.
- Scale out with stateless servers; keep state in shared stores.
- Talk in percentiles (p99) and nines; find and remove single points of failure.
