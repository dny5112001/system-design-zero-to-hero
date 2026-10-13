# Case study 05 · Notification system (push, SMS, email)

**Difficulty:** ⭐⭐ medium. Tests: queues, fan-out, third-party integration, retries, idempotency.

## 1. Requirements

**Functional**
- Send notifications via **mobile push** (iOS/Android), **SMS**, **email** (maybe in-app).
- Triggered by other services ("order shipped") or scheduled/bulk campaigns ("sale starts!").
- Respect **user preferences** (opt-outs per channel/type), quiet hours.
- Templates with variables and multiple languages.

**Non-functional**
- Scale: e.g. 10M push + 1M SMS + 5M email per day; campaign bursts to millions in minutes.
- **No lost notifications**, **no duplicates** (or very rare) — a double OTP or double "payment received" is bad.
- Soft real-time: transactional ones (OTP) within seconds; marketing can be slower.
- Third-party providers will be slow/fail sometimes.

## 2. Estimates

```
~16M notifications/day ≈ 200/s average; campaigns spike to 10–50k/s
Notification log: 16M × 1 KB ≈ 16 GB/day
```

Moderate average, very **bursty** → queues are essential.

## 3. API

```
POST /v1/notifications
{ "user_id": 42, "type": "order_shipped", "channels": ["push","email"],
  "data": { "order_id": "A123" }, "idempotency_key": "order-A123-shipped", "priority": "high" }
→ 202 Accepted { notification_id }
```

## 4. High-level design

```
services ─► Notification API ─► validate, dedupe (idempotency key), check prefs & rate limits
                                       │
                                       ▼
                         Kafka / queues per channel & priority
                   ┌──────────────┬──────────────┬──────────────┐
                   ▼              ▼              ▼
              Push workers    SMS workers    Email workers  ── render template
                   │              │              │
                 APNs / FCM    Twilio / SNS    SES / SendGrid   (third-party providers)
                   │
        status / receipts ─► notification log DB ─► analytics
  retries ─► retry queue (backoff) ─► DLQ after N tries
```

Supporting stores:
- **User preferences & device tokens** (SQL / KV, cached in Redis).
- **Templates** (DB + cache).
- **Notification log** (Cassandra/DynamoDB: status per notification for auditing and dedupe).

## 5. Deep dives

### Reliability: never lose one
- API writes the request durably (DB/Kafka) **before** returning 202.
- Workers commit/ack a message only **after** the provider accepted it.
- Transient errors → retry with exponential backoff + jitter; after N → **DLQ** + alert.

### No duplicates
- **Idempotency key** from the caller → dedupe at the API (unique key table with TTL).
- At-least-once queues mean workers may see a message twice → check the
  notification log (`status = sent`?) before sending; providers' own idempotency where available.

### Priorities
Separate queues: **high** (OTP, security alerts, transactional) vs **low**
(marketing). Dedicated workers for high so a campaign never delays an OTP.

### Bulk campaigns (fan-out)
"Notify 20M users": a campaign job reads the audience in **chunks**, enqueues
batches, respects per-provider throughput limits (APNs/FCM/SMS limits) and
**user-level rate limits** (don't spam: max N marketing pushes/day).

### Third-party failures
- Circuit breaker per provider; **fallback provider** for SMS/email.
- Respect provider rate limits (token bucket per provider).
- Handle invalid device tokens (remove them when APNs/FCM says unregistered).

### Preferences and compliance
Check opt-outs and quiet hours (user's time zone) before sending; unsubscribe
links in email; regulations (e.g. DND/TRAI rules for SMS in India, CAN-SPAM, GDPR).

### Tracking
Delivery receipts, opens, clicks → events → analytics. Store status:
`queued → sent → delivered → opened / failed`.

## 6. Wrap-up

- Bottlenecks: provider throughput (scale workers, multiple providers), campaign bursts (queues absorb).
- Failures: retries, DLQ, provider fallback, idempotency.
- Monitor: queue lag per channel, send success rate per provider, p99 time-to-deliver for OTPs.

## Follow-ups

- "Exactly-once OTP?" (Idempotency key + status check; short validity window anyway.)
- "In-app notifications / notification inbox?" (Store per-user inbox in a DB; push via WebSocket/SSE if online.)
- "Scheduled notifications?" (A scheduler — case study 16.)
