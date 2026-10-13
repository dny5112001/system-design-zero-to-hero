# 11 · Message queues, pub/sub and event streaming

## Why queues?

Some work doesn't need to happen **while the user waits**: sending an email,
resizing a photo, transcoding a video, updating search indexes, recalculating
recommendations.

A **message queue** is a to-do list between services: the producer drops a
message and moves on; workers pick messages up and process them.

```
API ──► [ queue: msg msg msg ] ──► worker 1
                                └─► worker 2
```

Benefits:
- **Faster responses** — return "accepted" immediately.
- **Decoupling** — producer doesn't need to know who consumes or if they're up.
- **Smoothing spikes (load levelling)** — a burst of 100k jobs waits in the queue; workers drain it at their pace.
- **Reliability** — if a worker crashes, the message is redelivered.
- **Independent scaling** — add workers when the queue grows.

## Queue vs pub/sub vs log

| Model | Each message goes to | Examples |
|---|---|---|
| **Queue (point-to-point)** | **one** consumer (work is shared out) | SQS, RabbitMQ, Redis lists |
| **Pub/sub** | **every** subscriber gets a copy | SNS, Google Pub/Sub, Redis pub/sub |
| **Log / stream** | messages kept in order for days; each **consumer group** reads at its own position, and can re-read | **Kafka**, Kinesis, Pulsar, Redpanda |

## Kafka in plain words

- A **topic** is a named stream ("orders").
- Topics are split into **partitions** for parallelism; messages with the same
  **key** go to the same partition → **ordered per key** (e.g. per order_id).
- Messages are **kept** (e.g. 7 days) whether read or not. Consumers track their
  **offset** (position). New consumers can replay history.
- A **consumer group** shares partitions: each partition is read by one member →
  max parallelism = number of partitions.
- Replicated across brokers for durability.

Use Kafka for: event-driven architectures, activity tracking, logs/metrics
pipelines, change data capture (CDC), stream processing (with Flink / Kafka Streams).
Use a simple queue (SQS/RabbitMQ) for: background jobs/tasks.

## Delivery guarantees

| Guarantee | Meaning | Reality |
|---|---|---|
| At-most-once | may lose messages, never duplicates | rarely what you want |
| **At-least-once** | never lost, **may be duplicated** | the common default |
| Exactly-once | neither | hard; possible within a system (Kafka transactions), not across arbitrary systems |

**The practical answer:** at-least-once delivery + **idempotent consumers** =
effectively exactly-once.

### Idempotent consumer

Processing the same message twice has the same effect as once:
- keep a table of processed message IDs (`INSERT ... ON CONFLICT DO NOTHING`), or
- make the operation naturally idempotent (`SET status = 'paid'` rather than `balance = balance - 10`), or
- use a unique constraint on a business key.

## Important patterns and problems

### Retries and the dead-letter queue (DLQ)
A message keeps failing (bad data, a bug)? After N retries (with backoff), move it
to a **DLQ** for inspection, so it doesn't block everything else.

### Ordering
Queues generally don't guarantee global order. If order matters, order **per key**
(Kafka partition key, SQS FIFO message group) — never globally (too slow).

### Backpressure
Consumers can't keep up → queue grows → memory/lag. Watch **consumer lag**;
autoscale consumers; rate-limit producers; shed low-priority work.

### The dual-write problem → Transactional outbox
"Save the order to the DB **and** publish OrderCreated to Kafka." If one succeeds
and the other fails, they disagree.

**Outbox pattern**: in the **same DB transaction**, insert the order **and** a row
in an `outbox` table. A separate relay process (or CDC, e.g. Debezium) reads the
outbox and publishes to Kafka, retrying until done. Consumers are idempotent.

### Change Data Capture (CDC)
Read the database's change log (Postgres WAL, MySQL binlog) and stream every
change as an event — to update caches, search indexes, warehouses, without the
app doing dual writes.

### Fan-out
One event → many consumers (notifications, analytics, search) via pub/sub or a
stream with several consumer groups.

## Event-driven architecture — the trade-off

✅ Loose coupling, scalability, easy to add new consumers, audit trail.
❌ Harder to follow the flow and debug, eventual consistency, duplicates and
ordering to handle, needs good tracing.

## Remember

- Queues for background jobs and spikes; pub/sub for fan-out; Kafka for durable, replayable, ordered-per-key streams.
- Assume at-least-once → make consumers idempotent.
- Outbox/CDC solves "save to DB and publish an event" reliably. Use DLQs and watch lag.
