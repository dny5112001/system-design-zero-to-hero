# Case study 02 · Unique ID generator for distributed systems

**Difficulty:** ⭐ easy — often a sub-question inside bigger designs.

## 1. Requirements

- IDs must be **unique** across many servers and data centres.
- **64-bit** numbers (fit in a `BIGINT`), roughly **sortable by time**.
- Generate **10,000+ IDs/sec** per node, low latency, no single point of failure.

## 2. Options

| Option | ✅ | ❌ |
|---|---|---|
| **DB auto-increment** | simple | one DB = bottleneck + SPOF; doesn't work across shards |
| **Multi-master auto-increment** (each server steps by N: 1, 1+N, 1+2N...) | no coordination | hard to add servers; not time-ordered across servers |
| **UUID v4** (128-bit random) | zero coordination | 128 bits; not sortable; random inserts fragment B-tree indexes |
| **UUID v7 / ULID** | time-ordered, no coordination | 128 bits |
| **Ticket server** (central counter, e.g. Flickr) | simple, numeric | central dependency (run 2 with odd/even) |
| **Snowflake** | 64-bit, time-sorted, no coordination per ID | needs unique machine IDs and sane clocks |

## 3. Snowflake layout (Twitter)

```
 0 | 41 bits: milliseconds since custom epoch | 5 bits: datacenter | 5 bits: machine | 12 bits: sequence
sign        (~69 years)                           (32)                 (32)               (4,096 per ms)
```

- **Timestamp** first → IDs sort roughly by creation time (great for "latest first" and index locality).
- **Datacenter + machine** (10 bits = 1,024 generators) → no two generators collide.
- **Sequence** resets every millisecond → up to **4,096 IDs/ms per machine** (~4 million/s).
- 2⁴¹ ms ≈ **69.7 years** from your chosen epoch.

### Generation logic

```
now = current_ms()
if now == last_ms:
    sequence = (sequence + 1) & 4095
    if sequence == 0: wait until next millisecond
else:
    sequence = 0
last_ms = now
id = (now - EPOCH) << 22 | datacenter << 17 | machine << 12 | sequence
```

## 4. Deep dives

### Assigning machine IDs
Static config, or lease one from ZooKeeper/etcd at startup (released on
shutdown). Must never be duplicated — two machines with the same ID = duplicate IDs.

### Clock going backwards
NTP can adjust time backwards. If `now < last_ms`: refuse to generate (error) or
wait until the clock catches up; alert. Some implementations keep a logical
"last timestamp" and keep incrementing.

### Sortability caveat
Ordering is only **roughly** by time across machines (clock skew of a few ms).
Within one machine it's strict.

### Embedding or library?
Usually a **library in each service** (no network hop) — not a separate service —
as long as machine IDs are assigned safely.

## 5. When to pick what (say this)

- Need 64-bit, sortable, high throughput → **Snowflake**.
- Want zero coordination and 128 bits is fine → **UUIDv7/ULID**.
- Don't need sortable, just unique → UUIDv4.
- Low volume, one database → plain auto-increment is fine.

## Follow-ups

- "What if you need IDs that aren't guessable?" (Don't expose Snowflake IDs where enumeration matters, or add an opaque public ID.)
- "How many IDs before overflow?" (69 years from epoch; pick a recent epoch.)
