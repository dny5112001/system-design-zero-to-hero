# 14 · Microservices, communication, and staying up when things fail

## Monolith vs microservices

| | Monolith | Microservices |
|---|---|---|
| What | one app, one deploy | many small services, each owning one business area and its own data |
| ✅ | simple to build, test, debug, deploy; fast in-process calls; easy transactions | teams deploy independently; scale parts separately; failures isolated; different tech per service |
| ❌ | big teams step on each other; one bug can take all down; scale all-or-nothing | network calls, partial failures, distributed data, eventual consistency, much more ops (monitoring, tracing, CI/CD) |

**Honest advice (and a good interview line):** start with a well-structured
**modular monolith**; split out services when team size or scaling needs
clearly demand it. Split along **business boundaries** (orders, payments,
catalogue), not technical layers. **Database per service** — other services
access its data only through its API or events.

## How services talk

| Style | How | ✅ | ❌ |
|---|---|---|---|
| **Synchronous** | REST/gRPC request-response | simple, immediate answer | caller waits; failures cascade; tight coupling |
| **Asynchronous** | events/messages via a queue or Kafka | decoupled, absorbs spikes, resilient | eventual consistency, harder to trace |

Rule of thumb: sync for queries that need an answer now; async for "this
happened" notifications and long work.

### Service discovery

Instances come and go — how does A find B? A registry (Consul, etcd, Eureka) or
the platform does it (Kubernetes Services + DNS). Clients or a load balancer pick a healthy instance.

### Service mesh

A sidecar proxy (Envoy) next to each service, managed centrally (Istio, Linkerd):
mTLS between services, retries, timeouts, traffic splitting, metrics — without
changing app code. Powerful, but adds complexity.

## Distributed transactions

An order needs: reserve stock (Inventory), charge card (Payments), create
shipment (Shipping) — three services, three databases. No single transaction.

### Two-phase commit (2PC)
A coordinator asks everyone "can you commit?" (prepare), then "commit!".
✅ atomic. ❌ blocking: if the coordinator dies mid-way, participants hold locks;
slow; rarely used across microservices.

### Saga — the usual answer
A sequence of **local** transactions; if a step fails, run **compensating**
actions to undo the earlier ones.

```
Reserve stock ✔ → Charge card ✘ → Release stock (compensation) → Order failed
```

- **Choreography**: each service reacts to events ("StockReserved" → Payments charges). Simple for few steps; hard to see the whole flow.
- **Orchestration**: a central orchestrator (e.g. Temporal, AWS Step Functions) tells each service what to do next. Clear flow, easier to monitor.
- Compensations must be idempotent; some steps can't be undone (email sent) → do them last.

Pair with the **outbox pattern** (chapter 11) so state changes and events never disagree.

## Resilience patterns — expect failure

At scale, **something is always broken**. Design so one failure doesn't spread.

### Timeouts
**Every** network call needs a timeout. Without one, a slow dependency ties up
all your threads and you go down too.

### Retries with exponential backoff + jitter
Retry transient errors, waiting longer each time (100ms, 200ms, 400ms...) plus
randomness (**jitter**) so thousands of clients don't retry in sync.
Only retry **idempotent** operations (or use idempotency keys). Cap the number
of retries; use **retry budgets** to avoid "retry storms" multiplying load.

### Circuit breaker
Like an electrical fuse. After many failures calling B, **stop calling** it for
a while (fail fast / return a fallback), then let a few test requests through
(half-open). Protects B while it recovers and keeps A responsive.

```
CLOSED (normal) ──too many failures──► OPEN (fail fast) ──after cool-down──► HALF-OPEN (try a few)
       ▲                                                                         │
       └──────────────────────── successes ◄────────────────────────────────────┘
```

### Bulkhead
Separate pools (threads, connections) per dependency — like watertight
compartments in a ship. A slow recommendations service can't exhaust the
threads needed for checkout.

### Fallbacks and graceful degradation
If recommendations fail, show "popular items". If the feed ranker is slow, show
a chronological feed. Better partial than nothing.

### Load shedding and rate limiting
When overloaded, reject low-priority work early (429/503) to keep the important
work fast.

### Idempotency everywhere
Retries + at-least-once messaging mean duplicates **will** happen.

## Remember

- Modular monolith first; microservices along business lines with their own data.
- Cross-service consistency = sagas (+ outbox), not 2PC.
- Timeouts, retries with backoff+jitter, circuit breakers, bulkheads, fallbacks, load shedding.
