# 02 · Numbers every engineer should know + estimation

## Latency numbers (rough, but the ratios matter)

| Operation | Time | Feels like (if 1 ns = 1 second) |
|---|---|---|
| Read from CPU cache (L1) | ~1 ns | 1 second |
| Read from main memory (RAM) | ~100 ns | ~2 minutes |
| Read 1 MB sequentially from RAM | ~10 µs | ~3 hours |
| Read 4 KB randomly from SSD | ~20–100 µs | ~1 day |
| Round trip inside a data centre | ~0.5 ms | ~6 days |
| Read 1 MB sequentially from SSD | ~0.2–1 ms | ~1 week |
| Disk (HDD) seek | ~5–10 ms | ~3 months |
| Round trip across a continent | ~50–80 ms | ~2 years |
| Round trip India ↔ US | ~150–250 ms | ~6 years |

**The lessons:**
- **Memory is ~1000× faster than disk/network** → caching is powerful.
- **Network calls are expensive** → avoid chatty services; batch requests.
- **Crossing oceans is very slow** → put data near users (CDNs, regions).
- **Sequential beats random** → logs, Kafka and LSM-tree databases write sequentially for speed.

## Powers of two and units

| Power | Exact | Approx | Name |
|---|---|---|---|
| 2¹⁰ | 1,024 | 1 thousand | 1 KB |
| 2²⁰ | | 1 million | 1 MB |
| 2³⁰ | | 1 billion | 1 GB |
| 2⁴⁰ | | 1 trillion | 1 TB |
| 2⁵⁰ | | 1 quadrillion | 1 PB |

Handy sizes: a char ≈ 1 byte (ASCII), an int/long = 4/8 bytes, a UUID = 16 bytes,
a tweet-sized text ≈ 300 bytes, a typical photo ≈ 200 KB–2 MB, a minute of HD
video ≈ 50–100 MB (before compression variants).

## Time shortcuts

- **1 day ≈ 86,400 s ≈ 10⁵ s** (round up — easier maths).
- 1 month ≈ 2.5 million s; 1 year ≈ 3 × 10⁷ s.
- **1 million requests/day ≈ 12 requests/sec.** (Remember this one.)
- 1 billion/day ≈ 12,000/sec.

## Availability ("nines")

| Availability | Downtime per year | per month |
|---|---|---|
| 99% (two nines) | 3.65 days | ~7.3 hours |
| 99.9% (three) | 8.8 hours | ~44 min |
| 99.99% (four) | 53 min | ~4.4 min |
| 99.999% (five) | 5.3 min | ~26 s |

Two components that **both** must work (in series): multiply →
99.9% × 99.9% = **99.8%**. Two copies where **either** is enough (in parallel):
1 − (0.001 × 0.001) = **99.9999%**. That's why redundancy works.

## What one machine can roughly handle

Very rough, but useful for "do we need more than one?":

| Thing | Ballpark |
|---|---|
| Web/app server (simple requests) | 1k–10k+ requests/sec |
| SQL database (one good server) | ~5k–20k simple queries/sec; far less for complex ones |
| Redis / Memcached | ~100k+ operations/sec per node |
| Kafka | ~100k–1M+ messages/sec per cluster (depends on size) |
| Single server RAM | 64 GB – 1 TB+ |
| Single disk | several TB SSD; 10–20+ TB HDD |

## How to estimate (the recipe)

1. **Start from users**: DAU (daily active users).
2. **Actions per user per day** → requests per day.
3. **Divide by 10⁵** → average requests/sec (QPS).
4. **Peak ≈ 2–3× average** (or more for spiky apps).
5. **Size per item × items per day** → storage per day → × 365 × years.
6. Add **replication** (× 3 is typical).

### Worked example: a Twitter-like app

Assume 200 M DAU; each user reads the timeline 20 times/day and posts 0.5 times/day.

```
Writes: 200M × 0.5      = 100M posts/day  → 100M / 10⁵ ≈ 1,000 writes/sec   (peak ~3,000)
Reads:  200M × 20       = 4B reads/day    → 4B / 10⁵   ≈ 40,000 reads/sec  (peak ~100,000)
Read:write ratio        ≈ 40:1  → read-heavy → caching + precomputed feeds matter

Storage (text): 100M posts × 500 bytes (text + metadata) = 50 GB/day
                × 365 × 5 years ≈ 90 TB, × 3 replicas ≈ 270 TB
Media: say 10% of posts have a 500 KB image → 10M × 500 KB = 5 TB/day → object storage + CDN
```

What this tells us: one database can't take 100k reads/sec → we need caching and
read replicas or sharding; media must go to object storage, not the database.

### Worked example: bandwidth

```
Feed request returns 20 posts × 1 KB = 20 KB
40,000 reads/sec × 20 KB = 800 MB/s ≈ 6.4 Gbps out (text only — images come from the CDN)
```

## Estimation tips for the interview

- **Say your assumptions out loud** and round: "let's call it 10⁵ seconds a day".
- **Don't over-do it.** 2–3 minutes. Only compute numbers that affect the design.
- **Conclude something**: "...so a single database won't be enough for reads."
- Keep a few anchors: 1M/day ≈ 12/s; 1 KB × 1B = 1 TB; 1 day ≈ 10⁵ s.
