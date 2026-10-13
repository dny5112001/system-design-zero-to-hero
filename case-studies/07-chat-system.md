# Case study 07 · Chat system (WhatsApp / Messenger / Slack)

**Difficulty:** ⭐⭐⭐ medium–hard. Tests: real-time connections, message ordering, delivery guarantees, storage at scale, presence.

## 1. Requirements

**Functional**
- 1-to-1 chat and **group chat** (up to ~500 members).
- Delivery states: **sent ✓, delivered ✓✓, read** (blue ticks).
- **Offline users** get messages when they come back (+ push notification).
- Online/last-seen **presence**.
- Media (images, video) — via object storage.
- History synced across a user's devices.

**Non-functional**
- 500M DAU; low latency (< ~200 ms delivery when both online).
- **Never lose a message**; correct **order within a conversation**.
- Highly available. (End-to-end encryption — mention.)

## 2. Estimates

```
500M DAU × 40 messages/day = 20B messages/day ≈ 230k messages/s (peak ~500k+)
Message ~ 100 bytes text + metadata ≈ 200 B → 4 TB/day → ~1.5 PB/year (before replication)
Concurrent connections: maybe 100–200M open WebSockets → ~50k–100k per gateway server → thousands of servers
```

## 3. Connections: how messages reach the phone

Clients keep a **persistent WebSocket** (or MQTT, as WhatsApp historically used
a custom XMPP variant) to a **chat gateway** server. Push in both directions.

Because connections are **stateful**, we need to know **which gateway** each user is on:

```
Session registry (Redis): user_id → gateway_server_id (+ device)
```

## 4. High-level design

```
Sender ══WS══► Gateway A ─► Chat service ─► 1) assign message ID & seq (per conversation)
                                           2) store in Message DB (durable)  ✔ "sent" to sender
                                           3) look up recipient in session registry
                                     ┌─────┴───────────────────────┐
                              online │                              │ offline
                                     ▼                              ▼
                      route to Gateway B (pub/sub or direct RPC)   Push notification service (APNs/FCM)
                                     ║                              │
Recipient ◄════════WS══════════ Gateway B                         (fetches on reconnect)
     └─ ack "delivered" ─► Chat service ─► update status ─► notify sender ✓✓
```

## 5. Deep dives

### Message storage
Huge write volume, access pattern = "messages of conversation X, newest first, paginated"
→ **wide-column store** (Cassandra/ScyllaDB/HBase — Discord moved to ScyllaDB):

```
messages: PRIMARY KEY ((conversation_id), message_id)   -- message_id time-ordered (Snowflake), clustering DESC
```

Partition by conversation keeps a chat together; very large/old chats → bucket
partitions by time (`(conversation_id, month)`) to avoid huge partitions.

Inbox / conversation list per user: `user_conversations(user_id, last_message_at, conversation_id, unread_count)`.

### Ordering
Assign a **per-conversation sequence number** at the chat service (single writer
per conversation via partitioning, or Snowflake ID per conversation shard).
Clients sort by it and detect gaps ("missing 1042 → fetch it"). Don't rely on
device clocks.

### Delivery guarantees
- Client gives each message a **client message ID** (idempotency) → retries don't create duplicates.
- Server acks "stored" → **sent ✓**.
- Recipient device acks receipt → **delivered ✓✓**; read event → **read**.
- On reconnect, the client asks "give me everything after seq N" for each conversation → nothing lost even if a push was missed.

### Group chat
Fan-out the message to each member's online gateways (small groups: on write).
Store the message **once** (per conversation), not per member. For very large
groups/channels (Slack, Telegram channels) → members pull instead.

### Presence
Heartbeats over the WebSocket; store `user → online/last_seen` in Redis with TTL.
Publishing every status change to every contact is expensive → only to contacts
with the chat open / subscribed, batched, and with some delay tolerance.

### Media
Upload to object storage (pre-signed URL); the message carries the media URL +
thumbnail; downloads via CDN. With E2E encryption, media is encrypted on the device first.

### Multi-device
Each device has its own connection and its own delivery state; sync via "fetch
since seq N".

### End-to-end encryption (mention)
Signal protocol: the server only relays encrypted blobs and can't read content.
Implications: server-side search/moderation isn't possible; keys live on devices.

### Gateway failure
Clients reconnect (with backoff + jitter, to avoid a thundering herd) to another
gateway; registry updated; client resyncs from last seq.

## 6. Wrap-up

- Stateful gateways + session registry; durable store before ack; per-conversation sequence; client-side idempotency; resync on reconnect.
- Monitor: delivery latency p99, undelivered queue size, connection counts per gateway.

## Follow-ups

- "How do you scale to 100M concurrent connections?" (Lightweight event-loop gateways ~50–100k+ connections each, horizontal, sticky by user via registry.)
- "Message search?" (Server-side index if not E2E; on-device search if E2E.)
- "Typing indicators?" (Ephemeral, not stored, best-effort via the same connection.)
