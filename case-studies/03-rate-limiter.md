# Case study 03 · Rate limiter

**Difficulty:** ⭐⭐ easy–medium. Tests: algorithms, distributed counters, atomicity, where to place components.

## 1. Requirements

**Functional**
- Limit how many requests a **client** (user ID, API key, IP) can make in a time window, per rule: e.g. "100 requests/minute per user on `/search`", "5 logins/minute per IP".
- Over the limit → reject with **429** + `Retry-After` and rate-limit headers.

**Non-functional**
- Adds very little latency (< 1–2 ms).
- Works across **many API servers** (a shared view of counts).
- Highly available: if the limiter fails, **fail open** (allow) for most APIs — don't take the product down — but maybe fail closed for login/abuse-sensitive endpoints.
- Accuracy can be slightly approximate.

## 2. Where does it live?

| Place | ✅ | ❌ |
|---|---|---|
| Client side | — | can't trust clients |
| **API gateway / middleware** | central, before work is done | gateway must scale |
| Inside each service | knows business context | duplicated logic |

Usual answer: **at the API gateway** (or a shared middleware library) backed by **Redis**.

## 3. Algorithms (the core of the interview)

### Token bucket ⭐ (most common; AWS, Stripe)
A bucket holds up to **B** tokens and refills at **R** tokens/sec. Each request
takes one token; no token → reject.
- ✅ allows short **bursts** (up to B) while enforcing the average rate; memory: 2 numbers per client.
- Store `tokens` and `last_refill_time`; on each request, add `elapsed × R` (capped at B), then take one.

### Leaky bucket
Requests enter a queue that drains at a fixed rate. ✅ smooth output (good for
protecting a fragile backend). ❌ bursts wait or drop; old requests can block new ones.

### Fixed window counter
Count requests per window (`user:42:2026-10-12T10:05`); reset each minute.
✅ simplest. ❌ **edge burst**: 100 at 10:05:59 + 100 at 10:06:00 = 200 in 2 seconds.

### Sliding window log
Store each request's timestamp (Redis sorted set); count those in the last 60 s.
✅ exact. ❌ memory grows with traffic.

### Sliding window counter ⭐
Weighted mix of the current and previous fixed windows:
`count ≈ current + previous × (fraction of previous window still in range)`.
✅ cheap and smooth; ❌ approximate (good enough in practice — Cloudflare uses it).

## 4. Design

```
client ─► API gateway ─► [rate limit check] ─► Redis (counters/buckets, sharded)
                │                 ▲
                │           rules from config store (cached in gateway memory)
                ├─ allowed ─► backend services
                └─ rejected ─► 429 + headers  (and log/metrics)
```

- **Rules**: in a config service/YAML (`endpoint, key type, limit, window`), cached locally, hot-reloaded.
- **Keys**: `rl:{rule}:{client_id}` (+ window for fixed/sliding windows), with a TTL so idle keys disappear.

## 5. Deep dives

### Race conditions
Two servers read "99", both allow, both write 100 → limit exceeded.
Fix: make the check-and-update **atomic** — `INCR` + `EXPIRE` for fixed windows,
or a **Lua script** in Redis for token bucket / sliding window (runs atomically).

### Scaling Redis
Shard counters by key (Redis Cluster). Each request = 1 round trip (~0.5 ms in the same DC).
For extreme scale: keep **local in-memory counters** per gateway node and sync
to Redis periodically (approximate but fast), or give each node a share of the limit.

### Multi-region
Global exact limits need cross-region coordination (slow). Usual answer: limit
**per region** (each region gets a share), or replicate counters asynchronously
and accept small overshoot.

### Response headers
```
HTTP/1.1 429 Too Many Requests
Retry-After: 12
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1760263560
```

### Beyond simple limits
- Different tiers (free vs paid), per-endpoint costs (a search costs 5 tokens).
- Layered: per IP at the edge (DDoS/WAF), per user/API key at the gateway, per tenant in services.
- **Load shedding** under overload is related but different: drop low-priority traffic regardless of user.

## 6. Wrap-up

- Token bucket in Redis via an atomic Lua script, at the gateway, rules cached locally.
- Fail open on limiter outages (with alerts), except sensitive endpoints.
- Monitor: 429 rate per rule, Redis latency, top limited clients.

## Follow-ups

- "Why not just use fixed windows?" (Edge bursts.)
- "How do you rate-limit by IP when users share NAT?" (Combine keys, higher IP limits, prefer API keys/user IDs.)
- "What happens if Redis is down?" (Fail open + local fallback limits.)
